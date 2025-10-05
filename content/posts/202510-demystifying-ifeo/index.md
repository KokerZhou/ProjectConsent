---
title: "Demystifying IFEO: Launching Processes Without Infinite Debugger Loops"
tags: ["WindowsInternals", "RedTeaming"]
date: 2025-10-05
draft: false
---

Image File Execution Options (IFEO) is a powerful Windows registry feature for debugging, but it often leads to headaches—like infinite loops when your process tries to launch an IFEO-configured executable. You've probably seen advice to use `DEBUG_PROCESS` flags and detach immediately, but that requires messy debug code.

Today, I'm revealing a cleaner secret: Hook `NtCreateUserProcess` with Detours to set the `IFEOSkipDebugger` flag (0x00000004) in the process creation info. This skips the IFEO debugger entirely for that launch.

Bonus: We'll also cover getting/setting parent process IDs—handy for advanced scenarios like spoofing parents to evade detection.

---

## Setting Up IFEO via Registry

To enable IFEO for a specific executable (e.g., `something.exe`), run this as admin:

```bat
REG ADD "HKLM\Software\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\something.exe" /v Debugger /t REG_SZ /d "X:\path\to\IFEOSkipDebugger.exe" /f
```

This sets a debugger to launch instead of `something.exe`. For more granularity, use subkeys under the path for named processes.

But how do you launch `something.exe` from your app without triggering an infinite loop? Enter the hook.

---

## The Secret: Hooking NtCreateUserProcess with Detours

By intercepting `NtCreateUserProcess` and tweaking the `InitFlags` in `PsCreateInitialState`, we force `IFEOSkipDebugger`.

```cpp
#include "include/detours.h"

#ifdef _WIN64
#pragma comment(lib, "lib.X64/detours.lib")
#else
#pragma comment(lib, "lib.X86/detours.lib")
#endif

typedef NTSTATUS(NTAPI *PNTCREATEUSERPROCESS)(PHANDLE ProcessHandle, PHANDLE ThreadHandle,                       //
                                              ACCESS_MASK ProcessDesiredAccess, ACCESS_MASK ThreadDesiredAccess, //
                                              PVOID ProcessObjectAttributes, PVOID ThreadObjectAttributes,       //
                                              DWORD ProcessFlags, DWORD ThreadFlags,                             //
                                              PVOID ProcessParameters, PVOID CreateInfo, PVOID AttributeList);
PNTCREATEUSERPROCESS p_NtCreateUserProcess;
NTSTATUS NTAPI NewNtCreateUserProcess(PHANDLE ProcessHandle, PHANDLE ThreadHandle,                       //
                                      ACCESS_MASK ProcessDesiredAccess, ACCESS_MASK ThreadDesiredAccess, //
                                      PVOID ProcessObjectAttributes, PVOID ThreadObjectAttributes,       //
                                      DWORD ProcessFlags, DWORD ThreadFlags,                             //
                                      PVOID ProcessParameters, PVOID CreateInfo, PVOID AttributeList)
{
    auto p = (PBYTE)CreateInfo;
#ifdef _WIN64
    if (*(ULONG *)&p[8] == 0) // PsCreateInitialState
    {
        auto &InitFlags = *(ULONG *)&p[16];
        InitFlags |= 0x00000004; // IFEOSkipDebugger
    }
#else
    if (*(ULONG *)&p[4] == 0) // PsCreateInitialState
    {
        auto &InitFlags = *(ULONG *)&p[8];
        InitFlags |= 0x00000004; // IFEOSkipDebugger
    }
#endif
    return p_NtCreateUserProcess(ProcessHandle, ThreadHandle,                     //
                                 ProcessDesiredAccess, ThreadDesiredAccess,       //
                                 ProcessObjectAttributes, ThreadObjectAttributes, //
                                 ProcessFlags, ThreadFlags,                       //
                                 ProcessParameters, CreateInfo, AttributeList);
}
```

Wrap your `CreateProcessW` call with Detours attach/detach:

```cpp
    DetourTransactionBegin();
    DetourAttach(&(PVOID &)p_NtCreateUserProcess, NewNtCreateUserProcess);
    DetourTransactionCommit();
    BOOL result = CreateProcessW(/* ... */);
    DetourTransactionBegin();
    DetourDetach(&(PVOID &)p_NtCreateUserProcess, NewNtCreateUserProcess);
    DetourTransactionCommit();
```

### Why This Works

- `NtCreateUserProcess` is the low-level API for user-mode process creation.
- We modify `CreateInfo` (a `PS_CREATE_INFO` structure) to set `IFEOSkipDebugger` in `InitFlags`.
- This flag tells the kernel to ignore IFEO's `Debugger` value for this spawn.
- Bonus perk: Bypasses Windows 10/11 app aliases (e.g., `python.exe`) that might trigger unwanted redirects.

No debug privileges needed. No infinite loops. Clean and efficient.

---

## Bonus: Get/Set Process Parent ID

### Getting the Parent Process ID

Use `NtQueryInformationProcess` for the current process's parent PID:

```cpp
DWORD GetParentProcessId()
{
    PROCESS_BASIC_INFORMATION pbi{};
    ULONG retLen = 0;
    if (NT_SUCCESS(NtQueryInformationProcess(GetCurrentProcess(), ProcessBasicInformation, &pbi, sizeof(pbi), &retLen)))
    {
        auto InheritedFromUniqueProcessId = (DWORD)(ULONG_PTR)pbi.Reserved3;
        return InheritedFromUniqueProcessId;
    }
    return 0;
}
```

### Setting (Spoofing) the Parent Process

To spoof the parent when creating a child process:

```cpp
template <typename T, typename Deleter> auto make_unique_with_deleter(T *ptr, Deleter deleter)
{
    return std::unique_ptr<T, Deleter>(ptr, deleter);
}

    HANDLE hParentProcess = OpenProcess(PROCESS_CREATE_PROCESS, FALSE, dwParentProcessId);
    if (!hParentProcess)
    {
        //
    }
    auto hParentProcessUP = make_unique_with_deleter(&hParentProcess, [](auto *ptr) { CloseHandle(*ptr); });

    SIZE_T attrListSize = 0;
    InitializeProcThreadAttributeList(NULL, 1, 0, &attrListSize);
    auto attrListBufUP = std::make_unique<BYTE[]>(attrListSize);
    auto attrList = (LPPROC_THREAD_ATTRIBUTE_LIST)attrListBufUP.get();
    if (!InitializeProcThreadAttributeList(attrList, 1, 0, &attrListSize))
    {
        //
    }
    auto attrListUP = make_unique_with_deleter(&attrList, [](auto *ptr) { DeleteProcThreadAttributeList(*ptr); });

    if (!UpdateProcThreadAttribute(attrList, 0, PROC_THREAD_ATTRIBUTE_PARENT_PROCESS, &hParentProcess,
                                   sizeof(hParentProcess), NULL, NULL))
    {
        //
    }
```

Pass this attribute list to `CreateProcess` via `STARTUPINFOEX.lpAttributeList`.

This is often used in red teaming to make child processes appear spawned by benign parents (e.g., `explorer.exe`), evading EDR tools.

---

## Why This Matters

IFEO is great for global hooks/debuggers, but self-launching breaks things. This technique keeps your tools flexible without complexity.

For parent spoofing: Essential for stealth in security research or custom launchers.

---

## References

- Geoff Chappell: [PS_CREATE_INFO](https://www.geoffchappell.com/studies/windows/km/ntoskrnl/inc/api/ntpsapi/ps_create_info/)
- Detours: [Microsoft Research](https://www.microsoft.com/en-us/research/project/detours/)

---

*Happy low-level hacking*