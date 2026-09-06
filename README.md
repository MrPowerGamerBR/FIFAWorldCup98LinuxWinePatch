# FIFAWorldCup98LinuxWinePatch

<img width="320" height="240" alt="20260905_181545_Wine Desktop" src="https://github.com/user-attachments/assets/eaf13780-34e9-46e5-b4b1-8cb687e04500" />
<img width="320" height="240" alt="20260905_181725_Wine Desktop" src="https://github.com/user-attachments/assets/b1500074-3251-4ed7-b8d0-4531167c070d" />
<img width="320" height="240" alt="20260905_181744_Wine Desktop" src="https://github.com/user-attachments/assets/bdeb5430-e767-4ad0-87de-5726dc350cc2" />
<img width="320" height="240" alt="20260905_181757_Wine Desktop" src="https://github.com/user-attachments/assets/674f9736-1639-46e3-a2c2-2f3b1bc5e129" />

This is a executable patch for FIFA World Cup 98 and FIFA Road to World Cup 98 to make it work under Linux + Wine.

You can download the patched executable from the [Releases](https://github.com/MrPowerGamerBR/FIFAWorldCup98LinuxWinePatch/releases) tab.

## Issues

* Cutscenes SOMETIMES do not work (they just get skipped instead of playing) and, for FIFA Road to World Cup 98, cutscenes don't work altogether.
* Sadly it still doesn't work on Windows 11.

## How it Works

Some months ago I knew that it was probably a race condition because setting the process to ONLY use a single core and to enable `WINEDEBUG=+thread` made the game lock up... less. And if there's a *making things slow makes it work* smoke, then there's a *race condition* fire somewhere.

Another thing that pointed to a race condition was that Wine complained about wait timeouts when the game locked up.

```
06b4:err:sync:RtlpWaitForCriticalSection section 7BD00300 "../wine/dlls/ntdll/loader.c: loader_section" wait timed out in thread 06b4, blocked by 06ac, retrying (60 sec)
```

So, just for funsies, I tried using Claude Fable to help me debug the issue. While Claude helped me figure out how to fix the issue, **the fix on the release page and the write up is entirely written by myself**, because I wanted to *learn* how to fix the issue. The FIFA Road to World Cup 98 patch was also written entirely by myself without any help.

How can we figure out what that thread is doing? Thankfully Wine has some [very comprehensive debugging channels](https://gitlab.winehq.org/wine/wine/-/wikis/Debug-Channels), so if we run it with `WINEDEBUG=+relay,+timestamp`, we can know what's going on behind the scenes

```
WINEDEBUG=+relay,+timestamp WINEPREFIX="/home/mrpowergamerbr/WinePrefixes/FIFA98/" wine fifawc.exe >>file.txt 2>&1`
```

First we find the lock up again, because the thread PIDs change every run...

```
21099.663:0688:err:sync:RtlpWaitForCriticalSection section 7BD00300 "../wine/dlls/ntdll/loader.c: loader_section" wait timed out in thread 0688, blocked by 062c, retrying (300 sec)
```

So this time it is `062c`! So, what is `062c` actually doing?

```
21094.617:062c:Call ntdll.NtClose(00000138) ret=7b50ebe7
21094.617:062c:Call ntdll.NtClose(00000138) ret=6ffffbb81526
21094.617:062c:Ret  ntdll.NtClose() retval=00000000 ret=6ffffbb81526
21094.617:062c:Ret  ntdll.NtClose() retval=00000000 ret=7b50ebe7
21094.617:062c:Ret  KERNEL32.CloseHandle() retval=00000001 ret=004c388a
21094.617:062c:Call KERNEL32.SetEvent(00000130) ret=004c37b9
21094.617:062c:Call ntdll.NtSetEvent(00000130,00000000) ret=7b55f13c
21094.617:062c:Call ntdll.NtSetEvent(00000130,00000000) ret=6ffffbb81526
21094.617:062c:Ret  ntdll.NtSetEvent() retval=00000000 ret=6ffffbb81526
21094.617:062c:Ret  ntdll.NtSetEvent() retval=00000000 ret=7b55f13c
21094.617:062c:Ret  KERNEL32.SetEvent() retval=00000001 ret=004c37b9
21094.617:062c:Call ntdll.NtQueryInformationThread(fffffffffffffffe,0000000c,035cff04,00000004,00000000) ret=6ffffbb930af
21094.617:062c:Ret  ntdll.NtQueryInformationThread() retval=00000000 ret=6ffffbb930af
21094.617:062c:Call PE DLL (proc=776F22C0,module=776F0000 L"ws2_32.dll",reason=THREAD_DETACH,res=00000000)
21094.617:062c:Ret  PE DLL (proc=776F22C0,module=776F0000 L"ws2_32.dll",reason=THREAD_DETACH,res=00000000) retval=1
21094.617:062c:Call PE DLL (proc=77934020,module=77930000 L"opengl32.dll",reason=THREAD_DETACH,res=00000000)
21094.617:062c:Call ucrtbase.free(00000000) ret=77933f17
21094.617:062c:Call KERNEL32.HeapFree(00110000,00000000,00000000) ret=7aae9916
21094.617:062c:Ret  KERNEL32.HeapFree() retval=00000001 ret=7aae9916
21094.617:062c:Ret  ucrtbase.free() retval=00000001 ret=77933f17
21094.617:062c:Ret  PE DLL (proc=77934020,module=77930000 L"opengl32.dll",reason=THREAD_DETACH,res=00000000) retval=1
21094.617:062c:Call PE DLL (proc=79F89E90,module=79F70000 L"ole32.dll",reason=THREAD_DETACH,res=00000000)
21094.617:062c:Ret  PE DLL (proc=79F89E90,module=79F70000 L"ole32.dll",reason=THREAD_DETACH,res=00000000) retval=1
21094.617:062c:Call PE DLL (proc=79DFE9C0,module=79DF0000 L"combase.dll",reason=THREAD_DETACH,res=00000000)
21094.617:062c:Ret  PE DLL (proc=79DFE9C0,module=79DF0000 L"combase.dll",reason=THREAD_DETACH,res=00000000) retval=1
21099.663:0688:err:sync:RtlpWaitForCriticalSection section 7BD00300 "../wine/dlls/ntdll/loader.c: loader_section" wait timed out in thread 0688, blocked by 062c, retrying (300 sec)
```

So the thread got stuck while it was being terminated, what gives?

For reference, when a thread exits cleanly, it should look like this

```
05e0:Ret  PE DLL (proc=77672E70,module=77670000 L"imm32.dll",reason=THREAD_DETACH,res=00000000) retval=1
05e0:Call PE DLL (proc=7A55F710,module=7A540000 L"user32.dll",reason=THREAD_DETACH,res=00000000)
05e0:Call win32u.NtUserCallNoParam(00000007) ret=6ffffb86cb0c
05e0:Ret  win32u.NtUserCallNoParam() retval=00000000 ret=6ffffb86cb0c
05e0:Call win32u.NtUserCallNoParam(00000008) ret=6ffffb86cb0c
05e0:Ret  win32u.NtUserCallNoParam() retval=00000000 ret=6ffffb86cb0c
05e0:Ret  PE DLL (proc=7A55F710,module=7A540000 L"user32.dll",reason=THREAD_DETACH,res=00000000) retval=1
05e0:Call PE DLL (proc=7AAA7080,module=7AA90000 L"ucrtbase.dll",reason=THREAD_DETACH,res=00000000)
05e0:Call KERNEL32.HeapFree(00110000,00000000,00000000) ret=7aaa6cda
05e0:Ret  KERNEL32.HeapFree() retval=00000001 ret=7aaa6cda
05e0:Ret  PE DLL (proc=7AAA7080,module=7AA90000 L"ucrtbase.dll",reason=THREAD_DETACH,res=00000000) retval=1
05e0:Call PE DLL (proc=7AEAAB60,module=7AEA0000 L"msvcrt.dll",reason=THREAD_DETACH,res=00000000)
05e0:Call KERNEL32.HeapFree(00110000,00000000,00000000) ret=7aeaa7aa
05e0:Ret  KERNEL32.HeapFree() retval=00000001 ret=7aeaa7aa
05e0:Ret  PE DLL (proc=7AEAAB60,module=7AEA0000 L"msvcrt.dll",reason=THREAD_DETACH,res=00000000) retval=1
05e0:Call ntdll.NtTerminateThread(fffffffffffffffe,00000000) ret=6ffffbb81526
```

If we look around that, do we have anything suspicious?

```
21094.617:0608:Call KERNEL32.TerminateThread(00000134,00000000) ret=004c3478
21094.617:062c:Call KERNEL32.HeapFree(00110000,00000000,00000000) ret=7aae9916
21094.617:0608:Call ntdll.NtTerminateThread(00000134,00000000) ret=7b5678ab
21094.617:062c:Ret  KERNEL32.HeapFree() retval=00000001 ret=7aae9916
21094.617:0608:Call ntdll.NtTerminateThread(00000134,00000000) ret=6ffffbb81526
21094.617:062c:Ret  ucrtbase.free() retval=00000001 ret=77933f17
21094.617:062c:Ret  PE DLL (proc=77934020,module=77930000 L"opengl32.dll",reason=THREAD_DETACH,res=00000000) retval=1
21094.617:062c:Call PE DLL (proc=79F89E90,module=79F70000 L"ole32.dll",reason=THREAD_DETACH,res=00000000)
21094.617:062c:Ret  PE DLL (proc=79F89E90,module=79F70000 L"ole32.dll",reason=THREAD_DETACH,res=00000000) retval=1
21094.617:062c:Call PE DLL (proc=79DFE9C0,module=79DF0000 L"combase.dll",reason=THREAD_DETACH,res=00000000)
21094.617:062c:Ret  PE DLL (proc=79DFE9C0,module=79DF0000 L"combase.dll",reason=THREAD_DETACH,res=00000000) retval=1
```

Oh... why is that thread (`0608`) terminating another thread suspiciously near where the `062c` thread stops working... We could assume that it is killing our thread, but let's figure it out if it is really `0608` the one that's assassinating the thread at COLD BLOOD.

To track it down, we need to figure out how to convert the `ThreadHandle` (the first parameter of `NtTerminateThread`) to a thread ID ([docs](http://undocumented.ntinternals.net/index.html?page=UserMode%2FUndocumented%20Functions%2FNT%20Objects%2FThread%2FNtTerminateThread.html)), so we need to scavenge the logs to figure out WHERE the thread was created.

By searching for `CreateThread.*00000134`, we get the following result.

```
21094.108:0608:Ret  ntdll.NtCreateThreadEx() retval=00000000 ret=7b515098
21094.108:0608:Ret  KERNEL32.CreateThread() retval=00000134 ret=004c33ac
21094.108:0608:Call KERNEL32.SetThreadPriority(00000134,00000001) ret=004c3646
21094.108:0608:Call ntdll.NtSetInformationThread(00000134,00000003,0022fc24,00000004) ret=7b5616d1
21094.108:0608:Call ntdll.NtSetInformationThread(00000134,00000003,0022fc24,00000004) ret=6ffffbb81526
21094.108:0608:Ret  ntdll.NtSetInformationThread() retval=00000000 ret=6ffffbb81526
21094.108:0608:Ret  ntdll.NtSetInformationThread() retval=00000000 ret=7b5616d1
21094.108:0608:Ret  KERNEL32.SetThreadPriority() retval=00000001 ret=004c3646
21094.108:0608:Call KERNEL32.ResumeThread(00000134) ret=004c33da
21094.108:0608:Call ntdll.NtResumeThread(00000134,0022fc34) ret=7b5584eb
21094.108:0608:Call ntdll.NtResumeThread(00000134,0022fc34) ret=6ffffbb81526
21094.108:0608:Ret  ntdll.NtResumeThread() retval=00000000 ret=6ffffbb81526
21094.108:0608:Ret  ntdll.NtResumeThread() retval=00000000 ret=7b5584eb
21094.108:0608:Ret  KERNEL32.ResumeThread() retval=00000001 ret=004c33da
21094.108:062c:Call wow64.Wow64LdrpInitialize(1015bf820) ret=6fffffc03ad0
```

The `retval` of the `CreateThread` is the `ThreadHandle` of the thread `062c`. We can't be 100% sure that it is for that specific thread, because nothing binds the thread creation to the thread ID, but judging the timestamps between when the thread was created and when the `062c` thread started executing code, we can be fairly certain that this is it.

You may be wondering "why does the game call `ResumeThread` if the thread was created right now???", it is because...

```
21094.108:0608:Call KERNEL32.CreateThread(00000000,00000000,004c3300,0042bdfc,00000004,0022fc74) ret=004c33ac
```

The [fifth parameter](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createthread) is `CREATE_SUSPENDED`. So the game creates the thread in suspended mode, configures the thread, then resumes it.

Okay, so now we know (somewhat) what is causing the issue: Thread `0608` tries to `TerminateThread` the thread `062c` (`ThreadHandle` = `00000134`) and, because `062c` is in the middle of a thread shutdown, it causes locks to not be released.

In fact, in the Windows' docs [TerminateThread is considered harmful](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-terminatethread) for that *exactly* reason, that if you aren't 100% sure what is the other thread doing at the time of the termination, you may have unforeseen consequences.

Is there's any way we can actually *see* the code? Thankfully, yes!

```
21094.617:0608:Call KERNEL32.TerminateThread(00000134,00000000) ret=004c3478
```

The `ret` address is where the `TerminateThread` should jump back to after the thread is terminated. When jumping to that address in Ghidra, we get to this function

```c

void __fastcall FUN_004c3460(undefined4 param_1,undefined4 param_2)

{
  undefined4 *in_EAX;
  HANDLE unaff_EBX;
  
  if (in_EAX == (undefined4 *)0x0) {
    DAT_00585c9c = s_win\threads.c_005692b0;
    DAT_00585ca0 = 0x122;
    FUN_004c2bb0(s_removethread_-_CANNOT_REMOVE_THE_00569340,param_2,param_1);
  }
  else if (in_EAX == (undefined4 *)0xffffffff) {
    unaff_EBX = GetCurrentThread();
  }
  else {
    unaff_EBX = (HANDLE)*in_EAX;
  }
  TerminateThread(unaff_EBX,0);
  CloseHandle(unaff_EBX);
  return;
}
```

If the thread is already shutting down anyway, what would happen if we just... no-op'd the `TerminateThread` function?

To do that, we know that in Ghidra the hex values for that `TerminateThread` function are

```
004c3471 2e ff 15 9c 85 55 00       CALL       dword ptr CS:[->KERNEL32.DLL::TerminateThread]   = 0015902a
```

We also need to no-op the "call constructor" too because, if we don't, the game will crash (oops).

```
004c346e 6a 00           PUSH       0x0
004c3470 53              PUSH       EBX
004c3471 2e ff 15 9c 85 55 00       CALL       dword ptr CS:[->KERNEL32.DLL::TerminateThread]   = 0015902a
```

If you are curious, the `PUSH` calls are essentially both the parameters passed to the `CALL` in reverse order. `push 0` -> `push ebx` -> `TerminateThread(unaff_EBX,0);`. I don't know enough about x86 assembly, but I suppose that `PUSH` is essentially push values to the stack, just like how a stack-based VM works (like the Java Virtual Machine™, or GameMaker's VM).

In a hex editor (I used ImHex), replace the `6a 00 53 2e ff 15 9c 85 55 00` sequence with `90 90 90 90 90 90 90 90 90 90`, then save the edited executable.

And that's all there's to it! :)

You may wonder "Why not use [`WaitForSingleObject`](https://learn.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-waitforsingleobject)? It is the proper solution if you want to wait a thread to terminate!" and the answer is that I didn't notice any meaningful difference between no-op'ing the function call out and using `WaitForSingleObject`. It is also possible to do however, you can just track the function in Ghidra, check the PTR address of the function, then replace the function address on the opcode with the pointer. The `CALL` opcode uses little endian for the pointers, so you need to swap the bytes of the pointer.

FIFA Road to World Cup 98 has the same lock up issue, I tried applying the same fix to it and, while it works *sometimes*, some other times it crashes with "SHOWDCT: No DCT chunks found!".

A very hacky solution for this is to just... not call the function that handles the DCT chunks.

To find it, I searched on Ghidra for `SHOWDCT`, then I found where it is called from. Thankfully there is only one caller.

Then, to no-op it, just replace `e8 49 fc ff ff` with `90 90 90 90 90`.

This does have the caveat that the game won't play cutscenes anymore, but it does fix the crash!

Probably there are better way to fix both games, considering that the issue seems to both be related to cutscenes, but for now, this shall do.