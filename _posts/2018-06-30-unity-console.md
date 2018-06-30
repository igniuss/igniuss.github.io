---
layout: post
title: Unity3D Console
categories: [unity3D]
tags: [unity3D]
comments: true
permalink: unity3D_console
---

![Example](https://i.imgur.com/HIO71Yl.png)

Ever wanted to add a console output to your Unity Application? 

If the answer is yes, then let's get cracking!👏👏


# Introduction 
Let's first start by making a little manager monobehaviour that takes care of everything 😄

```csharp
using UnityEngine;
using System;
public class ConsoleManager : MonoBehaviour {
  public void Start() {

  }
}
```

Next we'll need to define the Free & Alloc console functions from the kernel32 dll.

```csharp
[DllImport("Kernel32.dll")]
private static extern bool AllocConsole();

[DllImport("kernel32.dll", SetLastError = true, ExactSpelling = true)]
private static extern bool FreeConsole();
```
The `DLLImport` attributes are in the `System.Runtime.InteropServices` namespace, so just add 

`using System.Runtime.InteropServices` on the top of your file to get rid of those pesky errors.

### So let's talk about these imports for a second.

`AllocConsole` is used to initialize standard input, output, and error handles for a new console, that'll be bound to your process.

A process can be associated with **only a single console**, so the `AllocConsole` function fails if the process already has one.

That's why we have `FreeConsole`, that's used to 'Detach' the process from the current console, so you can call `AllocConsole` again to attach a new one.

So, now that we know what these functions do, *how do we use them?*

```csharp
private void Start() {
  AllocConsole();
}
```
Now when the monobehaviour starts, it'll spawn a new console! 

And *this is where we run into our first issue*

### Exiting playmode, doesn't kill the console.. and closing the console *also closes the editor*

Thankfully, we can detect when a game (or play-mode for that matter) stops

```csharp
private void OnDestroy() {
  FreeConsole();
}
```

Now when you exit playmode, or the game, it'll free and close the console, without killing Unity!

*This doesn't stop the editor from closing when you close the console manually though...*

# How to actually use it

So we've got a console now, but how do we interact with it? 

In this example, I'll hook up Unity's logging to the console, so I can view the output in builds.

So we'll hook up `Application.logMessageReceivedThreaded` to a function that'll format the output, and write it to the console.

```csharp
private void Start() {
  AllocConsole();
  Application.logMessageReceivedThreaded += OnLogMessageReceived;
}
private void OnLogMessageReceived(string condition, string stackTrace, LogType type) {
        ConsoleColor color = Console.ForegroundColor;
        switch (type) {
            case LogType.Log:
            case LogType.Assert:
                color = ConsoleColor.White;
                break;
            case LogType.Exception:
            case LogType.Error:
                color = ConsoleColor.Red;
                break;
            case LogType.Warning:
                color = ConsoleColor.Yellow;
                break;
        }
        Console.ForegroundColor = ConsoleColor.Black;
        Console.ForegroundColor = color;
        Console.Write(string.Format("[{0}] ", DateTime.UtcNow.ToString("d/M/yyyy hh:mm")));
        Console.ResetColor();
        Console.WriteLine(condition);
    }
```

So we make a callback for `logMessageReceivedThreaded`, which takes a condition, stacktrace, and logtype.

We then select a color based on the logtype, and make a message that'll show a new message in the console, looking something like this!

`[30/7/2018 20:00] We've got ourselves a log!`

*Except*, we have a tiny problem..
Nothing is showing up in the console!

This is because `AllocConsole` initializes *standard input, output and error handles, __which we can't use!__*

![my-own-console]({{site.BASE_PATH}}/assets/img/unity3D-console/myownconsole.png)


So.. *Let's fix that.*

```csharp
private TextWriter original_consoleout;
private StreamWriter new_consoleout;

private void Start() {
  AllocConsole();
  new_consoleout = new StreamWriter(Console.OpenStandardOutput());
  writer.AutoFlush = true;
  original_consoleout = Console.Out;
  Console.SetOut(new_consoleout);
  Application.logMessageReceivedThreaded += OnLogMessageReceived;
}
```
You can also overwrite the Input, and process those inside your Unity instance.

I'll leave that up for someone else (or maybe a future tutorial 🤷‍)


*But* we forgot some things, we gotta make sure to set the out back to the original in our OnDestroy function, *and* dispose our writer (otherwise, it'll stay in memory when you exit play-mode, or destroy the script)

```csharp
private void OnDestroy() {
  FreeConsole();
  if (new_consoleout != null) {
    new_consoleout.Flush();
    new_consoleout.Dispose();
  }
  Console.SetOut(original_consoleout);
}
```

Now, the reason why we 'Reset' the Console out, is because Unity internally, calls `Console.WriteLine`, and the Unity-Editor console will bug out if you don't set it back 😉

# Conclusion

Allocating a new console is pretty simple in Windows, especially when you're using C# as a runtime.

But there are a couple of things you need to take into consideration.
* Don't forget, Closing a console **will also close the process**
* Manually set the console out to use the `Console.Write/WriteLine` functions
* Be sure to reset the console out when you're inside of Unity, to not bug out the editor
* Always Dispose things that should be disposed

One of these points I haven't mentioned in much detail yet, closing the console **will also close the process**...


Which, for a headless server application, *is pretty good!*

So when the server intializes, a new console window opens up, and when you close it, the Unity Instance closes as well! Totally intended behaviour!

*Not so much intended behaviour when you're doing this with a non-headless game.*

I haven't found a way to stop that from happening yet (but, I haven't done much research in this either. I just needed a quick solution for our servers)


<details>
<summary>The final source</summary>

The final script has a couple of differences, mostly just variable and function names.

<pre>
using System;
using System.IO;
using System.Runtime.InteropServices;
using UnityEngine;


public class ConsoleManager : MonoBehaviour {
#if UNITY_STANDALONE_WIN //Only get behaviour when we're running for windows
    private TextWriter original;
    private StreamWriter writer;

    [DllImport("kernel32.dll", SetLastError = true, ExactSpelling = true)]
    private static extern bool FreeConsole();

    [DllImport("Kernel32.dll")]
    private static extern bool AllocConsole();

#if UNITY_EDITOR
    public bool run = true;
#endif

    private void Awake() {
        var args = Environment.GetCommandLineArgs();
        foreach (var arg in args) {
            if (arg == "-console" || arg == "-c") {
                StartConsole();
                break;
            }
        }
#if UNITY_EDITOR
        if (run) { StartConsole(); return; }
#endif

        Debug.Log("No Console command found, destroying consolemanager.");
        GameObject.Destroy(this);
    }

    private void StartConsole() {
        AllocConsole();
        writer = new StreamWriter(Console.OpenStandardOutput());
        writer.AutoFlush = true;
        original = Console.Out;
        Console.SetOut(writer);
        Application.logMessageReceivedThreaded += OnLogMessage;
    }

    private void OnLogMessage(string condition, string stackTrace, LogType type) {
        ConsoleColor color = Console.ForegroundColor;
        switch (type) {
            case LogType.Log:
            case LogType.Assert:
                color = ConsoleColor.White;
                break;

            case LogType.Error:
                color = ConsoleColor.Red;
                break;

            case LogType.Exception:
            case LogType.Warning:
                color = ConsoleColor.Yellow;
                break;
        }
        Console.ForegroundColor = ConsoleColor.Black;
        Console.ForegroundColor = color;
        Console.Write(string.Format("[{0}] ", DateTime.UtcNow.ToString("d/M/yyyy hh:mm")));
        Console.ResetColor();
        Console.WriteLine(condition);
    }

    private void OnDestroy() {
        FreeConsole();
        if (writer != null) {
            writer.Flush();
            writer.Dispose();
        }
        Console.SetOut(original);
    }
#endif
}
</pre>
</details>