# Getting Started with Hypervisor Development

This guide provides step-by-step instructions for setting up your development environment to build and test the hypervisor project on Windows.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installing Development Tools](#installing-development-tools)
- [Setting Up the Testing Environment](#setting-up-the-testing-environment)
- [Creating Your First Driver](#creating-your-first-driver)
- [Configuring Windows for Driver Development](#configuring-windows-for-driver-development)
- [Setting Up Nested Virtualization](#setting-up-nested-virtualization)

## Prerequisites

Before you begin, ensure you have:
- A Windows system with administrator privileges
- A CPU with Intel VT-x or AMD-V support (check your CPU specifications)
- At least 8GB of RAM (16GB recommended for nested virtualization)
- 50GB of free disk space

**Note:** This project requires kernel-level development. You'll be working with Windows drivers that run at Ring 0. Be prepared for potential system instability and Blue Screens of Death (BSODs) during development and testing.

## Installing Development Tools

### Step 1: Install Visual Studio

Download and install the free Community edition of Visual Studio:
- [Visual Studio Community](https://visualstudio.microsoft.com/vs/community)

During installation, select the **Desktop development with C++** workload.

### Step 2: Install Windows Driver Kit (WDK)

After Visual Studio is installed, download and install the WDK:
- [Windows Driver Kit (WDK)](https://docs.microsoft.com/en-us/windows-hardware/drivers/download-the-wdk)

Make sure to install the WDK version that matches your Visual Studio version.

### Step 3: Install Windows SDK and WinDbg

WinDbg is the kernel debugger we'll use to debug our hypervisor:
- [Windows SDK](https://developer.microsoft.com/en-us/windows/downloads/windows-sdk/)
- Alternatively, install **WinDbg Preview** from the Microsoft Store (recommended)

### Step 4: Install OSR Driver Loader

OSR Driver Loader is a simple tool for loading and unloading test drivers:
- [OSR Driver Loader](https://www.osronline.com/article.cfm?article=157)

### Step 5: Install DebugView

DebugView allows you to view debug output from kernel drivers:
- [SysInternals DebugView](https://docs.microsoft.com/en-us/sysinternals/downloads/debugview)

Download and extract it to a convenient location.

## Setting Up the Testing Environment

### Project Setup

Hypervisor development requires kernel-level code execution through a Windows Driver. Since WDK doesn't support inline assembly in x64 mode, you'll need to:

1. Create a new **Kernel Mode Driver (KMDF)** project in Visual Studio
2. Configure the project for x64 architecture
3. Add support for external assembly files (.asm) using MASM

### Enabling Inline Assembly

For assembly code:
- Create separate `.asm` files for assembly routines
- Add them to your project
- Configure MASM build customization
- Export functions to be called from C/C++ code

Refer to the project source code for examples of properly configured assembly integration.

## Creating Your First Driver

### Basic Driver Structure

A Windows kernel driver requires two main functions:

1. **DriverEntry**: Entry point called when the driver loads
2. **DrvUnload**: Cleanup function called when the driver unloads

### Device Registration

The driver creates a device object to enable user-mode communication:

```c
RtlInitUnicodeString(&DriverName, L"\\Device\\MyHypervisor");
RtlInitUnicodeString(&DosDeviceName, L"\\DosDevices\\MyHypervisor");

NtStatus = IoCreateDevice(DriverObject, 0, &DriverName, 
                         FILE_DEVICE_UNKNOWN, FILE_DEVICE_SECURE_OPEN, 
                         FALSE, &DeviceObject);

if (NtStatus == STATUS_SUCCESS)
{
    DriverObject->DriverUnload = DrvUnload;
    DeviceObject->Flags |= IO_TYPE_DEVICE;
    DeviceObject->Flags &= (~DO_DEVICE_INITIALIZING);
    IoCreateSymbolicLink(&DosDeviceName, &DriverName);
}
```

### Complete Driver Example

Here's a minimal driver that demonstrates the basic structure:

```c
#include <ntddk.h>
#include <wdf.h>
#include <wdm.h>

NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath);
VOID DrvUnload(PDRIVER_OBJECT DriverObject);

#pragma alloc_text(INIT, DriverEntry)
#pragma alloc_text(PAGE, DrvUnload)

NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath)
{
    NTSTATUS       NtStatus     = STATUS_SUCCESS;
    PDEVICE_OBJECT DeviceObject = NULL;
    UNICODE_STRING DriverName, DosDeviceName;

    DbgPrint("[Hypervisor] DriverEntry Called");

    RtlInitUnicodeString(&DriverName, L"\\Device\\MyHypervisor");
    RtlInitUnicodeString(&DosDeviceName, L"\\DosDevices\\MyHypervisor");

    NtStatus = IoCreateDevice(DriverObject, 0, &DriverName, 
                             FILE_DEVICE_UNKNOWN, FILE_DEVICE_SECURE_OPEN, 
                             FALSE, &DeviceObject);

    if (NtStatus == STATUS_SUCCESS)
    {
        DriverObject->DriverUnload = DrvUnload;
        DeviceObject->Flags |= IO_TYPE_DEVICE;
        DeviceObject->Flags &= (~DO_DEVICE_INITIALIZING);
        IoCreateSymbolicLink(&DosDeviceName, &DriverName);
    }
    return NtStatus;
}

VOID DrvUnload(PDRIVER_OBJECT DriverObject)
{
    UNICODE_STRING DosDeviceName;

    DbgPrint("[Hypervisor] DrvUnload Called");

    RtlInitUnicodeString(&DosDeviceName, L"\\DosDevices\\MyHypervisor");
    IoDeleteSymbolicLink(&DosDeviceName);
    IoDeleteDevice(DriverObject->DeviceObject);
}
```

**Note:** The complete source code for this project is available on GitHub. This basic driver template will be expanded with hypervisor functionality in subsequent development.

## Configuring Windows for Driver Development

### Disable Driver Signature Enforcement (DSE)

Windows prevents unsigned drivers from loading in kernel mode. For development, you need to disable this enforcement:

1. **Hold Shift** and click **Restart** in Windows
2. Navigate to **Troubleshoot** → **Advanced options** → **Startup Settings**
3. Click **Restart**
4. Press **F7** or **7** to select "Disable driver signature enforcement"

**Important:** This setting is temporary and resets after each reboot. You'll need to repeat this process each time you restart your development machine.

### Enable Debug Output

To view `DbgPrint()` messages from your driver:

1. Create a file named `dbgview.reg` with the following content:

```reg
Windows Registry Editor Version 5.00

[HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Session Manager\Debug Print Filter]
"DEFAULT"=dword:0000000f
```

2. Double-click the file to merge it into the registry
3. Reboot your system
4. Launch **DebugView** with administrator privileges
5. Enable **Capture Kernel** in the Capture menu

### Setup Kernel Debugging (Optional but Recommended)

For advanced debugging, set up kernel debugging over network (KDNET):

1. On the **target machine** (where the driver runs), run as administrator:
   ```cmd
   bcdedit /debug on
   bcdedit /dbgsettings net hostip:<debugger-ip> port:50000
   ```

2. Note the generated key

3. On the **host machine** (debugger), launch WinDbg and connect using the IP and key

For detailed instructions, search for "Setting Up KDNET Network Kernel Debugging" in the Windows documentation.

## Setting Up Nested Virtualization

If you don't have access to a physical machine, you can develop and test using nested virtualization.

### VMware Workstation

1. Power off your virtual machine
2. Edit the VM settings
3. Navigate to **Processors**
4. Check **Virtualize Intel VT-x/EPT or AMD-V/RVI**
5. (Optional) Check **Virtualize CPU performance counters**
6. Save and start the VM

Verify VT-x is enabled in the guest:
```cmd
systeminfo
```
Look for "Virtualization Enabled In Firmware: Yes"

### Hyper-V

Hyper-V nested virtualization has limitations. Basic hypervisor testing may not work properly until you implement specific compatibility features. 

To enable nested virtualization on Hyper-V:

1. Power off the VM
2. Run on the **host** (in PowerShell as Administrator):
   ```powershell
   Set-VMProcessor -VMName <YourVMName> -ExposeVirtualizationExtensions $true
   ```
3. Start the VM

**Note:** Some hypervisor features may require additional modifications to work under Hyper-V. Physical hardware is recommended for initial development.

### VirtualBox

VirtualBox supports nested virtualization on AMD and newer Intel CPUs:

1. Power off the VM
2. Go to **Settings** → **System** → **Acceleration**
3. Check **Enable Nested VT-x/AMD-V**
4. Save and start the VM

## Loading and Testing Your Driver

### Using OSR Driver Loader

1. Build your driver project in Visual Studio (Release or Debug configuration)
2. Locate the compiled `.sys` file (usually in `x64\Debug` or `x64\Release`)
3. Open **OSR Driver Loader** as Administrator
4. Click **Browse** and select your `.sys` file
5. Click **Register Service**
6. Click **Start Service** to load the driver
7. Check DebugView for output messages

### Verifying Driver Operation

If everything is set up correctly, you should see:
- "[Hypervisor] DriverEntry Called" message in DebugView
- No BSOD or system crashes
- Driver listed in OSR Driver Loader

To unload the driver:
1. Click **Stop Service** in OSR Driver Loader
2. Check DebugView for "[Hypervisor] DrvUnload Called" message

## Next Steps

Once your development environment is configured:

1. **Build the project** - Compile the hypervisor source code from GitHub
2. **Verify CPU support** - Ensure VT-x/AMD-V is enabled in BIOS
3. **Test basic driver loading** - Confirm the driver loads without errors
4. **Set up debugging** - Configure WinDbg for kernel debugging if needed
5. **Read the documentation** - Review the conceptual docs and references in the `RESOURCES.md` file

## Troubleshooting

### Common Issues

**Driver won't load:**
- Ensure Driver Signature Enforcement is disabled
- Run OSR Driver Loader as Administrator
- Check Windows Event Viewer for error messages

**No debug output in DebugView:**
- Verify registry setting is applied and system is rebooted
- Enable "Capture Kernel" in DebugView
- Run DebugView as Administrator

**BSOD on driver load:**
- Review your code for errors
- Set up kernel debugging with WinDbg for detailed crash analysis
- Test on a VM first, not your main development machine

**Nested virtualization not working:**
- Verify VT-x/AMD-V is enabled in the VM settings
- Check that the host system supports nested virtualization
- Some features may require physical hardware

## Additional Resources

For more detailed information about hypervisor concepts, technical references, and related projects, see the [RESOURCES.md](RESOURCES.md) file.