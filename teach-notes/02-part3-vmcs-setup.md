# HW1 part 3: building the VMCS (`adjustctl` and `vmcsinit`)

Source: Intel SDM Vol. 3, VMCS chapter (25 in the `books/` edition; the code
comments cite it as 27), VM-entry checks, and Appendix A (capability MSRs).
This note explains what part 3 does in words, and the VMCS background you need
to read KVM (`arch/x86/kvm/vmx/vmx.c`) with the same eyes.

---

## 1. What a VMCS is

The VMCS (virtual-machine control structure) is a 4 KB region of memory that
describes **one virtual CPU**. It tells the processor three things:

1. **What the guest looks like** when it starts (registers, segments, tables).
2. **What the host looks like** so the processor can come back after the guest
   stops.
3. **How the guest is allowed to run**: which of its actions are trapped.

Properties that make it different from a normal struct:

- Its layout is **implementation specific**. Software must never read or write
  the page directly. Only `VMREAD` and `VMWRITE` touch it, using a numeric
  **field encoding** (the constants in `vmx.h`). The first 4 bytes of the page
  are the revision ID, set once. Everything else belongs to the CPU.
- It must be made **current** on a logical processor (`VMPTRLD`) before
  `VMREAD`, `VMWRITE` or `VMLAUNCH` work. A processor has at most one current
  VMCS.
- It has a **launch state**: *clear* after `VMCLEAR`, *launched* after a
  successful `VMLAUNCH`. A clear VMCS must be entered with `VMLAUNCH`, a
  launched one with `VMRESUME`.
- The CPU may cache VMCS contents internally. `VMCLEAR` writes the cache back to
  memory, which is why this homework does `VMCLEAR` after every exit: the VMCS
  can then be loaded on any other CPU.

KVM equivalents: one `struct vmcs` per vCPU (`loaded_vmcs`), `vmcs_load()` is
`VMPTRLD`, `vmcs_clear()` is `VMCLEAR`, `vmcs_write32/64()` and
`vmcs_readl()` are `VMWRITE` / `VMREAD`, `init_vmcs()` is the big "set up
everything constant" function that corresponds to your `vmcsinit()`.

## 2. The six areas of a VMCS

| Area | Written by | Purpose |
|---|---|---|
| **Guest-state** | VMM before entry; CPU on exit | Registers the guest runs with: CR0/CR3/CR4, RIP/RSP/RFLAGS, segment registers, GDTR/IDTR, DR7, some MSRs, interruptibility. |
| **Host-state** | VMM | Registers the CPU loads on every VM exit: CR0/CR3/CR4, RSP/RIP, segment selectors, FS/GS/TR/GDTR/IDTR bases, some MSRs. |
| **VM-execution controls** | VMM | What the guest may do without exiting. Pin-based, primary processor-based, secondary processor-based, exception bitmap, CR masks, EPT pointer, I/O and MSR bitmaps. |
| **VM-exit controls** | VMM | What the CPU does on exit: save/load EFER, host in 64-bit mode, acknowledge interrupt. |
| **VM-entry controls** | VMM | What the CPU does on entry: load EFER, guest in IA-32e mode, event injection. |
| **VM-exit information** (read-only) | CPU | Why the exit happened: exit reason, qualification, instruction length, guest-physical address, IDT-vectoring info. |

The shape to remember: **entry loads guest state, exit saves guest state and
loads host state**. Everything else is policy. Part 3 fills in the three control
areas (except two fields done at run time) and the constant parts of guest
state. Part 4 fills in the host state.

## 3. `adjustctl()`: why controls need negotiation

### The problem

Each control word (pin-based, primary, secondary, exit, entry) is a 32-bit
bitmask. Not every CPU implements every bit, and some bits **must** be 1 on some
CPUs for historical reasons ("default-1" bits, reserved but required). If you
write a control value the CPU doesn't accept, `VMLAUNCH` fails with
"invalid control field". So you cannot hard-code control words.

### The capability MSR

For each control word there is a read-only 64-bit MSR (Appendix A.3–A.5, and
A.6 for secondary):

- **Low 32 bits: "allowed-0 settings".** A bit that is **1** here means the
  control **must be 1**. The CPU requires it.
- **High 32 bits: "allowed-1 settings".** A bit that is **0** here means the
  control **must be 0**. The CPU cannot do it.
- A bit that is 0 in the low half and 1 in the high half is free: you choose.

### What the function does, in words

1. Read the capability MSR.
2. Take the bits you want and add the bits the CPU requires (OR with the low
   half).
3. Remove everything the CPU doesn't support (AND with the high half).
4. Check that every bit you *wanted* survived step 3. If one disappeared, the
   CPU doesn't support a control you need. Report which one and fail. A
   hypervisor that silently runs without a needed control behaves wrongly in
   ways that are hard to debug.
5. Otherwise hand back the final value.

This is exactly what KVM's `adjust_vmx_controls()` / `setup_vmcs_config()`
does, except that KVM has "required" and "optional" bit lists and masks the
optional ones silently.

### The TRUE MSRs

`IA32_VMX_BASIC` bit 55 says whether four extra "TRUE" MSRs exist (TRUE pin,
TRUE proc, TRUE exit, TRUE entry). Some bits are reserved-1 in the original
MSRs for compatibility with the first VMX CPUs even though newer CPUs can really
clear them (the classic example is CR3-load and CR3-store exiting). The TRUE
MSRs report what the hardware truly allows. So `vmcsinit` uses the TRUE variant
when available. There is no TRUE variant for the secondary controls (they have
no legacy bits).

## 4. `vmcsinit()`: what each part does and why

### 4.1 Pin-based controls

Pin-based controls govern asynchronous events from outside the CPU.

- **External-interrupt exiting**: a host device interrupt arriving while the
  guest runs causes a VM exit. Without it the interrupt would be delivered
  through the *guest's* IDT, and the host would never see its timer tick or disk
  interrupt. With it, the exit hands control to the host, which then gets the
  interrupt through its own IDT as soon as it re-enables interrupts.
- **NMI exiting**: the same for non-maskable interrupts.

(Not set: "acknowledge interrupt on exit", so the interrupt stays pending in the
local APIC and is taken normally after the exit. "Virtual NMIs" and
"preemption timer" are not used.)

KVM: `PIN_BASED_EXT_INTR_MASK | PIN_BASED_NMI_EXITING | PIN_BASED_VIRTUAL_NMIS`
(plus posted interrupts when used).

### 4.2 Primary processor-based controls

These decide which instructions and events trap.

- **HLT exiting**: the guest executes `HLT` to say "I'm done". Trapping it
  returns control to the monitor, which reports it to user space. Without it the
  physical CPU would really halt.
- **Unconditional I/O exiting**: every `IN` and `OUT` traps. This is the simple
  alternative to the **I/O bitmaps** (two 4 KB pages, one bit per port). The
  "use I/O bitmaps" control is left off. If a CPU's capability MSR forced it on,
  the VMCS would need bitmap pages the code doesn't allocate, so the code
  refuses such a CPU.
- **Activate secondary controls**: without this bit, the secondary word is
  ignored and EPT and unrestricted guest cannot be turned on.

The code also **refuses CPUs that force on** unwanted exits: CR3-load exiting,
CR3-store exiting, use of I/O bitmaps, use of MSR bitmaps. With them forced on,
the guest's writes to CR3 would trap (and EPT guests don't need that), or the
VMCS would require a bitmap page the code doesn't allocate. The TRUE MSRs help
avoid false alarms here.

An **MSR bitmap** is left off, which means *every* `RDMSR` / `WRMSR` in the guest
exits. That is simple and safe: the guest touches an MSR only when it sets up
64-bit mode (EFER), and `vmxmsr()` emulates that exit.

KVM: much longer, tuning for performance (MSR bitmaps, TPR shadow, APIC
virtualization, ...), but the same field.

### 4.3 Secondary processor-based controls

- **Enable EPT**: nested paging. The CPU translates guest-physical addresses to
  host-physical addresses through the EPT tree built in part 2. Without EPT, the
  VMM would have to shadow the guest's page tables in software.
- **Unrestricted guest**: lets the guest run with CR0.PE and CR0.PG clear, i.e.
  in real mode or unpaged protected mode. Without it, VMX requires the guest to
  be in paged protected mode (see the fixed-bit rules from the VMX-enabling
  note). This guest begins in 32-bit protected mode with no paging and switches
  itself to 64-bit mode. Unrestricted guest **requires EPT**.

KVM: `SECONDARY_EXEC_ENABLE_EPT | SECONDARY_EXEC_UNRESTRICTED_GUEST` (plus VPID,
APIC virtualization, XSAVES, ...).

### 4.4 VM-exit controls

- **Host address-space size**: on exit the CPU returns to **64-bit mode**. The
  host is a 64-bit kernel; the processor must be told, or it would drop to
  32-bit mode and crash.
- **Save IA32_EFER** and **load IA32_EFER**: the guest changes EFER (to enter
  long mode) and the host needs its own value back. These controls let the CPU
  save the guest's and load the host's automatically, with no `RDMSR` /
  `WRMSR` round trip.

Not set: "acknowledge interrupt on exit", explained above.

### 4.5 VM-entry controls

- **Load IA32_EFER**: on entry, load the guest's EFER from the VMCS.
- The "IA-32e mode guest" bit is left to `vmrun()`, because it depends on the
  guest's current EFER.LMA, which changes while the guest runs.

### 4.6 Exceptions and page faults

- **Exception bitmap = all ones**: every guest exception (vector 0–31) exits.
  The guest has no IDT yet, and a fault with no handler would escalate to a
  triple fault and reset the vCPU. Exiting lets the monitor see it. (KVM sets
  only the bits it needs, such as #PF for shadow paging and #DB/#BP for
  debugging.)
- **Page-fault error-code mask and match**: when the bitmap bit for #PF is set,
  these two fields choose which error codes exit. Mask = match = 0 means "all
  of them".

### 4.7 Switching off optional features

All set to zero or off so the processor does nothing extra:

- **CR3-target count = 0**: no CR3 values that would let a `MOV CR3` skip the exit.
- **MSR store/load counts = 0** (exit-store, exit-load, entry-load): no
  automatic MSR lists.
- **Entry interruption-information = 0**: don't inject any interrupt or
  exception on this entry. (Bit 31, "valid", is 0.)
- **VMCS link pointer = all ones**: "no shadow VMCS". The SDM says to set the
  field to all ones when it is not used, "to avoid VM-entry failures".
  (KVM nested virtualization is the big user of VMCS shadowing.)

### 4.8 EPT pointer

The 64-bit EPTP combines the physical address of the PML4, the walk length
(4 levels), and the memory type (write-back) of the EPT paging structures
themselves. It was built in `vmcreate()` and is stored in the VMCS here. On
every guest memory access the CPU starts from this pointer.

### 4.9 Control-register ownership: guest/host masks and read shadows

For CR0 and CR4 the VMCS holds a **guest/host mask** and a **read shadow**:

- A bit set in the **mask** is **owned by the host**. The guest may not change
  it: a write that would change it to a value different from the read shadow
  causes an exit. Reads of that bit return the shadow's value instead of the
  real register's.
- A bit clear in the mask is **owned by the guest**. It reads and writes the real
  register without exits.

Here:

- **CR0 mask = 0**: the guest owns all of CR0. Turning on paging, protection
  and so on is free.
- **CR4 mask = VMXE only**: the real CR4 must keep VMXE = 1 (it is a fixed-1 bit
  in VMX operation), but the guest should never be told VMX is on. The read
  shadow is 0 for that bit, so the guest reads VMXE = 0, and if it tries to set
  it, the write exits.

KVM uses the same machinery (`cr0_guest_owned_bits`, `cr4_guest_owned_bits`,
`vmx_set_cr0`, `vmx_set_cr4`).

### 4.10 Host return address

`HOST_RIP` is where the processor jumps after every VM exit. Here it is the
assembly label `vmx_exit`. Its stack pointer (`HOST_RSP`) is not written here:
it depends on the current kernel stack and is written before every entry in
`vmrun()` / part 4, along with the other host state.

### 4.11 Guest control registers

- **CR0** starts with protected mode on (PE), the always-1 bit ET, and NE
  (required: numeric-error). The fixed-0 bits from the MSR are added, **except**
  PE and PG, which unrestricted guest relaxes. Then the fixed-1 bits are masked
  in as usual. PG stays clear: no paging.
- **CR4**: VMXE plus whatever the CPU requires.
- **CR3**: 0, irrelevant until the guest enables paging.
- **DR7**: reset value `0x400` (bit 10 is reserved-1): no hardware breakpoints.

This is how a hypervisor chooses the guest's initial mode. KVM's
`vmx_vcpu_reset()` / `vmx_set_cr0()` do the equivalent but start the vCPU in real
mode at the reset vector.

### 4.12 Guest segment registers

In real hardware a segment register has a visible selector and a hidden
"descriptor cache" (base, limit, access rights) loaded from the GDT. The VMCS
stores **all four pieces explicitly** for each segment, because on entry the CPU
loads the hidden part from the VMCS without consulting any GDT. That is why a
guest with no GDT yet still needs full descriptor values.

- **CS**: selector 8, base 0, limit 4 GB, access rights = 32-bit code segment.
- **DS, ES, FS, GS, SS**: selector 0x10, base 0, limit 4 GB, 32-bit data
  segment. This is a flat 32-bit protected-mode layout.
- **LDTR**: marked **unusable**. As after reset, there is no LDT.
- **TR**: must be valid for entry. A busy 32-bit TSS with limit 0xFFFF is the
  simplest value that passes the entry checks.

The **access-rights** format is its own thing: type, S (descriptor type), DPL,
present, AVL, L (64-bit), D/B, granularity, and an "unusable" bit. It is not
the same layout as the GDT descriptor bytes.

### 4.13 Descriptor tables and the remaining guest state

- **GDTR, IDTR**: base 0, limit 0 (none yet).
- **Interruptibility state = 0**: not blocked by `STI`, `MOV SS` or NMI.
- **Activity state = 0**: active (not halted, not in shutdown).
- **Pending debug exceptions = 0**.
- **SYSENTER CS/ESP/EIP, DEBUGCTL = 0**: guest MSR state in the VMCS, zero.

### 4.14 The entry checks, and why they matter

At `VMLAUNCH` the CPU validates the **controls**, the **host state** and the
**guest state** (SDM "VM entries" chapter). If the controls or host state are
bad, `VMLAUNCH` itself fails with an error number (read it with `VMREAD` of the
VM-instruction error field). If the *guest* state is bad, the entry *starts*
but immediately takes a VM exit with the "VM-entry failure" bit set in the exit
reason. Typical culprits:

- CR0/CR4 violating the fixed bits (and PE/PG without unrestricted guest);
- a segment whose access rights contradict its selector or limit
  (for example granularity bit versus limit bits);
- TR not usable, or a TSS not busy;
- a non-canonical base in the descriptor tables;
- RFLAGS bit 1 not set; and
- the VMCS link pointer not all ones.

This explains why `vmcsinit()` writes values that look pedantic.

## 5. How this maps to KVM

| This homework | KVM (`arch/x86/kvm/vmx/vmx.c`) |
|---|---|
| `adjustctl` | `adjust_vmx_controls`, `setup_vmcs_config` |
| `vmcsinit` | `init_vmcs`, `vmx_vcpu_reset` |
| TRUE MSR selection | `IA32_VMX_TRUE_*` handling in `setup_vmcs_config` |
| Exception bitmap | `vmx_update_exception_bitmap` |
| CR guest/host masks | `cr0_guest_owned_bits`, `cr4_guest_owned_bits` |
| `vmxhoststate` | `vmx_vcpu_load_vmcs`, `vmx_prepare_switch_to_guest` |
| `vmx_enter` / `vmx_exit` | `__vmx_vcpu_run` in `vmenter.S` |
| exit handler | `vmx_handle_exit`, table `kvm_vmx_exit_handlers` |
| EPT walk | `kvm_mmu` / `tdp_mmu`, `vmx_load_mmu_pgd` writing `EPT_POINTER` |

When you read KVM, the same questions apply: which controls does it require,
which are optional, who owns each control register bit, and what does each exit
reason mean for the monitor.

## 6. Pitfalls to remember

- Every `vmwrite` goes to the **current** VMCS. If `VMPTRLD` didn't happen,
  `vmwrite` fails (the kernel panics in this homework).
- A control that is accepted by `adjustctl` can still conflict with another
  (for example unrestricted guest without EPT). Check combinations.
- The guest-state checks are the usual cause of "VM entry failure" exits. Read
  the exit reason and the qualification first.
- A VMCS must be **clear** before moving it between CPUs.
- Don't leave debug `cprintf` calls in `vmcsinit`; it runs with interrupts off
  and prints on every `vmcreate`.

## Self-check

1. In a capability MSR, which half holds the bits that must be 1, and which half
   the bits that must be 0?
2. Why can't you hard-code the control words?
3. What would break if external-interrupt exiting were off?
4. What does a CR4 read shadow of 0 combined with a mask of VMXE do?
5. Why does the guest need a full set of segment descriptor values when it has
   no GDT?
6. When does a bad guest-state field fail: at `VMLAUNCH`, or later?

Answers: (1) low = must-be-1, high = may-be-1. (2) CPUs differ and some bits are
reserved-1. (3) The host would never see device interrupts while a guest runs.
(4) The guest reads VMXE as 0 and a write to it exits, while the real bit stays
1. (5) Entry loads the hidden descriptor cache from the VMCS, not from a GDT.
(6) The entry starts and immediately exits with the entry-failure bit set; bad
controls or host state fail `VMLAUNCH` itself.
