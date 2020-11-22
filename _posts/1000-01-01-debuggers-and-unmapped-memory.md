---
layout: post
title: Debuggers and Unmapped Memory
---

Chances are if you've written C or C++ that you've used a debugger. Having written C/C++, you're also likely to have cut
your teeth on handling memory. Recently, I ran into a runtime error that caught my attention.

The mistake was simple, my code accessed unmapped memory. To illustrate an issue of the same type, consider the
following example program.

{% highlight cpp %}
int main()
{
    int* p = (int*)0x1122334455667788;
    return *p;
}
{% endhighlight %}


This program is almost certain to fail. There is a slight chance that `0x1122334455667788` is actually valid memory by
chance. Despite some low probability of success, we expect this program to fail.

The reason we expect it to fail is that we don't expect `0x1122334455667788` to be a mapped address. So, if we try to
read four bytes from that location, we should see the program crashing. Visual Studio displays a message with the
following text.

{% highlight md %}
Exception Thrown

Exception thrown: read access violation.
address was 0xFFFFFFFFFFFFFFFF.
{% endhighlight %}

The address it presents is somewhat peculiar. The address we tried to access is not the one reflected in the message.
So, let's try to alter the program a little bit.

{% highlight cpp %}
int main()
{
    int* p = (int*)0x1122334455667788;
    return *(p + 10);
}
{% endhighlight %}

Notice that we now access `p + 10`. Given the size of an `int`, in this case four bytes, the offset should be:
`0x1122334455667788 + 10 * 4 =  0x11223344556677B0`

 If we run this program, we get an even stranger message.

{% highlight md %}
Exception Thrown

Exception thrown: read access violation.
address was 0xFFFFFFFFFFFFFFD7.
{% endhighlight %}

The message now says that p is `0xFFFFFFFFFFFFFFD7` which is not true at all. But what causes this message? The
instruction that triggers the crash is `mov eax, dword ptr [rax+28h]`. In the former example, without the offset, the
instruction was `mov eax, dword ptr [rax]`. In both cases, `rax` is in fact `0x1122334455667788` so the debugger should
be able to show us the right address. If we run the latest version of the program in WinDbg, we get the following
message.

{% highlight md %}
CppTestProject!main+0x3c:
00000000`00d8173c 8b4028
mov eax,dword ptr [rax+28h] ds:11223344`556677b0=????????
{% endhighlight %}

This shows us an invalid address that does make sense. Here the address that we're not allowed to read from is
`0x11223344556677B0`.

### So, why didn't the Visual Studio debugger show the same message?

Trying to invalid memory at lower addresses does not seem to be a problem for the VS debugger. After messing around with
some addresses, it seems the crossing point is at `0x0000800000000000`. At lower addresses, Visual Studio shows the
right address in the exception window. At addresses higher then  `0x0000800000000000`, we get that `0xFFFFFFFFFFFFFFFF`
address anomaly.

There is a cue about this in the MSDN documentation. [This page][1] tells us that the most virtual memory a 64-bit
the process can have is `128 TB`. We'll assume that TB is in fact tebibyte. `128 TB` is `140737500000000` bytes.
`140737500000000` in hexadecimal is `0x0000800000000000`. A process cannot, under Windows 10, have more than `128 TB` of
virtual memory. Visual Studio seems unable to get the correct address outside the `128 TB` range.

### But what does the debugger see?

We suspect that reading data from addresses above `0x8000000000` triggers the wrongful message. The question is now
whether it's the debugger or some underlying mechanism that is to blame. To test if it's the debugger, we take the
following program. We compile this program into an exe `CrashingProgram.exe`.

{% highlight cpp %}
#include <Windows.h>

int main()
{
    Sleep(500);

    char* p = (char*)0x800000000000;
    char x = *p;

    return 0;
}

{% endhighlight %}

This program reads a single byte at address `0x8000000000`. Running the program from the debugger, we get the message
that we expect from earlier. We get the access violation error message, and the expected address as `0xFF...`.

We now make another program that launches `CrashingProgram.exe` and attach to it. We call this new program
`debugger.exe`. [This gist][2] contains the full program. `debugger.exe` uses the functions provided by WinAPI to
receive debug events. Using a custom program, we can get to know the exception details passed by Windows. Windows will
throw a bunch of debug events during normal execution.  The `debugger` program ingores most of the debugging events.
The only event that we do care about are access violation exceptions. In case you're curious, [this page][3] lists the
various event types.

Using the `debugger` program, we can check the exception information given by Windows. It turns out that the event from
Windows claims that the violation was at `0xFFFFFF...`. If the `CrashingProgram` uses an address only one byte lower,
we get the correct address. That is, accessing `0x7FFFFFFFFFFF`, the debug event contains the correct address.

### Conclusion

The conclusion of this investigation is somewhat lacking. We've reached a point where the strange behavior comes from
Windows itself. Digging deeper would involve quite a bit more effort and with uncertain results.

We can summerise our findings though. It turns out that Windows has an internal limit for virtual memory allocation.
This limitation suggests a few things. First, Windows seem unwilling to allow memory allocation at addresses above
`0x800000000000`. Further access violation events report address `-1` above the limit range. It seems fair to assume
that Windows misreports these addresses with intention.

Last, we saw that WinDbg did report the right address. From this, we must assume that WinDbg is doing some extra work.
WinDbg could be cross-checking with register values and current instruction.

Again, we don't know for sure why Visual Studio shows this wrong message. Though, we do know that if we see an access
violation at address `-1`, we should take that with a grain of salt.

[1]:https://docs.microsoft.com/en-us/windows/win32/memory/memory-limits-for-windows-releases#memory-and-address-space-limits
[2]:https://gist.github.com/egomeh/c632406d67e5bdf2906779d743ebbe9b
[3]:https://docs.microsoft.com/en-us/windows/win32/api/minwinbase/ns-minwinbase-debug_event