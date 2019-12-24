---
layout: post
title: "Locating Win32 Functions"
---
To recap, in the first two shellcoding posts we [Located EIP/RIP](https://blog.xenoscr.net/Finding-EIP/) and [Located the base address of Kernel32](https://blog.xenoscr.net/Locating-Kernel32-Base-Address/). In this post we will attempt to locate [Microsoft Win32 API](https://docs.microsoft.com/en-us/windows/win32/api/) functions. The ability to locate specific functions will allow us to perform virtually anything task that is required to gain a shell. 

# Locating the Image Export Directory Structure
To locate the addresses of the APIs needed our shellcode must loop through the named functions in the **IMAGE_EXPORT_DIRECTORY** of a DLL. In this instance the **kernel32.dll** is going to be searched to locate the desired APIs. The **IMAGE_EXPORT_DIRECTORY** structure is defined in the **winnt.h** header file as follows:

{% highlight c linenos %}
typedef struct _IMAGE_EXPORT_DIRECTORY {
    DWORD   Characteristics;
    DWORD   TimeDateStamp;
    WORD    MajorVersion;
    WORD    MinorVersion;
    DWORD   Name;
    DWORD   Base;
    DWORD   NumberOfFunctions;
    DWORD   NumberOfNames;
    DWORD   AddressOfFunctions;     // RVA from base of image
    DWORD   AddressOfNames;         // RVA from base of image
    DWORD   AddressOfNameOrdinals;  // RVA from base of image
} IMAGE_EXPORT_DIRECTORY, *PIMAGE_EXPORT_DIRECTORY;
{% endhighlight %}

## Locating the Export Table
The first task that must be completed before the **IMAGE_EXPORT_DIRECTORY** structure can be searched for the APIs is to locate it. To do this, the shellcode uses the following instructions. For the purposes of this blog post we will assume that the base address of **kernel32.dll** is stored in the **EBX** register:

{% highlight Nasm linenos %}
push ebx                        ; Preserve EBX
mov ebp, [esp]                  ; DLL Base Address  
mov eax, [ebp + 0x3c]           ; eax = PE header offset  
mov edx, [ebp + eax * 1 + 0x78]
add edx, ebp                    ; edx = exports directory table
{% endhighlight %}

1. The shellcode begins by preserving the **EBX** register so that it can be restored later. 
2. Next the PE header is located at an offset of **0x3C** (60) bytes from the base address of the DLL. 
3. Next the **IMAGE_EXPORT_DIRECTORY** address is located at an offset of **0x78** (120) bytes from the base address of the PE header.

**NOTE**: To view the whole **PE** file structure, check out the excellent image included in Wikipedia’s Portable Executable article: [https://en.wikipedia.org/wiki/Portable_Executable](https://en.wikipedia.org/wiki/Portable_Executable)  

Loc

<p align="center">
    <img src="/resources/images/2019-12-13-Locating-Win32-Functions/figure1.png" alt="Figure 1: IMAGE_EXPORT_DIRECTORY Visualization">
</p>
<p align="center">
    <em>Figure 1: IMAGE_EXPORT_DIRECTORY Visualization</em>
</p>
