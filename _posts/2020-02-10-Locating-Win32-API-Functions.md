---
layout: post
title: "Locating Win32 API Function Addresses"
---
Welcome back old readers, hello new readers! It has been a little over a month since my last blog post. Life is full and keeps me busy! In the last blog [post](https://blog.xenoscr.net/Encoding-Decoding-and-Storing-Function-Names/) I wrote about encoding function names using a hashing routine. As a reminder, encoding the function names served the purpose saving space as well as hiding the function names from being stored in an easily detectable string format.

In this installment of the shellcodee writing series, we will examine a routine that will use what we learned in the previous blog posts. To find the address of a Win32 API function we will need the [base address of kernel32](https://blog.xenoscr.net/Locating-Kernel32-Base-Address/), and an [encoded and stored](https://blog.xenoscr.net/Encoding-Decoding-and-Storing-Function-Names/) Win32 API function value. To keep things simple and consistent we will focus on the function name from the previous blog post **LoadLibraryA** which encodes to **0xEC0E4E8E**.

This post is going to get a bit into the weeds. We will attempt to explain how the assembly code is navigating the [PE File Header](https://en.wikipedia.org/wiki/Portable_Executable#Layout) to find the [Export Directory structure](https://docs.microsoft.com/en-us/windows/win32/debug/pe-format#export-directory-table) where the list of exported functions resides. We will then explain how the code steps through each of the exported function names, hashes them, and finally checks for a match. This will be a fairly involved post, do not be discouraged if you need to read through it a few times. I would encourage you to put the code into a debugger and step through it instruction by instruction to gain a better understanding of what is happening if needed.

# Win32 API Locating Assembly, in 32-Bit
Enough yammering, time for some assembly code. The following assembly program will locate the address of the **LoadLibraryA** function and store it at at an address pointed to by **EBP**. The code is a bit on the long side. Do not worry, we will break down what each section is doing following the main listing.

{% highlight Nasm linenos %}
[SECTION .text]

BITS 32

_start:
    jmp main
    
    ; Constants
    win32_library_hashes:
        call win32_library_hashes_return
        ; LoadLibraryA
        dd 0xEC0E4E8E
    
    ; ======== Function: find_kernel32
    find_kernel32:
        push esi
        xor eax, eax
        mov eax, [fs:eax+0x30]
        mov eax, [eax+0x0C]
        mov esi, [eax+0x1C]
        lodsd
        mov eax, [eax+0x08]
        pop esi
        ret
        
    ; ======= Function: find_function
    find_function:
        pushad
        mov ebp, [esp+0x24]
        mov eax, [ebp+0x3C]
        mov edx, [ebp+eax+0x78]
        add edx, ebp
        mov ecx, [edx+0x18]
        mov ebx, [edx+0x20]
        add ebx, ebp
    find_function_loop:
        jecxz find_function_finished
        dec ecx
        mov esi, [ebx+ecx*4]
        add esi, ebp
        
    compute_hash:
        xor edi, edi
        xor eax, eax
        cld
    compute_hash_again:
        lodsb
        test al, al
        jz compute_hash_finished
        ror edi, 0x0D
        add edi, eax
        jmp compute_hash_again
    compute_hash_finished:
    find_function_compare:
        cmp edi, [esp+0x28]
        jnz find_function_loop
        mov ebx, [edx+0x24]
        add ebx, ebp
        mov cx, [ebx+2*ecx]
        mov ebx, [edx+0x1C]
        add ebx, ebp
        mov eax, [ebx+4*ecx]
        add eax, ebp
        mov [esp+0x1C], eax
    find_function_finished:
        popad
        ret
        
    ; ======== Function: resolve_symbols_for_dll
    resolve_symbols_for_dll:
        lodsd
        push eax
        push edx
        call find_function
        mov [edi], eax
        add esp, 0x08
        add edi, 0x04
        cmp esi, ecx
        jne resolve_symbols_for_dll
    resolve_symbols_for_dll_finished:
        ret
        
    main:
        sub esp, 0x88                       ; Allocate space on stack for function addresses
        mov ebp, esp                        ; Set ebp as frame ptr for relative offset on stack
        call find_kernel32                  ; Find base address of kernel32.dll
        mov edx, eax                        ; Store base address of kernel32.dll in EDX
        jmp short win32_library_hashes
        win32_library_hashes_return:
        pop esi
        lea edi, [ebp+0x04]                 ; This is where we store our function addresses
        mov ecx, esi
        add ecx, 0x04                       ; Length of kernel32 hash list
        call resolve_symbols_for_dll
{% endhighlight %}

# The Main Function
## Setup the Stack and Storage
We start on line **6** with a jump to the **main** function located on line **83**. The first couple of lines make some space on the stack and setup a frame pointer, **EBP** which will be used throughout the assembly code to reference stored data. This will be important because, once we find the value or address of something, we need a way to reference it later.

## Find the Base Address of Kernel32
On line **86** we call the **find_kernel32** function to find the base address of **kernel32.dll** in memory. If you have not read the previous blog posts and would like to understand how the **find_kernel32** function works, check out [this](https://blog.xenoscr.net/Locating-Kernel32-Base-Address/) blog entry. Once the base address is located it is stored in the **EDX** register on line **87** for safe keeping.

## Get the Location of the Encoded Function Names
On line **88** the **win32_library_hashes** is called which in turn calls **win32_library_hashes_return**. The return address is then popped into **ESI**. This section is explained in the [previous](https://blog.xenoscr.net/Encoding-Decoding-and-Storing-Function-Names/) blog post. The assembly code is simply taking advantage of the behavior of a **CALL** instruction to obtain the memory address where the hashed function names are stored.

## Some Housekeeping
Lines **91** through **93** are setting up our storage location. **ESI**, and **EDI** will be used to point to locations we need to reference. **ESI** will be used to point to the location of the hashed function names and **EDI** will be used to point to the location where we will store the resolved function addresses.

# Resolving Symbols
This is where things start to get more interesting. On line **94** the **resolve_symbols_for_dll** function is called. It is important to remember that the base address of **kernel32** is currently stored in the **EDX** register. This will be important later.

For the moment, we will focus on just this function and ignore the call to the **find_function** function. We'll reference the following code block so we will not need to scroll back-and-forth too much. 
1. The **lodsd** instruction loads the value stored at **ESI** into the **EAX** register and increments **ESI** to point to the next address. If our code were looking for more than one function **ESI** would be ready and pointing to the next encoded function name.
2. **EAX**, now containing the encoded value of our first function name and **EDX**, which contains the address of **kernel32**, are then pushed to the stack. 
3. We will skip over the call to **find_function** for now. All that is important to know is that **EAX** now contains the address of the function we are looking for after it returns. 
4. That address is then stored in the location where **EDI** points. 
5. We restore the stack and increment **EDI** to point to the next address. 
6. We then compare **ESI** and **ECX** to see if we reached the end of the list. If we have not, the function loops until the end of the list is reached.

{% highlight Nasm linenos %}
    ; ======== Function: resolve_symbols_for_dll
    resolve_symbols_for_dll:
        lodsd
        push eax
        push edx
        call find_function
        mov [edi], eax
        add esp, 0x08
        add edi, 0x04
        cmp esi, ecx
        jne resolve_symbols_for_dll
    resolve_symbols_for_dll_finished:
        ret
{% endhighlight %}

## Finding the Function Addresses
This is the part I know you have been waiting for. How are we finding the Win32 API function Addresses? Once again, we'll refer to the following code blocks to avoid unnecessary scrolling.

### Locating the Exported Function Names
{% highlight Nasm linenos %}
    ; ======= Function: find_function
    find_function:
        pushad
        mov ebp, [esp+0x24]
        mov eax, [ebp+0x3C]
        mov edx, [ebp+eax+0x78]
        add edx, ebp
        mov ecx, [edx+0x18]
        mov ebx, [edx+0x20]
        add ebx, ebp
{% endhighlight %}

1. The first thing this function does is use a **PUSHAD** instruction to store all of the current registers so that they can be restored before the function returns. The **PUSHAD** command places the registers on the stack in the following order (top-down): **EAX, ECX, EDX, EBX, Original ESP, EBP, ESI, and EDI**

2. The value located at **ESP** + **0x24** (36 bytes) is then moved to the **EBP** register. Recall that prior to calling the **find_function** function that **EAX** (the hash value) and **EDX** (the base address of kernel32) were pushed to the stack, where **ESP** used to point prior to the **PUSHAD** instruction. What this means is the **EBP** now contains the base address of **Kernel32**.

3. The next move instruction loads the value located at an offset of **0x3C** from the base address of **Kernel32** into the **EAX** register. According to the [PE Structure](https://en.wikipedia.org/wiki/Portable_Executable#/media/File:Portable_Executable_32_bit_Structure_in_SVG_fixed.svg) diagram, the base address of the **PE Header** is located at an offset of **0x3C** from the base address of a PE file. This means that **EAX** now contains the offset from the base of **Kernel32** to the address of the **PE Header**.

4. The next move instruction adds the **EBP** (Kernel32's Base Address) to **EAX** (PE Header offset) and **0x78** and stores the value located there in the **EDX** register. Looking back at the [PE Structure](https://en.wikipedia.org/wiki/Portable_Executable#/media/File:Portable_Executable_32_bit_Structure_in_SVG_fixed.svg) diagram, the offset value of the **ExportTable** structure is located at that location. This means that **EDX** now contains the value of the offset of the **Export Table** from the base of **Kernel32**.

5. **EBP** (Kernel32's Base Address) is added to **EDX** (The export table's offset) so that **EDX** now points to the **Export Table** structure. The **Export Table**'s structure can be seen in **Figure 1**.

{% highlight c %}
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
<p align="center">
    <em>Figure 1: Export Table Structure</em>
</p>

{:start="6"}
6. **EDX** + **0x18** is stored in **ECX**. If we refer to the structure of the **Export Table** in **Figure 1**, we can see that **NuberOfNames** is located there. **ECX** now holds the number of exported functions.

7. **EDX** + **0x20** is stored in **EBX**. Again, referring to the structure of the **Export Table** in **Figure 1**, we can see that this offset points to a list of function names. **EBX** now contains the offset from the base of **Kernel32** containing a list of exported function names.

8. **EBP** is added to **EDX**, which now points to the list of exported function names.

### Iterating the Function Names to Find a Match
The next section of code will iterate the list of function names that is now stored in the **EBX** register hashing them one at a time and comparing them to the hashed value that is stored in our assembly code.

{% highlight Nasm linenos %}
    find_function_loop:
        jecxz find_function_finished
        dec ecx
        mov esi, [ebx+ecx*4]
        add esi, ebp
        
    compute_hash:
        xor edi, edi
        xor eax, eax
        cld
    compute_hash_again:
        lodsb
        test al, al
        jz compute_hash_finished
        ror edi, 0x0D
        add edi, eax
        jmp compute_hash_again
    compute_hash_finished:
    find_function_compare:
        cmp edi, [esp+0x28]
        jnz find_function_loop
        mov ebx, [edx+0x24]
        add ebx, ebp
        mov cx, [ebx+2*ecx]
        mov ebx, [edx+0x1C]
        add ebx, ebp
        mov eax, [ebx+4*ecx]
        add eax, ebp
        mov [esp+0x1C], eax
    find_function_finished:
        popad
        ret
{% endhighlight %}

1. There is first a check to see if **ECX** is zero. If **ECX** has reached zero the function will finish by restoring the registers and returning.

2. **ECX** is decreased by one, then the value pointed to by **EBX** (The list of function names) + **ECX** (Number of exported names) * **4** is moved into **ESI**. We are looking backwards from the end of the list, to the beginning. **ESI** now contains the offset of the sting containing the last function name from the base of **Kernel32**.

3. **EBP** (Kernel32's Base Address) is then added to **ESI**. **ESI** now points to the address of a null terminated string variable containing the name of the last function name in the list.

4. The hash is then calculated from the function name. This process is covered in detail in the [previous](https://blog.xenoscr.net/Encoding-Decoding-and-Storing-Function-Names/) blog post. If you need to, please refer to it.

5. The computed hash stored in the **EDI** register is then compared with the value stored at **ESP** + **0x28**. This was the name of the hash that was pushed to the stack prior to calling the **find_function** function.

6. If the values do not match, the loop moves to the next function name. If a match is found it continues on.

7. The next few steps can be a bit hard to follow, we will refer to **Figure 2** to help follow along with what is happening. **EDX** (Export Table's Base Address) + **0x24** is stored in **EBX**. **EBX** now contains the offset of the list of function addresses from the base of **Kernel32**.

<p align="center">
    <img src="/resources/images/2020-02-10-Locating-Win32-API-Functions/figure2.png" alt="Figure 2: Export Table Visualization">
</p>
<p align="center">
    <em>Figure 2: Export Table Visualization</em>
</p>

8. **EBP** (Kernel32's Base Address) is added to **EBX** so that **EBX** now points to a list of function ordinals. Not all functions are named and exported. The ordinal list contains the function number of only the named exported functions. See **Figure 2** for a visual.

9. **EBX** (Ordinal Base Address) is added to **ECX** (Function number) * 2, since the ordinals are twice the size of the function number, and stored in **CX**. **ECX** now contains the true function number.

10. **0x1C** is added to **EDX** (Export Table Base Address) and stored in **EBX**. **EBX** now contains the offset of the list of function addresses from the base of **Kernel32**.

11. **EBP** (Kernel32's Base Address) is added to **EBX** to make **EBX** point to the list of function addresses.

12. **ECX** (The function number) * 4 is added to **EBX** (The function address list) and stored in **EAX**. **EAX** now contains the offset from the base of **Kernel32** of the location containing the address of the desired function.

13. **EAX** (Offset to function address) is added to **EBP** (Kernel32's Base Address) so that **EAX** now contains the address of the function we are looking for.

14. The value is stored at **ESP** + **0x1C**, which contains the value of **EDI** that was pushed to the stack by the **PUSHAD** command at the beginning of the function. This stores the functions address where we can later reference it as needed.

15. The **POPAD** function restores the registers from the stack to their previous state and execution returns to the **resolve_symbols_for_dll** function.

# Now, Again But 64-Bit
I'm going to dispense with the play-by-play for the 64-Bit version of this assembly code. I will point out a few of the differences. The following assembly program will locate the address of two functions: **LoadLibraryA** and **CreateProcessA**.

{% highlight Nasm linenos %}
[SECTION .text]

BITS 64

_start:
    jmp main
    
    ; Constants
    win32_library_hashes:
        call win32_library_hashes_return
        ; LoadLibraryA      R13
        dd 0xEC0E4E8E
        ; CreateProcessA    R13 + 0x08
        dd 0x16B3FE72
    
    ; ======== Function: find_kernel32
    find_kernel32:
        push rsi
        mov rax, [gs:0x60]
        mov rax, [rax+0x18]
        mov rax, [rax+0x20]
        mov rax, [rax]
        mov rax, [rax]
        mov r11, [rax+0x20]             ; Kernel32 Base Stored in R11
        pop rsi
        ret
        
    ; ======= Function: find_function
    find_function:
        mov eax, [r11+0x3C]
        mov edx, [r11+rax+0x88]
        add rdx, r11                        ; RDX now points to the IMAGE_DATA_DIRECTORY structure
        mov ecx, [rdx+0x18]                 ; ECX = Number of named exported functions
        mov ebx, [rdx+0x20]
        add rbx, r11                        ; RBX = List of exported named functions
    find_function_loop:
        jecxz find_function_finished
        dec ecx                             ; Going backwards
        lea rsi, [rbx+rcx*4]                ; Point RSI at offset value of the next function name
        mov esi, [rsi]                      ; Put the offset value into ESI
        add rsi, r11                        ; RSI now points to the exported function name
        
    compute_hash:
        xor edi, edi                        ; Zero EDI
        xor eax, eax                        ; Zero EAX
        cld                                 ; Reset direction flag
    compute_hash_again:
        mov al, [rsi]                       ; Place the first character from the function name into AL
        inc rsi                             ; Point RSI to the next character of the function name
        test al, al                         ; Test to see if the NULL terminator has been reached
        jz compute_hash_finished
        ror edi, 0x0D                       ; Rotate the bits of EDI right 13 bits
        add edi, eax                        ; Add EAX to EDI
        jmp compute_hash_again
    compute_hash_finished:
    find_function_compare:
        cmp edi, r12d                       ; Compare the calculated hash to the stored hash
        jnz find_function_loop
        mov ebx, [rdx+0x24]                 ; EBX contains the offset to the AddressNameOrdinals list
        add rbx, r11                        ; RBX points to the AddressNameOrdinals list
        mov cx, [rbx+2*rcx]                 ; CX contains the function number matching the current function 
        mov ebx, [rdx+0x1C]                 ; EBX contains the offset to the AddressOfNames list
        add rbx, r11                        ; RBX points tot he AddressOfNames List
        mov eax, [rbx+4*rcx]                ; EAX contains the offset of the desired function address
        add rax, r11                        ; RAX contains the address of the desired function
    find_function_finished:
        ret
        
    ; ======== Function: resolve_symbols_for_dll
    resolve_symbols_for_dll:
        mov r12d, [r8d]                     ; Move the next function hash into R12
        add r8, 0x04                        ; Point R8 to the next function hash
        call find_function
        mov [r15], rax                      ; Store the resolved function address
        add r15, 0x08                       ; Point to the next free space
        cmp r9, r8                          ; Check to see if the end of the hash list was reached
        jne resolve_symbols_for_dll
    resolve_symbols_for_dll_finished:
        ret
        
    main:
        sub rsp, 0x110                      ; Allocate space on stack for function addresses
        mov rbp, rsp                        ; Set ebp as frame ptr for relative offset on stack
        call find_kernel32                  ; Find base address of kernel32.dll
        jmp  win32_library_hashes
        win32_library_hashes_return:
        pop r8                              ; R8 is the hash list location
        mov r9, r8
        add r9, 0x08                        ; R9 marks the end of the hash list
        lea r15, [rbp+0x10]                 ; This will be a working address used to store our function addresses
        mov r13, r15                        ; R13 will be used to reference the stored function addresses
        call resolve_symbols_for_dll
        int3
{% endhighlight %}

## Missing Commands?
When operating in 64-bit with Assembly, there are a few commands that are no longer exist, or are not useful anymore. **PUSHAD** and **POPAD** do not work on 64-bit registers. This being the case, they're of no use and we needed to find an alternative. The **LODSB** command also does not work in 64-bit. It was necessary to replace it with a **MOV AL, [RSI]** and **INC RSI** sequence. It does the exact same thing, only with more bytes and instructions.

## Register Differences
64-bit Assembly has many more registers available. Registers **R8** through **R15** have been added. That gives us  **8** new places to store things that we need. With this additional storage it was fairly easy to compensate for the missing **PUSHAD** and **POPAD** commands. There are some consideration however. To do anything with the function addresses we find now later, we will need to call the functions. 64-Bit **stdcall** function calls are a bit different according to the [documentation](https://docs.microsoft.com/en-us/cpp/build/x64-calling-convention?view=vs-2019). According to the section on [Parameter passing](https://docs.microsoft.com/en-us/cpp/build/x64-calling-convention?view=vs-2019#parameter-passing), the **R8**, and **R9** registers are used to pass the 3rd and 4th parameters to a function. We should avoid storing anything long-term in them. The same goes for the **RCX** and **RDX** parameters.

## PE File Offset Differences
I was able to skip finding the DOS header offset at **0x24** and skip straight to finding the **PE Header** offset value at offset **0x3C**. From there, finding the offset of the **_IMAGE_EXPORT_DIRECTORY** was a little different. Examining the table dump of the **_IMAGE_DOS_HEADER** and **_IMAGE_NT_HEADERS64** from **WinDbg** in **Figure 3**, we can see that the layout is a bit different. The **_IMAGE_DATA_DIRECTORY** is located at an offset of **0x70** from the **OptionalHeader** at offset **0x18**. That means that the total offset needs to be **0x88**, not **0x78** as it was in 32-Bit mode. 

{% highlight text %}
0:001> dt _IMAGE_DOS_HEADER 77260000
ntdll!_IMAGE_DOS_HEADER
   +0x000 e_magic          : 0x5a4d
   +0x002 e_cblp           : 0x90
   +0x004 e_cp             : 3
   +0x006 e_crlc           : 0
   +0x008 e_cparhdr        : 4
   +0x00a e_minalloc       : 0
   +0x00c e_maxalloc       : 0xffff
   +0x00e e_ss             : 0
   +0x010 e_sp             : 0xb8
   +0x012 e_csum           : 0
   +0x014 e_ip             : 0
   +0x016 e_cs             : 0
   +0x018 e_lfarlc         : 0x40
   +0x01a e_ovno           : 0
   +0x01c e_res            : [4] 0
   +0x024 e_oemid          : 0
   +0x026 e_oeminfo        : 0
   +0x028 e_res2           : [10] 0
   +0x03c e_lfanew         : 0n224
0:001> dt -r _IMAGE_NT_HEADERS64 772600eC
ntdll!_IMAGE_NT_HEADERS64
   +0x000 Signature        : 0
   +0x004 FileHeader       : _IMAGE_FILE_HEADER
      +0x000 Machine          : 0
      +0x002 NumberOfSections : 0
      +0x004 TimeDateStamp    : 0x202200f0
      +0x008 PointerToSymbolTable : 0x9020b
      +0x00c NumberOfSymbols  : 0x9b000
      +0x010 SizeOfOptionalHeader : 0x1000
      +0x012 Characteristics  : 8
   +0x018 OptionalHeader   : _IMAGE_OPTIONAL_HEADER64
      +0x000 Magic            : 0
      +0x002 MajorLinkerVersion : 0 ''
      +0x003 MinorLinkerVersion : 0 ''
      +0x004 SizeOfCode       : 0x15340
      +0x008 SizeOfInitializedData : 0x1000
      +0x00c SizeOfUninitializedData : 0x77260000
      +0x010 AddressOfEntryPoint : 0
      +0x014 BaseOfCode       : 0x1000
      +0x018 ImageBase        : 0x00010006`00000200
      +0x020 SectionAlignment : 0x10006
      +0x024 FileAlignment    : 0x10006
      +0x028 MajorOperatingSystemVersion : 0
      +0x02a MinorOperatingSystemVersion : 0
      +0x02c MajorImageVersion : 0xf000
      +0x02e MinorImageVersion : 0x11
      +0x030 MajorSubsystemVersion : 0x400
      +0x032 MinorSubsystemVersion : 0
      +0x034 Win32VersionValue : 0x122d97
      +0x038 SizeOfImage      : 0x1400003
      +0x03c SizeOfHeaders    : 0x40000
      +0x040 CheckSum         : 0
      +0x044 Subsystem        : 0x1000
      +0x046 DllCharacteristics : 0
      +0x048 SizeOfStackReserve : 0x00100000`00000000
      +0x050 SizeOfStackCommit : 0x00001000`00000000
      +0x058 SizeOfHeapReserve : 0
      +0x060 SizeOfHeapCommit : 0x0009fffc`00000010
      +0x068 LoaderFlags      : 0xad25
      +0x06c NumberOfRvaAndSizes : 0xf8bbc
      +0x070 DataDirectory    : [16] _IMAGE_DATA_DIRECTORY
         +0x000 VirtualAddress   : 0x1f4
         +0x004 Size             : 0x116000
{% endhighlight %}
<p align="center">
    <em>Figure 3: Finding the Export Table in WinDbg</em>
</p>

# Conclusion
It is all clear as mud, right? It's OK if it is, it took me a while and a lot of reading to understand what was going on. Spend some time reading about the **PE Header** and find whatever information you can on the various structures. Run the code in debuggers and see what is happening. **WinDbg** is excellent for getting a better understanding of the structures. The following, in no particular order, are some links I collected along the way while I was attempting to understand what the above assembly codes do. I hope they will help you on your journey of learning:
 
 - [https://docs.microsoft.com/en-us/previous-versions/ms809762(v=msdn.10)?redirectedfrom=MSDN](https://docs.microsoft.com/en-us/previous-versions/ms809762(v=msdn.10)?redirectedfrom=MSDN)
 - [https://docs.microsoft.com/en-us/windows/win32/api/winnt/ns-winnt-image_optional_header32](http://www.osdever.net/documents/PECOFF.pdf)
 - [https://docs.microsoft.com/en-us/windows/win32/api/winnt/ns-winnt-image_optional_header32](https://docs.microsoft.com/en-us/windows/win32/api/winnt/ns-winnt-image_optional_header32)
 - [https://en.wikipedia.org/wiki/Portable_Executable](https://en.wikipedia.org/wiki/Portable_Executable)
 - [https://docs.microsoft.com/en-us/windows/win32/debug/pe-format#export-directory-table](https://docs.microsoft.com/en-us/windows/win32/debug/pe-format#export-directory-table)
 - [https://www.aldeid.com/wiki/PE-Portable-executable](https://www.aldeid.com/wiki/PE-Portable-executable)
 - [https://resources.infosecinstitute.com/the-export-directory/](https://resources.infosecinstitute.com/the-export-directory/)
 
