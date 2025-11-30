# Hypervisor From Scratch

A Windows kernel-mode hypervisor implementation using Intel VT-x (VMX) technology for learning and experimentation.

## What Is This?

This repository contains a complete, working hypervisor implementation for Intel x64 processors. It demonstrates how to:

- Enable and configure Intel VT-x (Virtual Machine Extensions)
- Create and manage Virtual Machine Control Structures (VMCS)
- Handle VM exits and hypervisor events
- Implement Extended Page Tables (EPT) for memory virtualization
- Use VMCALL for guest-to-hypervisor communication
- Virtualize an already-running Windows system

## Key Features

### Hypervisor Capabilities
- **VMX Operation** - Full Intel VT-x support for entering and managing VMX root/non-root modes
- **VMCS Management** - Configure and manipulate Virtual Machine Control Structures
- **EPT (Extended Page Tables)** - Second-level address translation and memory isolation
- **VM Exit Handling** - Handle various VM exit reasons (CPUID, MSR access, I/O, etc.)
- **VPID Support** - Virtual Processor Identifiers for TLB optimization
- **Hidden Hooks** - EPT-based page-level monitoring and hooking
- **Syscall Hooking** - Intercept system calls using hypervisor capabilities
- **Live Virtualization** - Virtualize a running Windows system without reboot

### Components
- **Kernel Driver** - Complete hypervisor implementation with EPT, VMCS, and exit handlers
- **User-Mode App** - Control interface for loading/unloading and interacting with the hypervisor
- **Assembly Modules** - Low-level VMX operations (VMXON, VMLAUNCH, VMRESUME, etc.)
- **Memory Manager** - Pool allocation and management for hypervisor structures

## Getting Started

### Prerequisites

- **Operating System:** Windows 10/11 (x64)
- **Development Tools:**
  - Visual Studio 2019 or later
  - Windows Driver Kit (WDK) 10
  - Windows SDK
- **Hardware:** Intel CPU with VT-x support (check BIOS settings)
- **Test Environment:** Physical machine or nested virtualization (VMware/Hyper-V)

### Quick Start

1. **Read the documentation:**
   ```
   docs/getting_started.md     - Setup instructions
   docs/vmx_operations.md      - Technical details
   ```

2. **Build the project:**
   - Open `Hypervisor From Scratch.sln` in Visual Studio
   - Select x64 platform
   - Build solution (Ctrl+Shift+B)

3. **Load the driver:**
   - Disable Driver Signature Enforcement (see `docs/getting_started.md`)
   - Use OSR Driver Loader or sc.exe to load `MyHypervisorDriver.sys`

4. **Run the application:**
   - Execute `MyHypervisorApp.exe` as Administrator
   - Follow on-screen instructions to interact with the hypervisor

## Use Cases

This hypervisor is designed for:

- **Learning** - Understanding Intel VT-x and hypervisor internals
- **Research** - Experimenting with virtualization technologies
- **Security Research** - Implementing EPT hooks and memory monitoring
- **Reverse Engineering** - Low-level system analysis and instrumentation
- **Development** - Building custom hypervisor-based tools


## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
