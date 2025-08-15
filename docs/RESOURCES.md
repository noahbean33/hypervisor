# Resources

A collection of useful websites and books for operating systems and kernel development, plus hypervisor- and Windows-driver–specific references for this project.

## Websites

- OSDev.org Wiki: A comprehensive resource for hobbyist OS developers.
- Ralf Brown's Interrupt List: An extensive reference for PC hardware interrupts.

## Books

- Arpaci-Dussau, R. H., & Arpaci-Dusseau, A. C. (2023). Operating Systems: Three Easy Pieces.
- Bryant, R. E., & O'Hallaron, D. R. (2016). Computer Systems: A Programmer's Perspective (3rd ed.).
- Hennessy, J. L., & Patterson, D. A. (2017). Computer Architecture: A Quantitative Approach (6th ed.).
- Kernighan, B. W., & Ritchie, D. M. (1988). The C Programming Language (2nd ed.).
- Kerrisk, M. (2010). The Linux Programming Interface.
- Patterson, D. A., & Hennessy, J. L. (2021). Computer Organization and Design RISC-V Edition (2nd ed.).

## Hypervisor / VT-x / VMX

- Intel 64 and IA-32 Architectures Software Developer’s Manual (Volumes 1–3)
  - Virtual Machine Extensions (VMX) architecture
  - Virtual Machine Control Structure (VMCS)
  - Extended Page Tables (EPT)
  - CR0/CR4 fixed bits and VMX-related MSRs (e.g., IA32_FEATURE_CONTROL, IA32_VMX_*)

## Windows Kernel & Driver Development

- Windows Driver development fundamentals (WDM/KMDF), IRPs, DriverEntry, device objects, and symbolic links.
- Kernel debugging with WinDbg.
- Development-time driver loading approaches and driver signing/test modes for non-production use.

## Nested Virtualization

- Guidance on enabling nested virtualization in common hypervisors (e.g., VMware Workstation, Hyper-V).
- Considerations for exposing VT-x/VMX inside a guest VM.

## Tools

- WinDbg (kernel debugging)
- OSR Driver Loader (developer-friendly driver install/start/stop)
- Sysinternals tools useful for driver development
- HyperDbg (hypervisor-based debugger) as a learning reference

## Tutorials / Articles

- Hypervisor From Scratch (Parts 1–8)
- Awesome Virtualization (community-curated resources)