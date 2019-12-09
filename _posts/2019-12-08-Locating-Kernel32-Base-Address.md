---
layout: post
title: "Shellcoding: Locating Kernel32 Base Address"
---
Before the shellcode will we able to locate or use any Windows APIs, the base address of kernel32.dll must be located. Thankfully there’s a trick for that that has been around for quite some time. The method Uses the Win32 [Thread Information Block](https://en.wikipedia.org/wiki/Win32_Thread_Information_Block) (TIB) to locate the [Program Environment Block](https://docs.microsoft.com/en-us/windows/desktop/api/winternl/ns-winternl-_peb) (PEB) to locate InInitializationOrderModuleList, which contains the base address of kernel32.dll as it’s second list entry.

## 32-Bit Version
I will begin by demonstrating how to locate Kernel32.dll's base address in a 32-bit environment. As best as I can discern, from [SK Chong's](http://www.phrack.org/archives/issues/62/7.txt) 2001 Phrack article, this method was originally documented by researchers in "VX-zine". I was unable to locate the original source. Since SK's article is quite old, it is possible that the author's name(s) is/are lost to time. If someone knows the original source, it would be interesting knowledge and I would happily update this article. This method will work on all versions of Windows. It is not NULL byte free but, we will handle that in a future blog post. The following code will locate the base address of Kernel32:

{% highlight Nasm linenos %}
mov	eax,[fs:0x30]   ; Get the memory address of the PEB
                        ; structure and store it in EAX
mov	eax,[eax+0x0C]  ; Get the memory address of the PEB_LDR_DATA
                        ; structure and store it in EAX
mov	esi,[eax+0x1C]  ; Get the memory address of the first
                        ; LIST_ENTRY structure stored in the
                        ; InInitializationOrderModuleList and store
                        ; it in ESI
lodsd                   ; Loads the memory address of the second 
                        ; LIST_ENTRY structure
mov	ebx,[eax+0x08]  ; Save the base address of kernel32.dll in EBX
{% endhighlight %}

1. The first **MOV** instruction saves the address of the **PEB** structure in **EAX**. Windows uses the **FS** register to store the address of the **TIB** structure, the address of the **PEB** structure is located at an offset of **0x30** bytes. To illustrate what is happening here, let us take a look at some screen clips of **OllyDbg** with an example program loaded.

    a. The **FS** Register contains the address **0x7FFDE000**:
    
    <p align="center">
        <img src="/resources/images/2019-12-08-Locating-Kernel32-Base-Address/figure1.png" alt="Figure 1: FS Register Contains 0x7FFDE000">
    </p>
    <p align="center">
        <em>Figure 1: FS Register Contains 0x7FFDE000</em>
    </p>

    b. If we look at that location in the memory dump, we can see that the address of the **PEB** is located at an offset of **0x30** (48 bytes), its value is **0x7FFDF000** in this example:
    
    <p align="center">
        <img src="/resources/images/2019-12-08-Locating-Kernel32-Base-Address/figure2.png" alt="Figure 2: PEB Address at Offset 0x30">
    </p>
    <p align="center">
        <em>Figure 2: PEB Address at Offset 0x30</em>
    </p>

2. The second **MOV** instruction saves the address of the **PEB_LDR_DATA** structure in **EAX**. The **PEB_LDR_DATA** structure is located at an offset of **0x0C** bytes of the **PEB** structure, its value is **0x00341EA0** in this example:

    <p align="center">
        <img src="/resources/images/2019-12-08-Locating-Kernel32-Base-Address/figure3.png" alt="Figure 3: PEB_LDR_DATA Structure Address at Offset 0x0C">
    </p>
    <p align="center">
        <em>Figure 3: PEB_LDR_DATA Structure Address at Offset 0x0C</em>
    </p>

3. The third **MOV** instruction saves the address of the **InInitializationOrderModuleList** list in **ESI**. This list contains a list of the loaded modules. The first entry is **ntdll.dll**, the second is **kernel32.dll**, which is what we’re after. The first list entry address is at an offset of **0x1C** (28 bytes), it’s value is **0x00341F58** in this example:

    <p align="center">
        <img src="/resources/images/2019-12-08-Locating-Kernel32-Base-Address/figure4.png" alt="Figure 4: The InInitializationOrderModuleList list Address at Offset 0xC1">
    </p>
    <p align="center">
        <em>Figure 4: The InInitializationOrderModuleList list Address at Offset 0xC1</em>
    </p>
    
4. The **LODSD** instruction increments **ESI**, which contains the address of the first module, by one so that it now points to the **kernel32.dll** entry and saves the address in **EAX**, its value is **0x00342020** in this example:

    <p align="center">
        <img src="/resources/images/2019-12-08-Locating-Kernel32-Base-Address/figure5.png" alt="Figure 5: The LOSD Instruction Increments ESI to Point to the Next List Entry">
    </p>
    <p align="center">
        <em>Figure 5: The LOSD Instruction Increments ESI to Point to the Next List Entry</em>
    </p>
    
5. The final **MOV** instruction saves the base address of **kernel32.dll** in **EBX**. At an offset of **0x08** (8 bytes) from that location is where the base address of **kernel32.dll** is located, its value is **0x7C800000** in this example:

    <p align="center">
        <img src="/resources/images/2019-12-08-Locating-Kernel32-Base-Address/figure6.png" alt="Figure 6: The Base Address of Kernel32.dll Located at Offset 0x08">
    </p>
    <p align="center">
        <em>Figure 6: The Base Address of Kernel32.dll Located at Offset 0x08</em>
    </p>

## 64-Bit Version
Now, on to try the same thing on a 64-bit system. As I did in the first blog post, I will try the 32-bit code first to see if it works. Immediately, there is a problem:

<p align="center">
    <img src="/resources/images/2019-12-08-Locating-Kernel32-Base-Address/figure7.png" alt="Figure 7: fs:[0x30] is Invalid">
</p>
<p align="center">
    <em>Figure 7: fs:[0x30] is Invalid</em>
</p>
    
The **fs:[0x30]** value points to unknown memory, which is not going to help us. After referring to the [Win32 Thread Information Block](https://en.wikipedia.org/wiki/Win32_Thread_Information_Block) (TIB, a.k.a **Thread Environment Block** (**TEB**)) information on Wikipedia, it appears that on 64-bit systems the **Process Environment Block** (**PEB**) is located at **gs:[0x60]** instead. Before we go any further, let's see what else has changed. I decided to switch over to **WinDBG** to explore a little more.

### Examining the TIB(TEB) Structure
In **WinDBG** lets get the base address of the **TEB** using the **!teb** command:

<p align="center">
    <img src="/resources/images/2019-12-08-Locating-Kernel32-Base-Address/figure8.png" alt="Figure 7: Finding the Base Address of the TEB">
</p>
<p align="center">
    <em>Figure 8: Finding the Base Address of the TEB</em>
</p>

Next, using the base address we can examine the **TEB** structure with the following command:

{% highlight shell %}
dt _TEB 000007fffffde000
{% endhighlight %}

We can confirm that the **PEB** structure is located at an offset of **0x60**.

<p align="center">
    <img src="/resources/images/2019-12-08-Locating-Kernel32-Base-Address/figure9.png" alt="Figure 9: The PEB Structure is at Offset 0x60 of the TEB">
</p>
<p align="center">
    <em>Figure 9: The PEB Structure is at Offset 0x60 of the TEB</em>
</p>

### Examining the PEB Structure
In **WinDBG** the **PEB** structure can be dumped using the following command:

{% highlight shell %}
dt _PEB 000007fffffd9000
{% endhighlight %}

From the results we see that the **LDR** structure is at an offset of **0x18** of the **PEB** structures base address.

<p align="center">
    <img src="/resources/images/2019-12-08-Locating-Kernel32-Base-Address/figure10.png" alt="Figure 10: The LDR Structure is at Offset 0x18 of the PEB">
</p>
<p align="center">
    <em>Figure 10: The LDR Structure is at Offset 0x18 of the PEB</em>
</p>

### Examining the PEB_LDR_DATA List Structure
We know in that when working in 32-bit that the first list item of the **PEB_LDR_DATA** structure should contain information about the running program, **ntdll** as the second list item, and the third should contain information about **kernel32**. We will now attempt to verify that it is the same for a 64-bit system. First, to look at the list, issue the following command:

{% highlight shell %}
dt _PEB_LDR_DATA 000000007723d640
{% endhighlight %}

### Iterating the LDR Data Table to Find Kernel32
We are interested in the **InMemoryOrderModuleList** at offset **0x20**. We will begin with seeing what is there.

<p align="center">
    <img src="/resources/images/2019-12-08-Locating-Kernel32-Base-Address/figure11.png" alt="Figure 11: Examining the PEB_LDR_DATA Structure">
</p>
<p align="center">
    <em>Figure 11: Examining the PEB_LDR_DATA Structure</em>
</p>

To see what the first entry in the list points to, we will examine the **LDR_DATA_TABLE_ENTRY** structure contents by issuing the following command:

{% highlight shell %}
dt _LDR_DATA_TABLE_ENTRY poi(000000007723d640+0x20)
{% endhighlight %}

As we can see in **Figure 12**, the first entry is the current process. Which happens to be **notepad.exe** in this example.

<p align="center">
    <img src="/resources/images/2019-12-08-Locating-Kernel32-Base-Address/figure12.png" alt="Figure 12: Looking at the First LDR Data Table Entry">
</p>
<p align="center">
    <em>Figure 12: Looking at the First LDR Data Table Entry</em>
</p>

So far, so good, now we will look at the next entry. To do that, we need to evaluate what **000000007723d640+0x20** points to. Since we are looking through a doubly-linked list, this should bring us to the next entry. The following command will evaluate the value:

{% highlight shell%}
? poi(000000007723d640+0x20)
{% endhighlight %}

<p align="center">
    <img src="/resources/images/2019-12-08-Locating-Kernel32-Base-Address/figure13.png" alt="Figure 13: Evaluating the pointer of the Next LDR Data Table Entry">
</p>
<p align="center">
    <em>Figure 13: Evaluating the pointer of the Next LDR Data Table Entry</em>
</p>

With the evaluated value, we can now look at the next **LDR_DATA_TABLE_ENTRY** by issuing the following command:

{% highlight shell %}
dt _LDR_DATA_TABLE_ENTRY poi(00000000003c34e0)
{% endhighlight %}

The next entry points to **ntdll.dll**. This may be useful. It should not be necessary to know this address for the final shellcode but, we can note it's location in the list.

<p align="center">
    <img src="/resources/images/2019-12-08-Locating-Kernel32-Base-Address/figure14.png" alt="Figure 14: The Second LDR Data Table Entry points to ntdll.dll">
</p>
<p align="center">
    <em>Figure 14: The Second LDR Data Table Entry points to ntdll.dll</em>
</p>

To look at the next entry in the list, we need to evaluate the address of the next entry. The first element of the **LDR_DATA_TABLE_ENTRY** structure is a doubly-linked list. To find the **FLink**, we just need to find evaluate the pointer, then we can look at the next entry. The commands to do this are:

{% highlight shell %}
? poi(00000000003c34e0)
dt _LDR_DATA_TABLE_ENTRY poi(00000000003c35f0)
{% endhighlight %}

Excellent, we found the entry for **kernel32.dll**.

<p align="center">
    <img src="/resources/images/2019-12-08-Locating-Kernel32-Base-Address/figure15.png" alt="Figure 15: The Third LDR Data Table Entry points to kernel32.dll">
</p>
<p align="center">
    <em>Figure 15: The Third LDR Data Table Entry points to kernel32.dll</em>
</p>

### Putting It All Together
Now that we know how to find the LDR Data Table entry that contains the details for the **kernel32.dll** module, it is time to write some Assembly that will duplicate what we did in the previous section so that we can locate and store the base address of **kernel32** so that it can be used to locate Win32 APIs.

We begin by listing the steps we performed in **WinDBG** to find the base address of **kernel32**.

1. Find the address of the **PEB** structure.
2. Find the address of the **PEB_LDR_DATA** structure.
3. Find the address of the **InMemoryOrderModuleList** list.
4. Iterate to the third **InMemoryOrderModuleList** entry.
5. Store the base address of **kernel32.dll**.

With this basic list of tasks we will begin to assemble some Assembly (heh...) to accomplish the required tasks.

#### 1: Find the Address of the PEB Structure
According to the Wikipedia article on the [Thread Information Block](https://en.wikipedia.org/wiki/Win32_Thread_Information_Block) is stored in the **gs** register and the **PEB** is located at an offset of **0x60**. With this information, the following Assembly instruction should store the address of the **PEB** in the **RAX** register:

{% highlight Nasm %}
mov rax, [gs:0x60]
{% endhighlight %}

#### 2: Find the Address of the PEB_LDR_DATA Structure
The **PEB_LDR_DATA** structure is located at an offset of **0x18** from the base address of the **PEB**. The following instruction should store the address of the **PEB_LDR_DATA** structure in the **RAX** register.

{% highlight Nasm %}
mov rax, [rax+0x18]
{% endhighlight %}

#### 3: Find the Address of the InMemoryOrderModuleList list
The **InMemoryOrderModuleList** is at the offset of **0x20** of the **PEB_LDR_DATA** structure. The following assembly will store the address in the **RAX** register:

{% highlight Nasm %}
mov rax, [rax+0x20]
{% endhighlight %}

#### 4: Iterate to the Third InMemoryOrderModuleList Entry
The **InMemoryOrderModuleList** is a doubly-linked list. The first element of each list is the **FLink**, so we can use this to quickly iterate to the 3rd entry in the list by loading the first pointer twice. The following code will leave the address of the third entry in the **RAX** register:

{% highlight Nasm %}
mov rax, [rax]
mov rax, [rax]
{% endhighlight %}

#### 5: Save the Base Address of Kernel32
The final step is to save the base address of **kernel32.dll** somewhere so that we can use it later to retrieve other Win32 functions. According to [Microsoft's x64 Calling Convention](https://docs.microsoft.com/en-us/cpp/build/x64-calling-convention?view=vs-2019#callercallee-saved-registers) the **R12-R15** registers are Caller/Callee Saved registers. This means that we should be able to reserve one of these registers to store the base address of **kernel32.dll** for the remainder of our shellcode. The base address is found at an offset of **0x20** from the beginning of the list entry. The code to store the base address in the **R12** register is:

{% highlight Nasm %}
mov r12, [rax+0x20]
{% endhighlight %}

#### The Complete Assembly
Putting it all together, the assembly program to find the base address of **kernel32.dll** and store it in the **R12** register is as follows:

{% highlight Nasm linenos %}
[SECTION .text]

BITS 64

global _start

_start:
    mov rax, [gs:0x60]
    mov rax, [rax+0x18]
    mov rax, [rax+0x20]
    mov rax, [rax]
    mov rax, [rax]
    mov r12, [rax+0x20]
{% endhighlight %}

As you can see in **Figure 16**, the base address of **kernel32.dll** has been stored successfully in **R12**.

<p align="center">
    <img src="/resources/images/2019-12-08-Locating-Kernel32-Base-Address/figure16.png" alt="Figure 16: The Base Address of Kernel32.dll is Stored in R12">
</p>
<p align="center">
    <em>Figure 16: The Base Address of Kernel32.dll is Stored in R12</em>
</p>

Alternately, if you needed to save the base address of **ntdll.dll** as well you could use a slight variation, this variation will store the base address of **ntdll.dll** in the **R13** register:

{% highlight Nasm linenos %}
[SECTION .text]

BITS 64

global _start

_start:
    mov rax, [gs:0x60]
    mov rax, [rax+0x18]
    mov rax, [rax+0x20]
    mov rax, [rax]
    mov r13, [rax+0x20]
    mov rax, [rax]
    mov r12, [rax+0x20]
{% endhighlight %}

# Conclusion
In this installment of my shellcoding blog series we learned how to find the base address of **kernel32.dll** using both x86 and x64 Assembly code. The ability to find the base address of **kernel32.dll** is important because it can be used to call the **LoadLibrary** Win32 function to locate the address of other interesting Win32 functions, and even load other DLLs to add additional functionality. Like adding **ws2_32.dll** to provide access to functions like: accept, bind, socket, WSASocket, and WSAStartup. The fact that our finished opcodes includes NULL bytes is acceptable since we will, in a later stage, use an encoding routine to avoid NULL bytes.
