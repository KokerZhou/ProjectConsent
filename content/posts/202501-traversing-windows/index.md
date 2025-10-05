---
title: "Traversing Windows Loader Structures: Enumerating DLLs & Console Processes"
tags: ["WindowsInternals", "ReverseEngineering", "CProgramming"]
date: 2025-01-16
draft: false
---

Sometimes, the most powerful tools are the ones you build yourself.

Today, I’m sharing a **minimal, self-contained C program** that demonstrates how to walk the **Windows NT Loader’s in-memory module lists** and extract loaded DLLs — **without using `EnumProcessModules` or `CreateToolhelp32Snapshot`**.

We’ll also peek into the **console subsystem** to list all processes attached to the current console using `GetConsoleProcessList`.

> **Use case**: Debugging, reverse engineering, anti-cheat systems, or just understanding how Windows *really* loads your process.

---

## The Code (Copy-Paste Ready)

```c
#include <stdio.h>
#include <windows.h>
#include <winternl.h>

VOID PrintDllNameFromInLoadOrderModuleList()
{
    int i = 0;
    LIST_ENTRY *ListEntryHead = &NtCurrentTeb()->ProcessEnvironmentBlock->Ldr->InMemoryOrderModuleList - 1;
    for (LIST_ENTRY *ListEntry = ListEntryHead->Flink; ListEntry != ListEntryHead; ListEntry = ListEntry->Flink)
    {
        LDR_DATA_TABLE_ENTRY *LdrDataTableEntry = CONTAINING_RECORD(ListEntry, LDR_DATA_TABLE_ENTRY, Reserved1);
        UNICODE_STRING *DllName = &LdrDataTableEntry->FullDllName;
        printf("InLoadOrderModuleList[%d] %wZ\n", i++, &DllName[0]);
    }
}

VOID PrintDllNameFromInMemoryOrderModuleList()
{
    int i = 0;
    LIST_ENTRY *ListEntryHead = &NtCurrentTeb()->ProcessEnvironmentBlock->Ldr->InMemoryOrderModuleList;
    for (LIST_ENTRY *ListEntry = ListEntryHead->Flink; ListEntry != ListEntryHead; ListEntry = ListEntry->Flink)
    {
        LDR_DATA_TABLE_ENTRY *LdrDataTableEntry =
            CONTAINING_RECORD(ListEntry, LDR_DATA_TABLE_ENTRY, InMemoryOrderLinks);
        UNICODE_STRING *DllName = &LdrDataTableEntry->FullDllName;
        printf("InMemoryOrderModuleList[%d] %wZ\n", i++, &DllName[0]);
    }
}

VOID PrintDllNameFromInInitializationOrderModuleList()
{
    int i = 0;
    LIST_ENTRY *ListEntryHead = &NtCurrentTeb()->ProcessEnvironmentBlock->Ldr->InMemoryOrderModuleList + 1;
    for (LIST_ENTRY *ListEntry = ListEntryHead->Flink; ListEntry != ListEntryHead; ListEntry = ListEntry->Flink)
    {
        LDR_DATA_TABLE_ENTRY *LdrDataTableEntry = CONTAINING_RECORD(ListEntry, LDR_DATA_TABLE_ENTRY, Reserved2);
        UNICODE_STRING *DllName = &LdrDataTableEntry->FullDllName;
        printf("InInitializationOrderModuleList[%d] %wZ\n", i++, &DllName[0]);
    }
}

VOID PrintFullProcessImageNameFromConsoleProcessList()
{
    DWORD dwProcessList[260]{};
    DWORD dwProcessCount;

    dwProcessCount = GetConsoleProcessList(dwProcessList, ARRAYSIZE(dwProcessList));

    if (!dwProcessCount || dwProcessCount > ARRAYSIZE(dwProcessList))
        return;

    for (DWORD i = 0; i != dwProcessCount; i++)
    {
        HANDLE hProcess;
        CHAR ExeName[MAX_PATH]{};
        DWORD dwSize = ARRAYSIZE(ExeName);

        hProcess = OpenProcess(PROCESS_QUERY_INFORMATION, FALSE, dwProcessList[i]);

        if (!hProcess)
            continue;

        if (!QueryFullProcessImageNameA(hProcess, 0, ExeName, &dwSize))
            continue;

        printf("ConsoleProcessList[%d] %10d %.*s\n", i, dwProcessList[i], dwSize, ExeName);
    }
}

int main()
{
    PrintDllNameFromInLoadOrderModuleList();
    PrintDllNameFromInMemoryOrderModuleList();
    PrintDllNameFromInInitializationOrderModuleList();
    PrintFullProcessImageNameFromConsoleProcessList();
    return 0;
}
```

---

## What You’ll See

```
InLoadOrderModuleList[0] C:\path\to\something.exe
InLoadOrderModuleList[0] C:\path\to\something.dll
...
ConsoleProcessList[0]       1234 C:\Windows\System32\cmd.exe
ConsoleProcessList[1]       5678 C:\path\to\something.exe
```

---

## Why These Three Lists?

| List                              | Purpose                           | Offset Field         |
|-----------------------------------|-----------------------------------|----------------------|
| `InLoadOrderModuleList`           | Modules in load order (exe first) | `Reserved1`          |
| `InMemoryOrderModuleList`         | Modules in memory layout order    | `InMemoryOrderLinks` |
| `InInitializationOrderModuleList` | Order of `DllMain` calls          | `Reserved2`          |

> **Note**: These structures are **undocumented** but stable across Windows 10/11.

---

## Bonus: `GetConsoleProcessList`

- Returns **all PIDs** attached to the current console
- Great for **parent/child process detection** in CLI tools

---

## References

- [NT Internals](https://www.geoffchappell.com/studies/windows/km/ntoskrnl/inc/api/ntldr/ldr_data_table_entry/index.htm)
- [ReactOS Source](https://doxygen.reactos.org/)
- [GetConsoleProcessList - MSDN](https://learn.microsoft.com/en-us/windows/console/getconsoleprocesslist)

---

## Final Thoughts

This snippet is a **perfect drop-in** for:

- Malware analysis sandboxes
- Game cheat engines
- Custom debuggers
- Learning Windows internals

No dependencies. No bloat. Just raw NT power.

---

*Happy hacking*