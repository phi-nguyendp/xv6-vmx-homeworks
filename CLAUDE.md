# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

xv6-64 (the x86-64 port of MIT's xv6, see `README`) extended with a small Intel VT-x hypervisor. This is homework 1 (branch `hw1-assign`): the hypervisor in `vmx.c` and `vmxasm.S` ships with **stubbed functions** that students fill in. Stubs are marked `/* HW1 part N: write this function; see HW1.md. */` (vmx.c) and `YOUR CODE, block A/B (HW1 part 4)` (vmxasm.S). Written answers go in `answers.md` (keep question numbers). `HW1.md` is referenced but not in the repo. The Intel SDM (Vol. 3) is in `books/`.

Until stubs are filled in, `vmxinit()` leaves `mycpu()->vmx_ok` clear and `vmcreate()`/`vmrun()` refuse to work, so a "working" xv6 boot does not mean the hypervisor works.

## Commands

Build and run (native x86-64 ELF toolchain; set `TOOLPREFIX=x86_64-elf-` otherwise). A Nix flake / `shell.nix` is provided.

- `make` — build `kernel`, `bootblock`, user programs, `fs.img`, `xv6.img`
- `make qemu` / `make qemu-nox` — run under QEMU (`-nographic` for the latter). `CPUS` (default 2) sets SMP. VT-x needs nested virtualization: QEMU must run with KVM and the host's `kvm_intel nested=1` (override with `make qemu QEMUKVM=`; see Makefile for the default).
- `make bochs` / `make bochs-nox` — Bochs; `-nox` is headless with the xv6 serial console on the tty via `bochs-console.py` (quit with `Ctrl-A x`). Bochs is the alternative for machines without nested KVM.
- `make qemu-gdb` / `qemu-nox-gdb` — start paused for gdb (`.gdbinit` generated from `.gdbinit.tmpl`)
- `make clean`

There is no automated test suite. Verify inside xv6: run `runvm helloguest` at the xv6 shell (guest prints via emulated COM1, then halts); `usertests` exists for the base OS.

CFLAGS deliberately disable `-Wunused-*` errors (so stubs compile) but keep `-Werror` otherwise. Kernel objects compile with `-m64 -mcmodel=large -DX86_64`.

## Architecture

Hypervisor design mirrors KVM: kernel does the privileged parts, a user-space VMM does device emulation.

- `vmxapi.h` — the ABI shared by kernel and user space (must stay free of kernel-only definitions): syscalls `vmcreate(image,len,memsz)`, `vmsetreg(vm,reg,val)`, `vmrun(vm,&vmexit)`, `VM_LOAD_ADDR` (0x100000), `VMREG_*` indices, exit reasons.
- `vmx.h` — VMCS field encodings, MSR/control bit definitions, `struct guestregs` (field order is what `VMREG_*` indexes into), `struct vmx`/VM state. Per-CPU `vmx_ok` lives in the CPU struct (`proc.h`).
- `vmx.c` — the hypervisor: `vmxinit()` (VMXON per CPU, called from `mpmain()` on every CPU), VMCS setup, EPT construction, `vmcreate`/`vmsetreg`/`vmrun`, in-kernel handling of exits (host interrupts, NMI, EFER WRMSR); other exits are returned to user space.
- `vmxasm.S` — VMLAUNCH/VMRESUME entry/exit trampoline; saves host callee-saved registers and swaps the 15 GPRs (not in the VMCS) around guest execution.
- `sysvmx.c` — thin syscall wrappers: fetch/validate args (`argint`, `argint64`, `argptr`) and call into `vmx.c`. New syscalls need entries in `syscall.h`, `syscall.c`, `usys.S`, `user.h`.
- `runvm.c` — user-space VMM: loads an image, sets RIP, loops on `vmrun()` emulating a transmit-only COM1 and stopping on HLT.
- `helloguest.c`, `helloguestasm.S`, `helloguest.ld` — the guest image, linked at `VM_LOAD_ADDR`; built by the `helloguest` Makefile target (flat binary via objcopy) and placed in `fs.img` alongside the user programs. Changing the load address requires changing both `vmxapi.h` and the linker script.

The rest is stock xv6-64 (`proc.c`, `vm.c`, `trap.c`, `fs.c`, ...). New user programs go in `UPROGS` in the Makefile.
