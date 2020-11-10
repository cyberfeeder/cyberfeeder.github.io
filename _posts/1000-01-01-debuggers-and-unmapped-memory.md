---
layout: page
title: Debuggers and Unmapped Memory
---

Chances are if you've written C or C++ that you've used a debugger.
Having written C/C++, you're also likely to have cut your teeth on
handling memory. 
Recently, I ran into a runtime error that caught my attention.

The mistake was simple, my code accessed unmapped memory. To illustrate an issue of the same type, consider the following example program.

{% highlight cpp %}
int main()
{
    int* p = (int*)0x1122334455667788;
    return *p;
}
{% endhighlight %}

This program is almost certain to fail.
There is a slight chance that `0x1122334455667788`
is actually valid memory by chance.
Despite some low probability of success,
we expect this program to fail.

The reason we expect it to fail is that we don't expect
`0x1122334455667788` to be a mapped address.
So, if we try to read four bytes from that location,
we should see the program crashing. 

![crash1](/assets/images/debuggers-and-unmapped-memory/crash1.png){:class="img-responsive"}

The message says: "Access violation reading location
0xFFFFFFFFFFFFFFFF". The address it presents is somewhat peculiar.
The address we tried to access is not the one reflected in the message.
So, let's try to alter the program a little bit.

{% highlight cpp %}
int main()
{
    int* p = (int*)0x1122334455667788;
    return *(p + 10);
}
{% endhighlight %}

Notice that we now access `p + 10`. Given the size of an `int`,
in this case four bytes, the offset should be:

 `0x1122334455667788 + 10 * 4 =  0x11223344556677B0`
 
 If we run this program, we get an even stranger message.
 
![crash2](/assets/images/debuggers-and-unmapped-memory/crash2.png){:class="img-responsive"}

The message now says that p is `0xFFFFFFFFFFFFFFD7` which is not true at all.
But what causes this message? The instruction that
triggers the crash is `mov eax, dword ptr [rax+28h]`.
In the former example, without the offset, the instruction
was `mov eax,dword ptr [rax]`. In both cases, `rax` is in
fact `0x1122334455667788` so the debugger should be able
to show us the right address. If we run the latest version
of the program in WinDbg, we get the following message.

![crash2windbg](/assets/images/debuggers-and-unmapped-memory/crash2windbg.png){:class="img-responsive"} 

This shows us an invalid address that does make sense.
Here the address that we're not allowed to read from is `0x11223344556677B0`.

**So again, why didn't the Visual Studio debugger show the same message?**
