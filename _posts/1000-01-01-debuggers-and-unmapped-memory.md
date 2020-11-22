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
some addresses, it seems the crossing point is at `0x0000800000000000`. At lower addresses, Visual Studio shows the right address
in the exception window. At addresses higher then  `0x0000800000000000`, we get that `0xFFFFFFFFFFFFFFFF` address
anomaly. 

There is a cue about this in the MSDN documentation. [This page][1] tells us that the most virtual memory a 64-bit
the process can have is `128 TB`. We'll assume that TB is in fact tebibyte. `128 TB` is `140737500000000` bytes.
`140737500000000` in hexadecimal is `0x0000800000000000`. A process cannot, under Windows 10, have more than `128 TB` of
virtual memory. Visual Studio seems unable to get the correct address outside the `128 TB` range.

### Conclusion

This conclusion is rather weak. We don't have any direct evidence for why MSVS shows the rather strange address.
We do have a clue by the memory limitation at `128 TB`. The strange message occurs after addressing memory beyond the
`128 TB` limit. Thus, we don't know for sure why we see the results we see. It does yet seem like faulty address
messages relate to the virtual memory limit in Windows.

[1]:https://docs.microsoft.com/en-us/windows/win32/memory/memory-limits-for-windows-releases#memory-and-address-space-limits