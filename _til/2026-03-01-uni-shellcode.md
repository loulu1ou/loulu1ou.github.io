---
layout: series
title: The Universal Shellcode Execution Pattern
order: 1
series_title: Today I Learned
---

Whether you are writing a simple loader or a complex nation-state implant, the CPU always needs four specific things to happen to run code. We call this the "Allocate-Write-Protect-Execute" pattern.

Executing shellcode typically requires four fundamental steps:
1. Allocate Memory
2. Copy payload to the executable memory
3. Changing memory protection
4. Execute --> create thread

Here is the breakdown of why this pattern exists and the variations you will see in more advanced malware.

---

### 1. The Standard Pattern (Local & Remote)

Here is the breakdown of why this pattern exists and the specific APIs used for Local (self) and Remote (injection) execution.

| Step | Action | Why? | Standard API (Local) | Standard API (Remote) |
| :--- | :--- | :--- | :--- | :--- |
| **1** | **Allocate** | You need "Private Property" in memory to store your code. | `VirtualAlloc` | `VirtualAllocEx` |
| **2** | **Write** | You need to move your payload from your buffer (data) to the new property. | `memcpy` / `RtlMoveMemory` | `WriteProcessMemory` |
| **3** | **Protect** | **(OpSec)** You change permissions from "Write" to "Execute". | `VirtualProtect` | `VirtualProtectEx` |
| **4** | **Execute** | You tell the CPU "Start working here." | `CreateThread` | `CreateRemoteThread` |

## Why Step 3 (Protection) Matters
Technically, you could skip step 3 - Protect if you allocate with RWX (PAGE_EXECUTE_READWRITE) in step 1.

So why do we use the `Allocate RW → Write → Protect RX → Execute` pattern?

The modern EDRs scan memory for `RWX` pages. If the page containing the shellcode is marked `Writable` and `Executable`, it is an immediate red flag because legitimate software almost never uses this configuration. 

By splitting the steps, you adhere to the W^X (Write XOR Execute) principle, blending in better with normal application behavior.

## Getting Creative with Step 4
Step 4 is the most heavily monitored step. APIs like CreateThread and CreateRemoteThread are "noisy"—they leave clear indicators that an analyst can spot.

Advanced malware keeps Steps 1, 2, and 3, but changes Step 4 to something else:
- APC Injection: Instead of creating a new thread, you tell an existing thread: "Hey, do this little job for me when you have time."
    - API: QueueUserAPC
- Callback Functions: You ask Windows to do something (like list windows), and say "Run this function for every window you find." If "this function" is your shellcode, Windows runs it for you.
    - API: EnumWindows, EnumSystemLocales, SetWindowsHookEx.
- Thread Hijacking: You pause a thread, force its CPU pointer (RIP) to jump to your memory (Step 2), and unpause it.
    - API: SetThreadContext, ResumeThread.

The Goal (Execute Shellcode) stays the same, but the Mechanism changes to evade detection.