# Enabling VMX

Source: Intel SDM Vol. 3 (`books/325384-sdm-vol-3abcd.pdf`): §24.6–24.8, §25.11.5,
Appendix A.1 / A.7 / A.8, and the VMXON entry in Chapter 31.
The comments in `vmx.c` / `vmx.h` cite "SDM 26.7"; in the copy of the SDM in
`books/` the same section is **24.7**. Section numbers shift between editions, so
search by title ("Enabling and Entering VMX Operation").

Goal of this note: understand what must be true before `VMXON` succeeds, so that
writing `vmxinit()` (HW1 part 1) is mechanical.

---

## 0. The big picture

```
 power-on / reset
      │  IA32_FEATURE_CONTROL = 0 (unlocked, VMX not allowed)
      ▼
 firmware (BIOS/UEFI)  ──── usually sets lock + bit 2 ────►  MSR 0x3A locked
      ▼
 xv6 boots, each CPU runs mpmain() → vmxinit()
      │
      │ 1. CPUID.1:ECX[5]?             ── no  → give up quietly (vmx_ok = 0)
      │ 2. FEATURE_CONTROL ok/locked?  ── no  → try to enable, else give up
      │ 3. CR0/CR4 meet fixed bits + CR4.VMXE = 1
      │ 4. VMXON region: page + revision ID
      │ 5. VMXON                       ── fail → report, vmx_ok = 0
      ▼
 VMX root operation, no current VMCS         (vmx_ok = 1)
      │
      │  later: VMCLEAR / VMPTRLD / VMWRITE ... / VMLAUNCH
      ▼
 VMX non-root operation (the guest runs)
```

Key vocabulary:

| Term | Meaning |
|---|---|
| VMX operation | The CPU mode entered by `VMXON`, left by `VMXOFF`. |
| VMX root operation | Where the hypervisor (here: xv6 kernel) runs inside VMX operation. |
| VMX non-root operation | Where the guest runs. Not reachable until a VMCS is set up. |
| VMXON region | Per-CPU scratch page the CPU uses to support VMX operation. |
| VMCS | Per-virtual-CPU control structure. Not needed yet for enabling VMX. |
| SMX | Safer Mode Extensions (Intel TXT, `GETSEC`). Irrelevant for xv6 beyond one MSR bit. |

---

## Step 1. Is VMX there? (§24.6)

`CPUID` with EAX=1, test **ECX bit 5**. If set, VMX is supported.

- Only the *presence* of VMX is reported here. Everything optional (EPT,
  unrestricted guest, VPID, ...) is reported through capability MSRs in Appendix A,
  which are used later when building the VMCS.
- This check must come **before** any VMX MSR read. The VMX capability MSRs
  (0x480 and up) exist only on VMX CPUs. `rdmsr` of a non-existent MSR raises #GP,
  and in the kernel that panics. `vmx.h` says exactly this in its MSR comment.

## Step 2. Does firmware allow it? `IA32_FEATURE_CONTROL` (MSR 0x3A) (§24.7)

Cleared to zero on every logical processor reset.

```
 bit   2          1            0
     ┌────────┬────────────┬────────┐
     │ VMXON  │ VMXON in   │  LOCK  │
     │ outside│ SMX (TXT)  │        │
     │ SMX    │            │        │
     └────────┴────────────┴────────┘
```

| Bit | Meaning |
|---|---|
| 0 | **Lock.** Clear → `VMXON` raises #GP. Set → `WRMSR` to this MSR raises #GP, frozen until power-up reset. |
| 1 | Enable `VMXON` inside SMX operation. Setting it on a CPU without both VMX and SMX raises #GP. |
| 2 | Enable `VMXON` outside SMX operation. **The one a normal OS needs.** |

The manual's rule for firmware: to enable VMX, set bit 1, bit 2 or both, **and** the
lock bit. Setting the lock bit alone, with both enable bits clear, is how the BIOS
setup option "Intel Virtualization Technology: Disabled" is implemented.

Decision table for the OS:

| Lock | Bit 2 | What it means | What to do |
|---|---|---|---|
| 1 | 1 | Firmware enabled VMX (the usual case on a lab machine). | Continue. |
| 1 | 0 | Firmware disabled VMX, permanently until reset. | Can't change it. Report and give up. |
| 0 | any | Nobody configured it (some VMs/hypervisors, minimal firmware). | Set bit 2 and bit 0 with `WRMSR`, then continue. |

Notes:
- The first CPU that writes the MSR locks it. Other CPUs have their own copy of the
  MSR (it is per logical processor), so each CPU must do this step itself. In
  practice firmware has usually already done it on all CPUs.
- The SMX bit (bit 1) is the TXT case, which this homework does not use. `vmx.h`
  only defines the lock bit and bit 2.

## Step 3. Make CR0 and CR4 acceptable, and set `CR4.VMXE` (§24.7, §24.8, A.7, A.8)

Two separate requirements.

### 3a. `CR4.VMXE` (bit 13, `0x2000`)

- `VMXON` with `VMXE = 0` raises **#UD**.
- Once in VMX operation `VMXE` can't be cleared. To clear it, `VMXOFF` first.
- Later, `vmcsinit()` uses the CR4 guest/host mask (`CR4_GUEST_HOST_MASK`,
  `CR4_READ_SHADOW`) so that the guest can't see or clear it. This is the reason the
  hypervisor "owns" `VMXE`.

### 3b. Fixed bits

In VMX operation, the CPU forces some bits of CR0 and CR4 to constant values.
Four read-only MSRs describe this:

| MSR | Address | Meaning |
|---|---|---|
| `IA32_VMX_CR0_FIXED0` | 0x486 | bit = 1 → CR0 bit **must be 1** |
| `IA32_VMX_CR0_FIXED1` | 0x487 | bit = 0 → CR0 bit **must be 0** |
| `IA32_VMX_CR4_FIXED0` | 0x488 | bit = 1 → CR4 bit **must be 1** |
| `IA32_VMX_CR4_FIXED1` | 0x489 | bit = 0 → CR4 bit **must be 0** |

Per bit, the pair (FIXED0, FIXED1) means:

| FIXED0 | FIXED1 | Bit is |
|---|---|---|
| 0 | 0 | fixed to 0 |
| 1 | 1 | fixed to 1 |
| 0 | 1 | flexible |
| 1 | 0 | never happens (the manual guarantees FIXED0 ⊆ FIXED1) |

To make a register legal: **force the must-be-1 bits on, force the must-be-0 bits
off**:

```
legal = (cr | FIXED0) & FIXED1
```

Illustration with made-up but typical numbers (read the real ones from the MSRs;
they differ between CPUs and between real hardware and an emulator):

```
CR0:  FIXED0 = 0x80000021   (PG, NE, PE must be 1)
      FIXED1 = 0xFFFFFFFF   (nothing forced to 0)
CR4:  FIXED0 = 0x00002000   (VMXE must be 1)  [newer CPUs may add more]
      FIXED1 = 0x00xxxxxx   (some high bits forced to 0 = features CPU lacks)
```

This is why **VMXE is itself a "fixed-1" bit in CR4**: the manual's note says the
first VMX processors required CR0.PE, CR0.NE, CR0.PG and CR4.VMXE to be 1 in VMX
operation.

What goes wrong if you skip 3b:
- `VMXON` raises #GP(0) ("values of CR0 and CR4 are not supported in VMX operation").
- Even after a successful `VMXON`, any later `MOV CR0/CR4`, `CLTS` or `LMSW` that
  produces an illegal value raises #GP. VM entry and VM exit also can't load illegal
  values. So the host values saved in the VMCS later must also be legal. This is
  why they will be derived from the CPU's current (already legal) CR0/CR4.

CR0.PE and CR0.PG being fixed to 1 means VMX works only in paged protected mode.
The "unrestricted guest" control (a secondary control, used later in this
homework) is what lets the *guest* run with PE/PG = 0. It doesn't change the
rules for the host.

## Step 4. Build the VMXON region (§25.11.5, A.1)

A block of memory the CPU may use internally while in VMX operation.

```
VMXON region (one 4 KB page per logical processor)

  byte 0                       3 4                              4095
  ┌───────────────────────────┬──────────────────────────────────┐
  │ 0 │ VMCS revision ID (31b)│  anything (CPU owns it; software │
  └───────────────────────────┴──  must not touch it after VMXON)┘
   ↑ bit 31 must be 0
```

Requirements:

- **4 KB aligned**, and the address passed to `VMXON` is **physical**, with no bits
  set above the physical address width. (`kalloc()` returns page-aligned kernel
  virtual memory; convert with `V2P`.)
- **Revision ID.** Read `IA32_VMX_BASIC` (MSR 0x480). Bits 30:0 are the VMCS
  revision identifier (bit 31 is always 0). Write that value in bits 30:0 of the first
  4 bytes of the region. Without it, `VMXON` fails with VMfailInvalid.
- **Size.** `IA32_VMX_BASIC` bits 44:32 say how many bytes are needed, a value
  between 1 and 4096. One page always suffices.
- **Zero the rest** is not required by the manual ("need not initialize ... in any
  other way"), but `kalloc()` in xv6 already fills pages with junk, and clearing is
  harmless and makes debugging easier.
- **One region per logical processor.** The CPU uses it for its own state, and the
  manual says not to access or modify it between `VMXON` and `VMXOFF`.

Other fields of `IA32_VMX_BASIC` you will meet later (not needed now):

| Bits | Field |
|---|---|
| 30:0 | VMCS revision ID |
| 44:32 | Region size (VMXON and every VMCS) |
| 48 | 1 = addresses limited to 32 bits (always 0 on Intel 64) |
| 53:50 | Memory type for VMCS and related structures (6 = write-back) |
| 55 | "TRUE" control MSRs supported (`VMX_BASIC_TRUE_CTLS` in `vmx.h`) |

The same revision ID is used for the VMCS region later, which is why `vmx.c` keeps
it in a global (`vmcs_revision`, with the comment "same on every CPU").

## Step 5. Execute `VMXON` (Chapter 31)

The instruction takes **a memory operand that contains the 64-bit physical address**
of the region, not the address in a register. Hence the `"m" (pa)` constraint in the
`vmxon()` wrapper in `vmx.c`: the compiler puts `pa` in memory so `VMXON` can read it.

The manual's pseudocode, in checking order:

```
if  register operand, CR0.PE=0, CR4.VMXE=0, RFLAGS.VM=1, or (LMA=1 and CS.L=0):
        #UD
elif not already in VMX operation:
    if  CPL>0, A20M mode, CR0/CR4 fixed bits violated,
        FEATURE_CONTROL.lock=0, or the SMX-appropriate enable bit=0:
            #GP(0)
    elif addr not 4 KB aligned, or addr has bits above physical width:
            VMfailInvalid           (CF = 1)
    elif region revision ID != CPU's revision ID, or bit 31 set:
            VMfailInvalid           (CF = 1)
    else:
            current-VMCS pointer := 0xFFFFFFFF_FFFFFFFF
            enter VMX operation; block INIT; disable A20M;
            clear MONITOR address-range state;
            (maybe) IA32_RTIT_CTL.TraceEn := 0
            VMsucceed               (CF = 0, ZF = 0)
elif in VMX non-root operation:   VM exit
elif CPL>0:                       #GP(0)
else:                             VMfail("VMXON executed in VMX root operation")
```

Observations:

- **Exceptions versus failures.** #UD and #GP are exceptions: the kernel traps
  (panic). VMfail variants are reported in flags and can be handled gracefully. A
  correct sequence never relies on either, which is why steps 1–4 precede it.
- **CF reports failure.** The wrapper in `vmx.c` uses `setna` (set if CF=1 or ZF=1)
  to turn flags into a return value.
- **Double `VMXON`.** Executing `VMXON` while already in VMX root operation gives
  VMfail, not success. Be careful if `vmxinit()` can run twice on a CPU.
- **After success there is no current VMCS.** The next instructions in the lifecycle
  (`VMCLEAR`, `VMPTRLD`) are what make one current.

Side effects of VMX operation to be aware of:

| Effect | Consequence |
|---|---|
| INIT signals blocked in root operation | Not a problem for xv6; matters for AP startup in other systems. |
| A20M disabled | Irrelevant on x86-64. |
| CR0/CR4 fixed-bit rules apply | See 3b. |
| Intel PT may be turned off | Irrelevant here. |

---

## Failure catalogue

| Symptom | Likely cause |
|---|---|
| Kernel panics with #GP in `rdmsr` before anything else | Read a VMX MSR on a CPU without VMX (Step 1 skipped). |
| #UD on the `vmxon` instruction | `CR4.VMXE` not set (Step 3a), e.g. forgot `lcr4`. |
| #GP(0) on `vmxon` | Lock bit clear, enable bit clear, or CR0/CR4 fixed bits wrong (Steps 2, 3b). |
| `vmxon` returns failure, no exception | Region not aligned, wrong/missing revision ID, bad address (virtual instead of physical), or already in VMX operation (Step 4). |
| Works on CPU 0, fails on others | Per-CPU state shared (one region for all CPUs), or the CR4/fixed-bit setup done only on the boot CPU. |
| `wrmsr` to FEATURE_CONTROL faults | It is already locked with the wrong value. You can't change it; check the lock bit first. |
| Works on real hardware, fails in QEMU or Bochs | Nested VMX not enabled on the host, or firmware leaves the MSR unlocked (exercises the "unlocked" branch). |

## Where each piece lives in this repo (read-only map)

| Need | In the repo |
|---|---|
| CPUID wrapper | `cpuid_insn()` in `vmx.c` |
| CPUID bit mask | `CPUID1_ECX_VMX` in `vmx.h` |
| `rdmsr` / `wrmsr` | `x86.h` |
| `rcr0` / `lcr0` / `rcr4` / `lcr4` | `x86.h` |
| MSR numbers | `MSR_IA32_FEATURE_CONTROL`, `MSR_IA32_VMX_BASIC`, `MSR_IA32_VMX_CR0_FIXED0/1`, `MSR_IA32_VMX_CR4_FIXED0/1` in `vmx.h` |
| Bit masks | `FEATURE_CONTROL_LOCKED`, `FEATURE_CONTROL_VMXON_OUTSIDE_SMX`, `CR4_VMXE` in `vmx.h` |
| The `VMXON` instruction | `vmxon(uint64 pa)` in `vmx.c` (returns 0 / -1) |
| Revision ID global | `vmcs_revision` in `vmx.c` |
| Per-CPU result flag | `vmx_ok` in `struct cpu` (`proc.h`) |
| Physical address of a kernel page | `V2P()` in `memlayout.h` |
| Page allocator | `kalloc()` / `kfree()` |
| Caller | `vmxinit()` is called from `mpmain()` in `main.c` on every CPU |
| Consumers of the flag | `vmcreate()` / `vmrun()` check `cpus[c].vmx_ok` |

The stub to fill in is `vmxinit()` in `vmx.c`; it takes no arguments and returns
nothing, so it reports success only through `mycpu()->vmx_ok`.

---

## HW1 part 1: why the questions ask what they ask

- **1.1 `return` instead of `panic()` when VMX is missing.** A CPU without VMX is a
  legitimate machine, not a kernel bug. xv6 should still boot and work as a plain OS.
  `vmx_ok` stays clear and `vmcreate()` / `vmrun()` refuse to work. `panic()` would
  make the whole OS unusable for a missing optional feature.
- **1.2 Handling the unlocked case.** The MSR resets to zero, so unlocked is the
  initial state. The manual only says what firmware *can* do. Some firmware, and
  some virtual machines, never set it. Handling it makes the hypervisor work in
  more environments. Locked-and-disabled is the only case that must fail.
- **1.3 Skipping `lcr4(cr4)` before `VMXON`.** `CR4.VMXE` stays 0, so `VMXON` raises
  #UD (invalid opcode) in the kernel and the machine panics at boot. (The bit set
  in a local variable doesn't reach the hardware register until it is loaded.)
- **1.4 One VMXON region per CPU.** The region is the CPU's private working memory
  for VMX operation, and software must not touch it while the CPU is in VMX
  operation. Two CPUs sharing one page would corrupt each other's state, with
  "unpredictable behavior" as the manual puts it.

## Quick self-check

1. Which MSR must you read to find the revision ID, and which bits?
2. Why is it safe to read `IA32_VMX_CR0_FIXED0` only after step 1?
3. After `VMXON` succeeds, is there a current VMCS? What is the pointer's value?
4. Suppose CR4 `FIXED0 = 0x2000` and `FIXED1 = 0x0FFFFF`. The current CR4 is
   `0x6A0`. What value must you load before `VMXON`?
   (Answer: `(0x6A0 | 0x2000) & 0x0FFFFF = 0x26A0`.)
5. If the lock bit is set and bit 2 is clear, can software fix it? What should
   `vmxinit()` do?

## Reference pointers in the SDM

- §24.6 Discovering support for VMX
- §24.7 Enabling and entering VMX operation
- §24.8 Restrictions on VMX operation
- §25.11.5 VMXON region
- Chapter 31 → "VMXON — Enter VMX Operation" (the pseudocode quoted above)
- Appendix A.1 Basic VMX information; A.7 / A.8 VMX-fixed bits in CR0 / CR4
