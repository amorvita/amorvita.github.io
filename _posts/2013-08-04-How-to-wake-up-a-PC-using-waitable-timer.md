---
layout: post
title: How to wake up a PC using waitable timer.
date: 2013-08-04 01:00
comments: true
categories: []
---
I used to love listening to the pop-song radio program that aired early in the morning when I was a teenager. However, waking up in the early morning was almost impossible for me, then I couldn't help wondering how convenient and wonderful it would be if the radio could automatically be turned on and wakes me up. (There was no such thing back then). However, I am guessing that some students nowadays might want the similar function from their PCs, so we are going to cover the method to do so today.

The power status of the Windows are divided into five main categories. Please refer to the link below for further information.

<a href="http://msdn.microsoft.com/en-us/library/windows/desktop/aa373229(v=vs.85).aspx">http://msdn.microsoft.com/en-us/library/windows/desktop/aa373229(v=vs.85).aspx</a>

If we were to categorize the five different power status, we have Operating mode (S0), Sleep mode (S1, S2, S3) and Hibernation (S4), soft off (s5) and Off(G3). The only categories we can wake up the computer via the software is when it's on Sleep mode S1,S2,S3 and S4 mode while hibernation (S4) mode is the most frequently used to turn on the computer automatically as it consumes the lowest amount of power.

Now we will look at the specific way we can turn the computer back on at a certain time. The most common technique for this would be to turn it off to the hibernation and wake it up at a certain time. Turning off the computer via hibernation mode is extremely simple. All it requires is the code as you see below.

	SetSystemPowerState(FALSE, FALSE);  

However, it will not work as well as you expect, as we do not have the authority to turn off the power. Thus, we will need to gain the authority through the code SeShutdownPrivilege. This code allows the authority to turn your computer off via the code "SetSystemPowerState" successfully.


	static BOOL WINAPI  
	_EnableShutDownPriv()  
	{  
	    BOOL ret = FALSE;  
	    HANDLE process = GetCurrentProcess();  
	    HANDLE token;  
	  
	    if(!OpenProcessToken(process  
				, TOKEN_ADJUST_PRIVILEGES | TOKEN_QUERY  
				, &token))  
		return FALSE;  
	  
	    LUID luid;  
	    LookupPrivilegeValue(NULL, "SeShutdownPrivilege", &luid);  
	  
	    TOKEN_PRIVILEGES priv;  
	    priv.PrivilegeCount = 1;  
	    priv.Privileges[0].Attributes = SE_PRIVILEGE_ENABLED;  
	    priv.Privileges[0].Luid = luid;  
	    ret = AdjustTokenPrivileges(token, FALSE, &priv, 0, NULL, NULL);  
	    CloseHandle(token);  
	  
	    return TRUE;  
	}  

Now I am going to cover the method to wake the computer up. In order to get the computer back on from the hibernation mode, we need what we call 'waitable timer', which operates when your PC is in the hibernation mode. Its primary function is as same is the timer, which triggers the event we appointed at the designated time. In other words, the event set in prior is triggered automatically as the code prepared starts its progress. The next code shows us how to use the pre-arranged timer. The only fact you need to be aware in this process is that the unit of time is based on 100 nano seconds. If you appoint the positive number, it becomes an absolute time, while using appointing negative becomes a relative time. The absolute time is recognized same as FILETIME on Windows. Therefore, as we appoint the negative number as per below, the timer will initiate itself after certain seconds from the moment we set the timer.

	HANDLE timer = CreateWaitableTimer(NULL, TRUE, "MyWaitableTimer");  
	if(timer == NULL)  
	    return FALSE;  
	  
	__int64 nanoSecs;  
	LARGE_INTEGER due;  
	  
	nanoSecs = -secs * 1000 * 1000 * 10;  
	due.LowPart = (DWORD) (nanoSecs & 0xFFFFFFFF);  
	due.HighPart = (LONG) (nanoSecs >> 32);  
	  
	if(!SetWaitableTimer(timer, &due, 0, 0, 0, TRUE))  
	    return FALSE;  

The last thing you need to remember is that you need to notify the Windows that certain power is required if you need to turn on the monitor separately once the computer is turned on from the Hibernation mode. All you need is the code as per below, which will trigger the notification and automatically turn on the monitor.

	SetThreadExecutionState(ES_DISPLAY_REQUIRED);  

Overall, first you gain the authority, then set the waitable timer and initiate hibernation. When the waitable timer starts its procedure, it notifies the system that we want the power on. This is all done as per the code below.

	DWORD CALLBACK  
	ShutDownProc(LPVOID p)  
	{  
	    PHANDLE timer = (PHANDLE) p;  
	  
	    WaitForSingleObject(*timer, INFINITE);  
	    CloseHandle(*timer);  
	    SetThreadExecutionState(ES_DISPLAY_REQUIRED);   
	  
	    return 0;  
	}  
  
	PWRMGMT_API BOOL WINAPI  
	HibernateAndReboot(int secs)  
	{  
	    if(!_EnableShutDownPriv())  
		return FALSE;  
	  
	    HANDLE timer = CreateWaitableTimer(NULL, TRUE, "MyWaitableTimer");  
	    if(timer == NULL)  
		return FALSE;  
	  
	    __int64 nanoSecs;  
	    LARGE_INTEGER due;  
	  
	    nanoSecs = -secs * 1000 * 1000 * 10;  
	    due.LowPart = (DWORD) (nanoSecs & 0xFFFFFFFF);  
	    due.HighPart = (LONG) (nanoSecs >> 32);  
	  
	    if(!SetWaitableTimer(timer, &due, 0, 0, 0, TRUE))  
		return FALSE;  
	  
	    if(GetLastError() == ERROR_NOT_SUPPORTED)  
		return FALSE;  
	  
	    HANDLE thread = CreateThread(NULL, 0, ShutDownProc, &timer, 0, NULL);  
	    if(!thread)  
	    {  
		CloseHandle(timer);  
		return FALSE;  
	    }  
	  
	    CloseHandle(thread);  
	    SetSystemPowerState(FALSE, FALSE);  
	    return TRUE;  
	}  

Once your PC is turned on, it will be ready for you to do whatever you would like to do.
