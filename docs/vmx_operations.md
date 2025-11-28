# Entering VMX Operation

This document covers the process of enabling Intel VT-x and entering VMX operation, including Windows Driver Kit (WDK) IRP handling, detecting virtualization support, and configuring the processor for hypervisor operation.

## Prerequisites

Before proceeding, ensure you have:
- Completed the development environment setup from `getting_started.md`
- A working Windows kernel driver project
- Understanding of basic driver concepts
- CPU with Intel VT-x support

## Table of Contents

- [IRP Major Functions](#irp-major-functions)
- [Detecting Hypervisor Support](#detecting-hypervisor-support)
- [Enabling VMX Operation](#enabling-vmx-operation)
- [Testing and Verification](#testing-and-verification)

## Overview

Enabling VMX (Virtual Machine Extensions) is the first step in creating a hypervisor. This process involves:

1. **Setting up driver communication** - Implementing IRP handlers for user-mode/kernel-mode communication
2. **Detecting VMX support** - Using CPUID to verify processor capabilities
3. **Enabling VMX operation** - Setting the CR4.VMXE bit and configuring MSRs
4. **Entering VMX root operation** - Executing VMXON instruction

## IRP Major Functions

### Overview

IRP (I/O Request Packet) Major Functions provide the mechanism for user-mode applications to communicate with kernel-mode drivers. Each device driver maintains a dispatch table of function pointers that Windows calls when specific operations are requested.

**Best Practice:** Minimize kernel-mode code complexity. Implement as much logic as possible in user-mode to reduce the risk of system crashes and simplify debugging.

### Understanding IRPs

An IRP is a data structure that represents an I/O request. It contains:
- Caller information and security context
- Request parameters and buffers  
- I/O status and completion information
- Stack locations for layered drivers

**Security Note:** Always validate caller privileges and parameters when processing IRPs. The kernel must not trust user-mode input.

### Configuring IRP Handlers

After creating a device object, configure the IRP dispatch table. Here's how to set up the major function handlers:

```c
if (NtStatus == STATUS_SUCCESS)
{
    // Initialize all handlers to unsupported
    for (Index = 0; Index < IRP_MJ_MAXIMUM_FUNCTION; Index++)
    {
        DriverObject->MajorFunction[Index] = DrvUnsupported;
    }

    DbgPrint("[Hypervisor] Configuring IRP major functions");
    
    // Set supported IRP handlers
    DriverObject->MajorFunction[IRP_MJ_CREATE]         = DrvCreate;
    DriverObject->MajorFunction[IRP_MJ_CLOSE]          = DrvClose;
    DriverObject->MajorFunction[IRP_MJ_DEVICE_CONTROL] = DrvIoctlDispatcher;
    DriverObject->MajorFunction[IRP_MJ_READ]           = DrvRead;
    DriverObject->MajorFunction[IRP_MJ_WRITE]          = DrvWrite;
    
    DriverObject->DriverUnload = DrvUnload;
    IoCreateSymbolicLink(&DosDeviceName, &DriverName);
}
else
{
    DbgPrint("[Hypervisor] Failed to create device: 0x%X", NtStatus);
}
```

### Implementing IRP Handlers

**Unsupported Handler** - Returns success for unimplemented operations:

```c
NTSTATUS DrvUnsupported(IN PDEVICE_OBJECT DeviceObject, IN PIRP Irp)
{
    DbgPrint("[Hypervisor] Unsupported IRP function called");
    
    Irp->IoStatus.Status      = STATUS_NOT_SUPPORTED;
    Irp->IoStatus.Information = 0;
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
    
    return STATUS_NOT_SUPPORTED;
}
```

**Read/Write/Close Handlers** - Placeholder implementations:

```c
NTSTATUS DrvRead(IN PDEVICE_OBJECT DeviceObject, IN PIRP Irp)
{
    DbgPrint("[Hypervisor] IRP_MJ_READ - Not implemented");
    
    Irp->IoStatus.Status      = STATUS_SUCCESS;
    Irp->IoStatus.Information = 0;
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
    
    return STATUS_SUCCESS;
}

NTSTATUS DrvWrite(IN PDEVICE_OBJECT DeviceObject, IN PIRP Irp)
{
    DbgPrint("[Hypervisor] IRP_MJ_WRITE - Not implemented");
    
    Irp->IoStatus.Status      = STATUS_SUCCESS;
    Irp->IoStatus.Information = 0;
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
    
    return STATUS_SUCCESS;
}

NTSTATUS DrvClose(IN PDEVICE_OBJECT DeviceObject, IN PIRP Irp)
{
    DbgPrint("[Hypervisor] IRP_MJ_CLOSE called");
    
    Irp->IoStatus.Status      = STATUS_SUCCESS;
    Irp->IoStatus.Information = 0;
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
    
    return STATUS_SUCCESS;
}
```

### IRP Major Function Reference

Windows defines the following IRP major functions:

```c
#define IRP_MJ_CREATE                   0x00
#define IRP_MJ_CREATE_NAMED_PIPE        0x01
#define IRP_MJ_CLOSE                    0x02
#define IRP_MJ_READ                     0x03
#define IRP_MJ_WRITE                    0x04
#define IRP_MJ_QUERY_INFORMATION        0x05
#define IRP_MJ_SET_INFORMATION          0x06
#define IRP_MJ_QUERY_EA                 0x07
#define IRP_MJ_SET_EA                   0x08
#define IRP_MJ_FLUSH_BUFFERS            0x09
#define IRP_MJ_QUERY_VOLUME_INFORMATION 0x0a
#define IRP_MJ_SET_VOLUME_INFORMATION   0x0b
#define IRP_MJ_DIRECTORY_CONTROL        0x0c
#define IRP_MJ_FILE_SYSTEM_CONTROL      0x0d
#define IRP_MJ_DEVICE_CONTROL           0x0e
#define IRP_MJ_INTERNAL_DEVICE_CONTROL  0x0f
#define IRP_MJ_SHUTDOWN                 0x10
#define IRP_MJ_LOCK_CONTROL             0x11
#define IRP_MJ_CLEANUP                  0x12
#define IRP_MJ_CREATE_MAILSLOT          0x13
#define IRP_MJ_QUERY_SECURITY           0x14
#define IRP_MJ_SET_SECURITY             0x15
#define IRP_MJ_POWER                    0x16
#define IRP_MJ_SYSTEM_CONTROL           0x17
#define IRP_MJ_DEVICE_CHANGE            0x18
#define IRP_MJ_QUERY_QUOTA              0x19
#define IRP_MJ_SET_QUOTA                0x1a
#define IRP_MJ_PNP                      0x1b
#define IRP_MJ_PNP_POWER                IRP_MJ_PNP
#define IRP_MJ_MAXIMUM_FUNCTION         0x1b
```

### User-Mode to Kernel-Mode Mapping

Windows treats devices as files. User-mode API calls map to IRP major functions:

| User-Mode API | IRP Major Function |
|---------------|-------------------|
| `CreateFile()` / `CreateFileA()` / `CreateFileW()` | `IRP_MJ_CREATE` |
| `ReadFile()` | `IRP_MJ_READ` |
| `WriteFile()` | `IRP_MJ_WRITE` |
| `CloseHandle()` | `IRP_MJ_CLOSE` |
| `DeviceIoControl()` | `IRP_MJ_DEVICE_CONTROL` |

Windows automatically copies user-mode buffers to kernel-mode when passing IRPs.

**Note:** IRP Minor Functions exist for more granular control but are not covered in this guide.

## Detecting Hypervisor Support

### Overview

Before enabling VMX, verify that the processor supports Intel VT-x. This is documented in Intel SDM Volume 3C, Section 23.6.

**Detection Steps:**
1. Verify Intel processor (vendor string = "GenuineIntel")
2. Check VMX support via CPUID (CPUID.1:ECX.VMX[bit 5] = 1)

### Detecting CPU Vendor

Use the CPUID instruction with EAX=0 to retrieve the CPU vendor string:

```cpp
std::string GetCpuID()
{
    char   SysType[13];
    string CpuID;
    
    _asm
    {
        // Execute CPUID with EAX = 0 to get CPU vendor
        XOR EAX, EAX
        CPUID
        
        // Extract vendor string from EBX, EDX, ECX
        MOV EAX, EBX
        MOV SysType[0], AL
        MOV SysType[1], AH
        SHR EAX, 16
        MOV SysType[2], AL
        MOV SysType[3], AH
        
        MOV EAX, EDX
        MOV SysType[4], AL
        MOV SysType[5], AH
        SHR EAX, 16
        MOV SysType[6], AL
        MOV SysType[7], AH
        
        MOV EAX, ECX
        MOV SysType[8], AL
        MOV SysType[9], AH
        SHR EAX, 16
        MOV SysType[10], AL
        MOV SysType[11], AH
        MOV SysType[12], 0
    }
    
    CpuID.assign(SysType, 12);
    return CpuID;
}
```

### Detecting VMX Support

Check bit 5 of ECX after executing CPUID with EAX=1:

```cpp
bool DetectVmxSupport()
{
    bool VMX = false;
    
    __asm
    {
        XOR    EAX, EAX
        INC    EAX          // EAX = 1
        CPUID
        BT     ECX, 0x5     // Test bit 5 (VMX support)
        JC     VMXSupport
        JMP    NopInstr
        
VMXSupport:
        MOV    VMX, 0x1
        
NopInstr:
        NOP
    }
    
    return VMX;
}
```

This checks CPUID leaf 1 and tests bit 5 of ECX. If set, VMX is supported.

### User-Mode Detection Example

```cpp
int main()
{
    std::string CpuId = GetCpuID();
    
    printf("[*] CPU Vendor: %s\n", CpuId.c_str());
    
    if (CpuId != "GenuineIntel")
    {
        printf("[!] Error: This program requires an Intel processor\n");
        return 1;
    }
    
    printf("[*] Processor virtualization technology: VT-x\n");
    
    if (!DetectVmxSupport())
    {
        printf("[!] Error: VMX operation not supported by processor\n");
        return 1;
    }
    
    printf("[+] VMX operation is supported\n");
    
    // Open handle to hypervisor device
    HANDLE hDevice = CreateFile(L"\\\\.\\MyHypervisorDevice",
                                GENERIC_READ | GENERIC_WRITE,
                                FILE_SHARE_READ | FILE_SHARE_WRITE,
                                NULL,
                                OPEN_EXISTING,
                                FILE_ATTRIBUTE_NORMAL | FILE_FLAG_OVERLAPPED,
                                NULL);
    
    if (hDevice == INVALID_HANDLE_VALUE)
    {
        printf("[!] Error: Failed to open device handle\n");
        return 1;
    }
    
    printf("[+] Device handle opened successfully\n");
    
    CloseHandle(hDevice);
    return 0;
}
```

## Enabling VMX Operation

### Overview

Once VMX support is confirmed, enable it by:
1. Setting CR4.VMXE bit (bit 13)
2. Configuring IA32_FEATURE_CONTROL MSR
3. Executing VMXON instruction

### CR4.VMXE Bit

Before entering VMX operation, set **CR4.VMXE[bit 13] = 1**. 

**Important Notes:**
- VMXON causes `#UD` (invalid opcode) exception if CR4.VMXE = 0
- Once in VMX operation, CR4.VMXE cannot be cleared
- VMXOFF instruction exits VMX operation and allows clearing CR4.VMXE

### IA32_FEATURE_CONTROL MSR

The IA32_FEATURE_CONTROL MSR (address 0x3A) controls VMX enablement:

**Bit 0 - Lock Bit:**
- If clear: VMXON causes `#GP` (general protection) exception
- If set: MSR cannot be modified until system reset
- BIOS uses this to provide VMX enable/disable option

**Purpose:** Prevents VMX from being disabled without a full system reset once locked.

### Assembly Implementation

**Header declaration** (e.g., `Source.h`):

```c
extern void inline AsmEnableVmxOperation(void);
```

**Assembly implementation** (e.g., `SourceAsm.asm`):

```asm
PUBLIC AsmEnableVmxOperation

AsmEnableVmxOperation PROC PUBLIC
    PUSH RAX              ; Save state
    
    XOR  RAX, RAX         ; Clear RAX
    MOV  RAX, CR4         ; Read CR4
    OR   RAX, 02000h      ; Set bit 13 (0x2000)
    MOV  CR4, RAX         ; Write CR4
    
    POP  RAX              ; Restore state
    RET
AsmEnableVmxOperation ENDP
```

**Note:** Bit 13 is the 14th bit when counting from 1. The hex value 0x2000 = binary 0010 0000 0000 0000.

### Driver Implementation

Call the assembly function in the `IRP_MJ_CREATE` handler:

```c
NTSTATUS DrvCreate(IN PDEVICE_OBJECT DeviceObject, IN PIRP Irp)
{
    // Enable VMX operation
    AsmEnableVmxOperation();
    DbgPrint("[Hypervisor] VMX operation enabled successfully");
    
    Irp->IoStatus.Status      = STATUS_SUCCESS;
    Irp->IoStatus.Information = 0;
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
    
    return STATUS_SUCCESS;
}
```

### Triggering from User-Mode

Open the device to trigger `IRP_MJ_CREATE`:

```cpp
HANDLE hDevice = CreateFile(L"\\\\.\\MyHypervisorDevice",
                            GENERIC_READ | GENERIC_WRITE,
                            FILE_SHARE_READ | FILE_SHARE_WRITE,
                            NULL,
                            OPEN_EXISTING,
                            FILE_ATTRIBUTE_NORMAL | FILE_FLAG_OVERLAPPED,
                            NULL);
                            
if (hDevice == INVALID_HANDLE_VALUE)
{
    printf("[!] Failed to open device\n");
    return 1;
}

printf("[+] VMX enabled successfully\n");
CloseHandle(hDevice);
```

**Important:** Name your `.asm` file differently from your `.c` file to avoid linker errors in Visual Studio. For example:
- Driver: `Source.c`
- Assembly: `SourceAsm.asm` 

## Testing and Verification

### Expected Output

In DebugView, you should see:
```
[Hypervisor] DriverEntry Called
[Hypervisor] Configuring IRP major functions
[Hypervisor] VMX operation enabled successfully
```

### Verification Steps

1. Build the driver project
2. Load the driver with OSR Driver Loader
3. Run the user-mode application
4. Check DebugView for success messages
5. Use SysInternals WinObj to verify device creation

### Troubleshooting

**No debug output:**
- Verify debug output is enabled 
- Ensure DebugView is running as Administrator
- Check "Capture Kernel" is enabled in DebugView

**BSOD or #GP exception:**
- Verify VMX is enabled in BIOS
- Check IA32_FEATURE_CONTROL MSR lock bit
- Ensure CPUID detection passed before enabling VMX

**Device not found:**
- Verify driver loaded successfully
- Check Windows Event Viewer for errors
- Ensure driver signature enforcement is disabled

## Summary

This document covered:
- Implementing IRP major function handlers for driver communication
- Detecting Intel VT-x support using CPUID
- Enabling VMX operation by setting CR4.VMXE bit
- Creating assembly routines in WDK drivers

**Next Steps:** Implement VMXON execution and enter VMX root operation mode. See additional documentation and the [RESOURCES.md](RESOURCES.md) file for further reading.