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
- [Windows Driver Kit Samples](https://github.com/Microsoft/Windows-driver-samples) - Microsoft's official WDK samples repository
- [IRP_MJ_DEVICE_CONTROL](https://docs.microsoft.com/en-us/windows-hardware/drivers/kernel/irp-mj-device-control) - IOCTL handling in drivers
- [Plug and Play Minor IRPs](https://docs.microsoft.com/en-us/windows-hardware/drivers/kernel/plug-and-play-minor-irps) - PnP IRP handling
- [_FAST_IO_DISPATCH structure](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/content/wdm/ns-wdm-_fast_io_dispatch) - Fast I/O operations
- [Filtering IRPs and Fast I/O](https://docs.microsoft.com/en-us/windows-hardware/drivers/ifs/filtering-irps-and-fast-i-o) - IRP filtering techniques
- [Windows File System Filter Driver Development](https://www.apriorit.com/dev-blog/167-file-system-filter-driver) - Filter driver guide
- [Setting Up Local Kernel Debugging](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/setting-up-local-kernel-debugging-of-a-single-computer-manually) - Local debugging setup

## Hypervisor Projects & Implementations

- [HyperPlatform](https://github.com/tandasat/HyperPlatform) - Intel VT-x based hypervisor
- [HyperPlatform User Documentation](https://tandasat.github.io/HyperPlatform/userdocument/)
- [HVPP](https://github.com/wbenny/hvpp) - C++ VT-x hypervisor library
- [HyperBone](https://github.com/DarthTon/HyperBone) - Minimalistic VT-x hypervisor with hooks
- [SimpleSvmHook](https://github.com/tandasat/SimpleSvmHook) - AMD-V based hypervisor example
- [Hypervisor For Beginners](https://github.com/rohaaan/hypervisor-for-beginners) - Educational hypervisor project
- [Gbhv](https://github.com/Gbps/gbhv) - Simple x64 hypervisor framework
- [DdiMon](https://github.com/tandasat/DdiMon) - Hypervisor-based monitoring tool
- [TitanHide](https://github.com/dotfornet/TitanHide) - Anti-detection hypervisor driver
- [VmcsAuditor](https://rayanfam.com/topics/vmcsauditor-a-bochs-based-hypervisor-layout-checker/) - Bochs-based hypervisor layout checker

## Extended Page Tables (EPT)

- [Performance Evaluation of Intel EPT](https://www.vmware.com/pdf/Perf_ESX_Intel-EPT-eval.pdf) - VMware whitepaper
- [Second Level Address Translation](https://en.wikipedia.org/wiki/Second_Level_Address_Translation) - SLAT overview
- [Memory Virtualization Slides](http://www.cs.nthu.edu.tw/~ychung/slides/Virtualization/VM-Lecture-2-2-SystemVirtualizationMemory.pptx) - Academic presentation
- [Best Practices for EPT and VT-d](https://software.intel.com/en-us/articles/best-practices-for-paravirtualization-enhancements-from-intel-virtualization-technology-ept-and-vt-d)
- [5-Level Paging and EPT](https://software.intel.com/sites/default/files/managed/2b/80/5-level_paging_white_paper.pdf) - Intel whitepaper
- [Xen Summit 2007 - EPT Presentation](http://www-archive.xenproject.org/files/xensummit_fall07/12_JunNakajima.pdf)

## Memory Management & Paging

- [Introduction to IA-32e Hardware Paging](https://www.triplefault.io/2017/07/introduction-to-ia-32e-hardware-paging.html)
- [x86 Paging Tutorial](https://cirosantilli.com/x86-paging) - Comprehensive paging guide
- [OSDev notes: Memory Management](http://ethv.net/workshops/osdev/notes/notes-2)
- [Memory Type Range Registers (MTRR)](https://en.wikipedia.org/wiki/Memory_type_range_register)
- [Memory Caching Types](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/content/wdm/ne-wdm-_memory_caching_type)
- [Write-back Cache Explained](https://whatis.techtarget.com/definition/write-back)
- [Inside Windows PFN Part 1](https://rayanfam.com/topics/inside-windows-page-frame-number-part1)
- [Inside Windows PFN Part 2](https://rayanfam.com/topics/inside-windows-page-frame-number-part2)

## x86/x64 Architecture & Instructions

- [Segmentation](https://wiki.osdev.org/Segmentation) - OSDev Wiki
- [x86 Memory Segmentation](https://en.wikipedia.org/wiki/X86_memory_segmentation)
- [x86 Calling Conventions](https://en.wikipedia.org/wiki/X86_calling_conventions)
- [SWAPGS Instruction](https://www.felixcloutier.com/x86/SWAPGS.html)
- [RDTSCP Instruction](https://www.felixcloutier.com/x86/rdtscp)
- [INVPCID Instruction](https://www.felixcloutier.com/x86/invpcid)
- [INVVPID Instruction](https://www.felixcloutier.com/x86/invvpid)
- [XSAVE Instruction](https://www.felixcloutier.com/x86/xsave)
- [XRSTORS Instruction](https://www.felixcloutier.com/x86/xrstors)
- [PAUSE Instruction](https://c9x.me/x86/html/file_module_x86_id_232.html)
- [Exceptions Reference](https://wiki.osdev.org/Exceptions)

## Virtual Processor Identifiers (VPID)

- [Virtual Processor IDs and TLB](http://www.jauu.net/2011/11/13/virtual-processor-ids-and-tlb/)
- [Spectre and Meltdown Performance Impact](https://arstechnica.com/gadgets/2018/01/heres-how-and-why-the-spectre-and-meltdown-patches-will-hurt-performance/)

## Windows Internals & Kernel Programming

- [What is IRQL?](https://blogs.msdn.microsoft.com/doronh/2010/02/02/what-is-irql/)
- [Non-Paged Pool and DISPATCH_LEVEL](https://stackoverflow.com/questions/18764211/why-we-can-access-memory-from-non-paged-pool-at-or-above-dispatch-level)
- [KVA Shadow: Mitigating Meltdown](https://msrc-blog.microsoft.com/2018/03/23/kva-shadow-mitigating-meltdown-on-windows/)
- [Windows Hotpatching](https://jpassing.com/2011/05/03/windows-hotpatching-a-walkthrough/)
- [R.I.P ROP: CET Internals](http://windows-internals.com/cet-on-windows)
- [PAGED_CODE Macro](https://technet.microsoft.com/en-us/ff558773(v=vs.96))
- [Deferred Procedure Calls](https://en.wikipedia.org/wiki/Deferred_Procedure_Call)
- [Reversing DPC: KeInsertQueueDpc](https://repnz.github.io/posts/practical-reverse-engineering/reversing-dpc-keinsertqueuedpc/)
- [Dumping DPC Queues](https://repnz.github.io/posts/practical-reverse-engineering/dumping-dpc-queues/)
- [WPP Software Tracing](https://docs.microsoft.com/en-us/windows-hardware/drivers/devtest/wpp-software-tracing)
- [TraceView Tool](https://docs.microsoft.com/en-us/windows-hardware/drivers/devtest/traceview)
- [Add WPP Tracing to Kernel Driver](http://kernelpool.blogspot.com/2018/05/add-wpp-tracing-to-kernel-mode-windows.html)

## Driver Development - IOCTLs

- [Introduction to Implementing IOCTLs](https://www.codeproject.com/Articles/9575/Driver-Development-Part-2-Introduction-to-Implemen)
- [Buffer Descriptions for I/O Control Codes](https://docs.microsoft.com/en-us/windows-hardware/drivers/kernel/buffer-descriptions-for-i-o-control-codes)
- [Defining I/O Control Codes](https://docs.microsoft.com/en-us/windows-hardware/drivers/kernel/defining-i-o-control-codes)

## Synchronization & Concurrency

- [Test-and-Set](https://en.wikipedia.org/wiki/Test-and-set)
- [_interlockedbittestandset Intrinsics](https://docs.microsoft.com/en-us/cpp/intrinsics/interlockedbittestandset-intrinsic-functions?view=vs-2019)
- [Spinlocks and Read-Write Locks](https://locklessinc.com/articles/locks/)
- [PAUSE Instruction in Spinlocks](https://stackoverflow.com/questions/12894078/what-is-the-purpose-of-the-pause-instruction-in-x86)
- [How PAUSE Works in Spinlocks](https://stackoverflow.com/questions/4725676/how-does-x86-pause-instruction-work-in-spinlock-and-can-it-be-used-in-other-sc)
- [volatile Keyword Introduction](https://www.embedded.com/introduction-to-the-volatile-keyword/)

## Hooking & Interception

- [Syscall Hooking Via EFER](https://revers.engineering/syscall-hooking-via-extended-feature-enable-register-efer/)
- [Software-based SMEP with Hypervisors](http://hypervsir.blogspot.com/2014/11/how-to-implement-software-based.html)
- [System Service Descriptor Table (SSDT)](https://ired.team/miscellaneous-reversing-forensics/windows-kernel/glimpse-into-ssdt-in-windows-x64-kernel)
- [Hook SSDT Shadow](https://m0uk4.gitbook.io/notebooks/mouka/windowsinternal/ssdt-hook)
- [DetourXS](https://github.com/DominicTobias/detourxs) - Inline hooking library
- [Nt Syscall Table](https://j00ru.vexillium.org/syscalls/nt/64/)
- [Win32k Syscall Table](https://j00ru.vexillium.org/syscalls/win32k/64/)

## VM Exits & Event Handling

- [VM-Exit Handler & Event Injection](https://revers.engineering/day-5-vmexits-interrupts-cpuid-emulation/)
- [Trap vs Interrupt](https://stackoverflow.com/questions/3149175/what-is-the-difference-between-trap-and-interrupt)
- [Knockin' on Heaven's Gate](http://rce.co/knockin-on-heavens-gate-dynamic-processor-mode-switching/) - Mode switching

## Hyper-V & Nested Virtualization

- [Disable Hyper-V via Command Line](https://stackoverflow.com/questions/30496116/how-to-disable-hyper-v-in-command-line)
- [Run Hyper-V with Nested Virtualization](https://docs.microsoft.com/en-us/virtualization/hyper-v-on-windows/user-guide/nested-virtualization)
- [Hypervisor Top-Level Functional Specification (TLFS)](https://docs.microsoft.com/en-us/virtualization/hyper-v-on-windows/reference/tlfs)
- [Requirements for Microsoft Hypervisor Interface](https://github.com/Microsoft/Virtualization-Documentation/raw/master/tlfs/Requirements%20for%20Implementing%20the%20Microsoft%20Hypervisor%20Interface.pdf)

## Security & Advanced Topics

- [Intel SGX Explained](https://www.semanticscholar.org/paper/Intel-SGX-Explained-Costan-Devadas/2d7f3f4ca3fbb15ae04533456e5031e0d0dc845a)
- [Intel VT-x Notes](https://github.com/tnballo/notebook/wiki/Intel-VTx)
- [HyperPlatform Issues: VMXOFF Safety](https://github.com/tandasat/HyperPlatform/issues/3)

## Miscellaneous Resources

- [What Is a Type 1 Hypervisor?](http://www.virtualizationsoftware.com/type-1-hypervisors/)
- Setting Up KDNET Network Kernel Debugging (Manually and Automatically)