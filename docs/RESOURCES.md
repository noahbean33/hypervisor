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

- [Hypervisor From Scratch (Parts 1–8)](https://rayanfam.com/topics/hypervisor-from-scratch-basic-concepts/)
- [Awesome Virtualization](https://github.com/Wenzel/awesome-virtualization) - Introducing books, papers, projects, courses, CVEs, and other hypervisor hypervisor-related works
- [7 Days to Virtualization: A Series on Hypervisor Development](https://revers.engineering/7-days-to-virtualization-a-series-on-hypervisor-development/)
- [HyperDbg Debugger](https://hyperdbg.org) - Hypervisor-based debugger for analyzing, fuzzing, and reversing
- [HyperDbg Documentation](https://docs.hyperdbg.org)
- [OpenSecurityTraining2 - Hypervisor Course](https://ost2.fyi/Hwdbg1) - Free comprehensive tutorial on hypervisor-based reverse engineering

## Academic & Technical References

- [Intel® 64 and IA-32 Architectures Software Developer's Manual](https://software.intel.com/en-us/articles/intel-sdm) - Combined Volumes 1-3
- [Hardware-assisted Virtualization](http://www.cs.cmu.edu/~412/lectures/L04_VTx.pdf) - CMU Lecture Materials
- [Instruction Set Mapping » VMX Instructions](https://docs.oracle.com/cd/E36784_01/html/E36859/gntbx.html) - Oracle Documentation
- [Intel / AMD CPU Internals](https://github.com/LordNoteworthy/cpu-internals) - Reference repository
- [Obtain Processor Manufacturer using CPUID](https://www.daniweb.com/programming/software-development/threads/112968/obtain-processor-manufacturer-using-cpuid) - CPUID usage example

## Windows Driver Development

- [Writing Windows Kernel Driver](https://resources.infosecinstitute.com/writing-a-windows-kernel-driver/) - InfoSec Institute Tutorial
- [Download WDK](https://docs.microsoft.com/en-us/windows-hardware/drivers/download-the-wdk) - Windows Driver Kit
- [Windows SDK Downloads](https://developer.microsoft.com/en-us/windows/downloads/windows-sdk/)
- [OSR Driver Loader](https://www.osronline.com/article.cfm?article=157) - Tool for loading test drivers
- [Windows 10: Disable Signed Driver Enforcement](https://ph.answers.acer.com/app/answers/detail/a_id/38288/~/windows-10%3A-disable-signed-driver-enforcement)
- [Windows Driver Kit Samples](https://github.com/Microsoft/Windows-driver-samples/blob/master/general/ioctl/wdm/sys/sioctl.c) - Microsoft's official WDK samples
- [IRP_MJ_DEVICE_CONTROL](https://docs.microsoft.com/en-us/windows-hardware/drivers/kernel/irp-mj-device-control) - IOCTL handling in drivers
- [Plug and Play Minor IRPs](https://docs.microsoft.com/en-us/windows-hardware/drivers/kernel/plug-and-play-minor-irps) - PnP IRP handling
- [_FAST_IO_DISPATCH structure](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/content/wdm/ns-wdm-_fast_io_dispatch) - Fast I/O operations
- [Filtering IRPs and Fast I/O](https://docs.microsoft.com/en-us/windows-hardware/drivers/ifs/filtering-irps-and-fast-i-o) - IRP filtering techniques
- [Windows File System Filter Driver Development](https://www.apriorit.com/dev-blog/167-file-system-filter-driver) - Filter driver guide
- [Setting Up Local Kernel Debugging](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/setting-up-local-kernel-debugging-of-a-single-computer-manually) - Local debugging setup

## Hypervisor Resources

- [What Is a Type 1 Hypervisor?](http://www.virtualizationsoftware.com/type-1-hypervisors/)
- Setting Up KDNET Network Kernel Debugging (Manually and Automatically)