# Always to be Top Window (Need Windows Private Api Calls)

# Get UIAccess via System token

This project is used to obtain UIAccess permissions, which allows your program window to obtain a higher Z order, such as higher than the task manager, etc., and on the same layer as the on-screen keyboard. It can be used to solve the problem of the window being blocked when making screen marking/recording tools.

## Effect comparison

Taking the Task Manager as an example, first open the "Send to Top" of the Task Manager. Its window Band is `ZBID_SYSTEM_TOOLS`, which is higher than the regular window Band.

When UIAccess is not enabled, regardless of whether `SetWindowPos(HWND_TOPMOST)` is used, the window Z order is always lower than that of the task manager:


After enabling UIAccess and calling `SetWindowPos(HWND_TOPMOST)`, the window Z order will be higher than the task manager:


## Conditions and usage

The program needs to be run with elevated privileges (`elevated`), so it is best to set up a list that requests administrator permissions, or start it through a process that has been elevated. Otherwise, UIAccess cannot be obtained, and the function return value is `ERROR_NOT_FOUND`. After adding the header file and source file, just call `PrepareForUIAccess()` at the beginning of the program. If the setting is successful, it will return `ERROR_SUCCESS`, otherwise it will return an error code.

## Program principle

> Compared with the previous version, the problem of UIAccess setting failure when the user permission "Replace Process Token" is turned off has been fixed; the program has changed from starting the process 3 times to 2 times, and there is no need to start another process with System permissions, eliminating IPC communication process.

The process starts with administrator privileges and then checks whether it has UIAccess privileges. It has not yet obtained permissions at this time, so it goes through the process list, tries to obtain the token of `winlogon.exe` under the same Session, and uses this token to create another token with `TokenUIAccess`, and then uses it to start another instance. This instance detects the UIAccess permissions. If the permissions are met, `ERROR_SUCCESS` is returned. Then the old process exits and the new process with permissions continues to run.

## Introduction to window Z order

In Windows 7 and below systems, directly use `SetWindowPos(HWND_TOPMOST)` to make the window on top. But starting with Windows 8, Microsoft introduced other window segments (Band). Their order from low to high is as follows:

```
ZBID_DESKTOP
ZBID_IMMERSIVE_BACKGROUND
ZBID_IMMERSIVE_APPCHROME
ZBID_IMMERSIVE_MOGO
ZBID_IMMERSIVE_INACTIVEMOBODY
ZBID_IMMERSIVE_NOTIFICATION
ZBID_IMMERSIVE_EDGY
ZBID_SYSTEM_TOOLS
ZBID_LOCK（仅Windows 10）
ZBID_ABOVELOCK_UX（仅Windows 10）
ZBID_IMMERSIVE_IHM
ZBID_GENUINE_WINDOWS
ZBID_UIACCESS
```

The default window segment is `ZBID_DESKTOP`, which causes the Z-order of the window to always be lower than that of windows with other higher-level segments set no matter `SetWindowPos`.

So why not set other window segments?

There are the following APIs in Windows that can change the window segment of a program:

```c
HWND WINAPI CreateWindowInBand(
	DWORD dwExStyle,
  	LPCWSTR lpClassName,
	LPCWSTR lpWindowName,
	DWORD dwStyle,
	int x,
	int y,
	int nWidth,
	int nHeight,
	HWND hWndParent,
	HMENU hMenu,
	HINSTANCE hInstance,
	LPVOID lpParam,
	DWORD dwBand
);
HWND WINAPI CreateWindowInBandEx(
	DWORD dwExStyle,
  	LPCWSTR lpClassName,
	LPCWSTR lpWindowName,
	DWORD dwStyle,
	int x,
	int y,
	int nWidth,
	int nHeight,
	HWND hWndParent,
	HMENU hMenu,
	HINSTANCE hInstance,
	LPVOID lpParam,
	DWORD dwBand,
	DWORD dwTypeFlags
);
BOOL WINAPI SetWindowBand(
	HWND hWnd, 
	HWND hwndInsertAfter, 
	DWORD dwBand
);
```

But the program that calls `CreateWindowInBand(Ex)` must be digitally signed with a Microsoft certificate. That is to say, only Windows built-in programs can use these APIs. This is what the Task Manager does. And `SetWindowBand` needs to call the private API: `NtUserEnableIAMAccess`, which has a handle-like parameter (key). This handle can only be obtained through `NtUserAcquireIAMKey`. The condition for successful calling of `NtUserAcquireIAMKey` is that the calling thread must be the current desktop thread (that is, the thread that calls `SetShellWindows(Ex)`), and it can only be obtained once, otherwise the function will be `ERROR_ACCESS_DENIED`, and you cannot even inject `explorer. exe` gets the key because `explorer.exe` has already called `NtUserAcquireIAMKey` once. In other words, only the desktop manager can use `SetWindowBand`.

Is there any other way? Note that the windows of the on-screen keyboard (`osk.exe`) and `Inspect.exe`, a tool of VS, can also be set to be higher than the task manager. After reverse reversal, I found that they were just `SetWindowPos(HWND_TOPMOST)`. Finally, I found that there was an item in the program's list:

```
<requestedExecutionLevel level="asInvoker" uiAccess="true"/>
```
## 📬 Contact

Have questions or want to contribute?

- **Telegram**: [@somerwork](https://t.me/somerwork)
- **Donate(BTC)**: bc1q43u0n865fuxc4j2vgm4wp98xuuaawgkgq8yrf4
---
