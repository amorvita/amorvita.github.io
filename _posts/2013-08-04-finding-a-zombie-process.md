---
layout: post
title: Finding a zombie process
date: 2013-08-04 14:00
comments: true
categories: []
---
Generally, 'zombie process' emerged in fork based operating systems freqenetly. 'zombie process' pops up many times in "Advanced Programming in the UNIX(R) Environment", Richard Stevens' masterpiece. However, 'zombie process' is not familiar in Windows. Jeffrey Richter's great book, "Windows via C/C++" doesn't describe 'zombie process' at all.

Then, is there no 'zombie process' in Windows? To tell the conclusion first, it's not. Why 'zombie process' is so unfamiliar in Windows? The main reason is that Windows hides information with the zombie process completely. Have you ever seen a process tagged 'zombie' or a dead process on your task manager? Maybe, you never see them. Have you ever met a dead process when you enumerate processes using Toolhelp or Psapi library? You never see them, too. These things are the same as using Windows NT native API.

We never see any zombie process in Windows, nevertheless how can I claim that 'zombie process' exists in Windows? Because we can see them using unpopular methods. There're two methods seeing the zombie process. One thing is that PID brute-force the way to check whether the process exists or not calling OpenProcess with random PID. The other way is that walking through the EPROCESS kernel structure to enumerate processes directly.

Anyway, let's start to find the evidence of the zombie process. I launched notepad.exe, then make it a zombie process, and see the result. Following windbg dump shows them. The dump tells us what the zombie process is in Windows clearly.

	kd> !process 0 0 notepad.exe  
	PROCESS fffffa8002236b30  
	    SessionId: 1  Cid: 08dc    Peb: 7fffffdf000  ParentCid: 076c  
	    DirBase: 3cc09000  ObjectTable: 00000000  HandleCount:   0.
	    Image: notepad.exe  
	  
	kd> !process fffffa8002236b30  
	PROCESS fffffa8002236b30  
	    SessionId: 1  Cid: 08dc    Peb: 7fffffdf000  ParentCid: 076c  
	    DirBase: 3cc09000  ObjectTable: 00000000  HandleCount:   0.  
	    Image: notepad.exe  
	    VadRoot 0000000000000000 Vads 0 Clone 0 Private 1. Modified 4. Locked 0.  
	    DeviceMap fffff8a0016b0200  
	    Token                             fffff8a001c6a060  
	    ElapsedTime                       00:00:27.341  
	    UserTime                          00:00:00.046  
	    KernelTime                        00:00:00.109  
	    QuotaPoolUsage[PagedPool]         0  
	    QuotaPoolUsage[NonPagedPool]      0  
	    Working Set Sizes (now,min,max)  (5, 50, 345) (20KB, 200KB, 1380KB)  
	    PeakWorkingSetSize                1799  
	    VirtualSize                       55 Mb  
	    PeakVirtualSize                   77 Mb  
	    PageFaultCount                    1850  
	    MemoryPriority                    BACKGROUND  
	    BasePriority                      8  
	    CommitCharge                      0  
	  
	No active threads  

I describe details of a zombie process for who can't understand what the zombie process is precisely. 'zombie process' is the process terminated completely, but the kernel process structure(EPROCESS) of it exists. The process can't be destroyed if it wants to be. This situation looks like a zombie, so we call them zombie. As you can see from the windbg dump, zombie process has no address spaces, no kernel objects, and no threads. It has only kernel process structure and PID resource.

Then, why can't the zombie process be terminated? The reason is simple. Because there's a process which references the terminated process. In this scenario, we can make a zombie process easily using the following code.


	#include "windows.h"  
	#include "tchar.h"  
	  
	int _tmain(int argc, _TCHAR* argv[])  
	{  
	    if(argc < 2)  
		return 0;  
	  
	    HANDLE process = OpenProcess(SYNCHRONIZE, FALSE, _tcstoul(argv[1], NULL, 10));  
	    if(!process)  
	    {  
		printf("open error %d\n", GetLastError());  
		return 0;  
	    }  
	  
	    getchar();  
	  
	    CloseHandle(process);  
	  
	    return 0;  
	}  

If you want to make a zombie process, just follow instructions. After terminating notepad, it becomes a zombie because zombie_maker holds the handle of the terminated notepad. It keeps the status permanently until closing its handle.

1. launching notepad.exe
2. executing zombie_maker.exe <br/>
ex) zombie_maker.exe `[notepad PID]`
3. terminating notepad.exe

Eventually, zombie process springs up when another process catch its handle in many case, like <a href="http://jiniya.net/tt/769">we can't delete a file when the explorer holds the file handle</a>. Then, what kind of program makes a zombie process in Windows frequently? That's a security program like anti-virus which scans process in runtime. Fortunately, these situation occurs in user level, we can solve them killing the process which holds the handle of the zombie process. However, we can't remove a zombie process until rebooting a system if the zombie process comes from a handle leak of a kernel mode driver.
