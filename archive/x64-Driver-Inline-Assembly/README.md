# x64 Driver Inline Assembly

A demonstration Windows kernel driver showing how to integrate x64 assembly code in kernel-mode drivers using the Windows Driver Kit (WDK).

## Overview

This project demonstrates the proper method for including assembly language code in x64 (64-bit) Windows kernel drivers. Unlike 32-bit drivers where inline assembly (`__asm` blocks) is supported, x64 drivers require a different approach using separate `.asm` files and the Microsoft Macro Assembler (MASM64).

## Why This Matters

**The Problem:** Microsoft's x64 compiler does not support inline assembly using `__asm` blocks in 64-bit code. This limitation affects kernel driver development when low-level operations require direct assembly instructions.

**The Solution:** Use separate assembly files (`.asm`) with external function declarations, allowing C code to call assembly routines while maintaining clean separation of concerns.

## Project Structure

```
x64-Driver-Inline-Assembly/
├── MyKernelModerDriver/
│   ├── Driver.c              # Main driver entry point (C code)
│   ├── Driver.asm            # Assembly implementation example 1
│   ├── Source.asm            # Assembly implementation example 2
│   ├── MyKernelModerDriver.inf  # Driver installation file
│   └── MyKernelModerDriver.vcxproj  # Visual Studio project
├── MyKernelModerDriver.sln   # Visual Studio solution
└── README.md
```

## How It Works

### 1. C Code (`Driver.c`)

The driver's C code declares external assembly functions and calls them:

```c
extern void inline MainAsm(void);

NTSTATUS Gs1DriverEntry(_In_ PDRIVER_OBJECT DriverObject,
                        _In_ PUNICODE_STRING RegistryPath)
{
    NTSTATUS status = STATUS_SUCCESS;
    WDF_DRIVER_CONFIG config;
    
    WDF_DRIVER_CONFIG_INIT(&config, WDF_NO_EVENT_CALLBACK);
    config.EvtDriverUnload = Unload;
    
    // Call assembly function
    MainAsm();
    
    return status;
}
```

### 2. Assembly Code (`Driver.asm` / `Source.asm`)

Assembly files contain the actual implementation:

**Driver.asm** - Simple arithmetic operations:
```asm
PUBLIC MainAsm
.code _text

MainAsm PROC PUBLIC
    mov eax, 10000h      ; EAX = 0x10000
    add eax, 40000h      ; EAX = 0x50000
    sub eax, 20000h      ; EAX = 0x30000
    ret
MainAsm ENDP

END
```

**Source.asm** - Register preservation and debugging:
```asm
PUBLIC MainAsm
PUBLIC MainAsm2
.code _text

MainAsm PROC PUBLIC
    push rax             ; Save register
    ; Custom assembly code here
    pop rax              ; Restore register
    ret
MainAsm ENDP

MainAsm2 PROC PUBLIC
    int 3                ; Breakpoint for debugging
    ret
MainAsm2 ENDP

END
```

## Key Techniques Demonstrated

1. **External Function Declaration** - Using `extern` keyword in C to reference assembly functions
2. **MASM64 Syntax** - Proper x64 assembly syntax for kernel-mode code
3. **Function Prologue/Epilogue** - Register preservation (`push`/`pop`)
4. **Public Symbol Export** - Making assembly functions visible to C code
5. **Debugging Support** - Using `int 3` for breakpoints

## Building the Project

### Prerequisites

- Visual Studio 2019 or later
- Windows Driver Kit (WDK) 10
- Windows SDK

### Build Steps

1. Open `MyKernelModerDriver.sln` in Visual Studio
2. Select configuration (Debug/Release) and platform (x64)
3. Build the solution (Ctrl+Shift+B)
4. The compiled driver will be in `x64\[Debug|Release]\MyKernelModerDriver.sys`

### Visual Studio Configuration

The `.asm` files must be configured with:
- **Item Type:** Microsoft Macro Assembler
- **Build Action:** Assemble (MASM)

This is typically configured automatically by the WDK project template.

## Loading the Driver

### Using OSR Driver Loader

1. Run OSR Driver Loader as Administrator
2. Browse and select `MyKernelModerDriver.sys`
3. Click "Register Service"
4. Click "Start Service"

### Command Line (sc.exe)

```cmd
sc create MyKernelModerDriver type= kernel binPath= C:\path\to\MyKernelModerDriver.sys
sc start MyKernelModerDriver
```

**Note:** Driver signature enforcement must be disabled for testing unsigned drivers.

## Use Cases

This technique is essential for:

- **Hypervisor development** - Executing privileged instructions (VMXON, VMREAD, etc.)
- **Performance-critical code** - Optimized assembly routines
- **Hardware interaction** - Direct register manipulation (CR0, CR4, MSRs)
- **Instruction set features** - Using CPUID, RDTSC, or SIMD instructions
- **Security research** - Low-level system inspection and modification

## Important Notes

1. **x64 Calling Convention:** Use the Microsoft x64 calling convention (RCX, RDX, R8, R9 for first 4 parameters)
2. **Register Volatility:** RAX, RCX, RDX, R8-R11 are volatile; RBX, RBP, RDI, RSI, RSP, R12-R15 are non-volatile
3. **Stack Alignment:** Ensure 16-byte stack alignment for function calls
4. **File Naming:** Use different names for `.c` and `.asm` files to avoid linker errors

## Related Documentation

For more information on using this technique in hypervisor development, see:
- `/docs/getting_started.md` - Development environment setup
- `/docs/vmx_operations.md` - Using assembly for VMX operations
- `/docs/RESOURCES.md` - Additional learning resources

## License

This is a sample/educational project demonstrating WDK techniques.
