# FIFAWorldCup98LinuxWinePatch

This is a executable patch 

You can download the patched executable from the [Releases] tab.

## Issues

* Cutscenes sometimes do not work
* Sadly it still doesn't work on Windows 11

## How it Works

Some months ago I knew that it was probably a race condition because setting the process to ONLY use a single core and to enable `WINEDEBUG=+thread` "fixed" the issue. And if there's a making things slow smoke, then there's a race condition fire somewhere. That fix however is not *that* good, where the game still locked up randomly when moving through menus.

Another thing that pointed to a race condition was that Wine complained about wait timeouts when the game locked up.

```
06b4:err:sync:RtlpWaitForCriticalSection section 7BD00300 "../wine/dlls/ntdll/loader.c: loader_section" wait timed out in thread 06b4, blocked by 06ac, retrying (60 sec)
```

So, just for funsies, I tried using Claude Fable to help me debug the issue. While Claude helped me figure out how to fix the issue, **the fix on the release page and the write up is entirely written by myself**, because I wanted to *learn* how to fix the issue.

How can we figure out what that thread is doing? Thankfully Wine has some [very comprehensive debugging channels](https://gitlab.winehq.org/wine/wine/-/wikis/Debug-Channels), so if we run it with `WINEDEBUG=+relay,+snoop` enabled, we can know what's going on behind the scenes

```
WINEDEBUG=+relay,+snoop WINEPREFIX="/home/mrpowergamerbr/WinePrefixes/FIFA98/" wine fifawc.exe >>file.txt 2>&1`
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

Oh... why is that thread (`0608`) terminating another thread suspiciously near where the `062c` thread stops working...  The first parameter is the `ThreadHandle`, the second parameter is the `ExitStatus` ([docs](http://undocumented.ntinternals.net/index.html?page=UserMode%2FUndocumented%20Functions%2FNT%20Objects%2FThread%2FNtTerminateThread.html)).

To track it down, we need to figure out how to convert the `ThreadHandle` to the Thread ID that Wine puts in the log prefix. To do that, we need to scavenge the logs to figure out WHERE the thread was created.

And after a while, we find it here!

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

The `retval` of the `CreateThread` is the `ThreadHandle` of the thread `062c`. We know it is for that specific thread by judging the timestamps between when the thread was created and when the `062c` thread started executing code.

You may be wondering "why does the game call `ResumeThread` if the thread was created right now???", it is because...

```
21094.108:0608:Call KERNEL32.CreateThread(00000000,00000000,004c3300,0042bdfc,00000004,0022fc74) ret=004c33ac
```

The [fifth parameter](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createthread) is `CREATE_SUSPENDED`. So the game creates the thread in suspended mode, configures the thread, then resumes it.

Okay, so now we know (somewhat) what is causing the issue: Thread `0608` tries to `TerminateThread` thread `062c` (`ThreadHandle` = `00000134`) and, because `062c` is in the middle of a thread shutdown, it causes locks to not be released. In fact, in the Windows' docs [TerminateThread is considered harmful](https://learn.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-terminatethread).

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

What would happen if we just... no-op'd the `TerminateThread` function?

To do that, we know that in Ghidra the hex values for that `TerminateThread` function are

```
004c3471 2e ff 15 9c 85 55 00       CALL       dword ptr CS:[->KERNEL32.DLL::TerminateThread]   = 0015902a
```

We also need to no-op the "call constructor" too

```
004c346e 6a 00           PUSH       0x0
004c3470 53              PUSH       EBX
004c3471 2e ff 15 9c 85 55 00       CALL       dword ptr CS:[->KERNEL32.DLL::TerminateThread]   = 0015902a
```

Sadly, just no-op'ing (replacing everything with `0x90`) doesn't work for us, because doing that crashes the game.

In a hex editor (I used ImHex), replace the `6a 00 53 2e ff 15 9c 85 55 00` sequence with `90 90 90 90 90 90 90 90 90 90`.

And that's all there's to it! :)

The Road to World Cup game has the same lock up issue, but I haven't made a fix for it yet, but with these instructions you probably can get it working too.