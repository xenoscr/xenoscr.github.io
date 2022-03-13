---
layout: post
title: Implementing Syscalls in Cobalt Strike Part 1 - Battling Imports and Dependencies
---

I was recently working to implement Syscalls in Cobalt Strike's Artifact Kit. When I started looking, [Syswhispers2](https://github.com/jthuraisamy/SysWhispers2) only had x64 support. I did some searching and found that there were a couple of forks that supported x86, including this [pull request](https://github.com/jthuraisamy/SysWhispers2/pull/9) that has been waiting since October of 2021. My first thought was: "This is very cool, why is this still not committed?" Then I thought about how difficult is is for me to find any time to work on my own side work, blogs, and try to contribute when I can to other efforts. It's completely understandable. At any rate, I had something to start with, I pulled down [bugproof](https://github.com/bugproof/SysWhispers2)'s fork and tried to follow along with [brsn](https://br-sn.github.io/Implementing-Syscalls-In-The-CobaltStrike-Artifact-Kit/)'s blog post that was featured in Raphael Mudge's video: [Using Direct Syscalls in Cobalt Strike's Artifact Kit](https://www.youtube.com/watch?v=mZyMs2PP38w). This two part blog series will document my journey and the reasons why I ended up forking Syswhispers2 and eventually making my own [pull request](https://github.com/jthuraisamy/SysWhispers2/pull/18).

This blog post will document the first part of my journey, specifically some successes and failures that lead me to choose my final solution. Which was to fork Syswhispers2 and edit it to include **x86**, **x64**, and **Nasm** assembler support as well as abandoning Visual Studio. While it is possible to work with the limitations of the binaries produced by Visual Studio, it was not the solution that I preferred. I'm including as much detail as I can so that others can learn from it and potentially improve upon it where I was unable to.

**NOTE**: You can use the code from [brsn](https://br-sn.github.io/Implementing-Syscalls-In-The-CobaltStrike-Artifact-Kit/)'s blog post as a stand-in for the Cobalt Strike Artifact Kit code, since I do not wish to violate any rules. The observations I will be making can be duplicated using the example code. If you have access to the official code Artifact Kit, you can fully follow along. Either way, there is something to be learned about compiler and linker behavior of Visual Studio Vs. MinGW.

# Learning by Imitating

They say that "Imitation is the most sincere form of flattery." The reality is that we all learn by imitating what others do. How we expand is upon what we imitate is innovation. I started this journey as we all do, by trying to duplicate what someone else has already done. In this case, i started by trying to duplicate what I read in brsn's blog post. What made what I saw so attractive was the ability to use Visual Studio's IDE. I was so excited, I threw the whole Artifact Kit into Visual Studio right away and started adding source files and headers and managed to successfully duplicate the functionality of the included **build.sh** script. Since the resulting Visual Studio Solution now contains the Artifact Kit code, I cannot share my solution. I can however share a few notes and tips I came up with while working to port the kit to Visual Studio.

# Project Settings

## Preprocessor Definitions

First, as I was attempting to follow along with brsn's post, I noted a few opportunities to improve. First, in the **build.sh** script there are two preprocessor definition values defined via **-D** flag values, which are:

- DATA_SIZE
- \_MIGRATE\_

The instructions in brsn's blog would have you define only the **DATA_SIZE** value via the source. He's a bit vague but I'm assuming he defined it in the **patch.h** header. In Visual Studio, you can define this value under: **Configuration Properties > C/C++ > Preprocessor > Preprocessor Definitions**. **Figure 01** and **Figure 02** demonstrate the settings for staged payload and a service payload, which leverages injection, respecively.

![Figure 01: Preprocessor Definitions for a staged payload](/resources/images/2022-03-12-Battling-Imports-and-Dependencies/image-20220312182017229.png)
{: .figure}

Figure 01: Preprocessor Definitions for a staged payload
{: .label}

![Figure 02: Preprocessor Definitions for a service payload using the \_MIGRATE\_ definition to satisfy the #ifdef \_MIGRATE\_ compiler directive](/resources/images/2022-03-12-Battling-Imports-and-Dependencies/image-20220312182106786.png)
{: .figure}

Figure 02: Preprocessor Definitions for a service payload using the \_MIGRATE\_ definition to satisfy the #ifdef \_MIGRATE\_ compiler directive
{: .label}

## Entry Points

The next hurdle to duplicate the functionality of the **build.sh** script was to support the compilation of a **DLL** as well as **EXE** files. Brsn's blog discusses setting the entry point under **Configuration Properties > Linker > Advanced > Entry Point** to **mainCRTStartup** for and **EXE** file. According to the documentation I found [here](https://docs.microsoft.com/en-us/cpp/build/reference/entry-entry-point-symbol?view=msvc-170), the correct setting for a **DLL** is **_DllMainCRTStartup**. **Figure 03** demonstrates the correct setting for a **DLL**.

![Figure 03: Entry Point setting for a DLL](/resources/images/2022-03-12-Battling-Imports-and-Dependencies/image-20220312194717849.png)
{: .figure}

Figure 03: Entry Point setting for a DLL
{: .label}

Additionally, you will need to set **Configuration Properties > General > Configuration Type** to **Dynamic Library (.dll)** to compile a **DLL**.

## Build Profiles and Source Exclusions

To support compiling all the necessary binaries to duplicate the functionality of the included **build.sh** script, you will need to setup **6** build profiles, each with **x86** and **x64** versions. In total there are **12** binaries that you need to produce to duplicate what **build.sh** does. Since I cannot provide the source, I will provide breakdown of the configuration settings as well as source exclusions to make to avoid compiler errors.

**NOTE**: There are no debugging profiles. You can delete them, they're not needed for the final solution.

### All Projects

| Configuration Setting                                        | Value                      | Example                  |
| ------------------------------------------------------------ | -------------------------- | ------------------------ |
| Configuration Properties > General > Output Directory        | $(SolutionDir)\<dist-name\>\\ | $(SolutionDir)dist-peek\\ |
| Configuration Properties > Linker > Debugging > Strip Private Symbols | /PDBSTRIPPED               | /PDBSTRIPPED             |
{:.centered-table}

Table 01: Configurations that apply to all projects.
{:.label}

### Staged EXE

| Configuration Setting                                        | Value                                      | Example                                    |
| ------------------------------------------------------------ | ------------------------------------------ | ------------------------------------------ |
| Configuration Properties > General >Target Name              | $(ProjectName)$(PlatformArchitecture)      | $(ProjectName)$(PlatformArchitecture)      |
| Configuration Properties > General > Configuration Type      | Application (.exe)                         | Application (.exe)                         |
| Configuration Properties > C/C++ > Preprocessor > Preprocessor Definitions | DATA_SIZE=1024;\_CRT\_SECURE\_NO\_WARNINGS | DATA_SIZE=1024;\_CRT\_SECURE\_NO\_WARNINGS |
| Configuration Properties > Linker > System > SubSystem       | Windows (/SUBSYSTEM:WINDOWS)               | Windows (/SUBSYSTEM:WINDOWS)               |
| Configuration Properties > Linker > Advanced > Entry Point   | mainCRTStartup                             | mainCRTStartup                             |
{:.centered-table}

Table 02: Staged EXE Configuration Settings
{:.label}

Files to include:

- main.c
- patch.c
- patch.h
- bypass-\<bypass_name\>.c (Where \<bypass_name\> is the name of the bypass you're using.)

Files to exclude:

- dllmain.c
- dllmain.def
- svcmain.c
- start_thread.c
- injector.c

### Full EXE

| Configuration Setting                                        | Value                                        | Example                                      |
| ------------------------------------------------------------ | -------------------------------------------- | -------------------------------------------- |
| Configuration Properties > General >Target Name              | $(ProjectName)$(PlatformArchitecture)big     | $(ProjectName)$(PlatformArchitecture)big     |
| Configuration Properties > General > Configuration Type      | Application (.exe)                           | Application (.exe)                           |
| Configuration Properties > C/C++ > Preprocessor > Preprocessor Definitions | DATA_SIZE=271360;\_CRT\_SECURE\_NO\_WARNINGS | DATA_SIZE=271360;\_CRT\_SECURE\_NO\_WARNINGS |
| Configuration Properties > Linker > System > SubSystem       | Windows (/SUBSYSTEM:WINDOWS)                 | Windows (/SUBSYSTEM:WINDOWS)                 |
| Configuration Properties > Linker > Advanced > Entry Point   | mainCRTStartup                               | mainCRTStartup                               |
{:.centered-table}

Table 03: Full EXE Configuration Settings
{:.label}

Files to include:

- main.c
- patch.c
- patch.h
- bypass-\<bypass_name\>.c (Where \<bypass_name\> is the name of the bypass you're using.)

Files to exclude:

- dllmain.c
- dllmain.def
- svcmain.c
- start_thread.c
- injector.c

### Staged Service EXE

| Configuration Setting                                        | Value                                      | Example                                    |
| ------------------------------------------------------------ | ------------------------------------------ | ------------------------------------------ |
| Configuration Properties > General >Target Name              | $(ProjectName)$(PlatformArchitecture)svc   | $(ProjectName)$(PlatformArchitecture)svc   |
| Configuration Properties > General > Configuration Type      | Application (.exe)                         | Application (.exe)                         |
| Configuration Properties > C/C++ > Preprocessor > Preprocessor Definitions | DATA_SIZE=1024;\_CRT\_SECURE\_NO\_WARNINGS | DATA_SIZE=1024;\_CRT\_SECURE\_NO\_WARNINGS |
| Configuration Properties > Linker > System > SubSystem       | Windows (/SUBSYSTEM:WINDOWS)               | Windows (/SUBSYSTEM:WINDOWS)               |
| Configuration Properties > Linker > Advanced > Entry Point   | mainCRTStartup                             | mainCRTStartup                             |
{:.centered-table}

Table 04: Staged Service EXE Configuration Settings
{:.label}

Files to include:

- svcmain.c
- patch.c
- patch.h
- bypass-\<bypass_name\>.c (Where \<bypass_name\> is the name of the bypass you're using.)

Files to exclude:

- dllmain.c
- dllmain.def
- main.c
- start_thread.c
- injector.c

### Full Service EXE

| Configuration Setting                                        | Value                                        | Example                                      |
| ------------------------------------------------------------ | -------------------------------------------- | -------------------------------------------- |
| Configuration Properties > General >Target Name              | $(ProjectName)$(PlatformArchitecture)svcbig  | $(ProjectName)$(PlatformArchitecture)svcbig  |
| Configuration Properties > General > Configuration Type      | Application (.exe)                           | Application (.exe)                           |
| Configuration Properties > C/C++ > Preprocessor > Preprocessor Definitions | DATA_SIZE=271360;\_CRT\_SECURE\_NO\_WARNINGS | DATA_SIZE=271360;\_CRT\_SECURE\_NO\_WARNINGS |
| Configuration Properties > Linker > System > SubSystem       | Windows (/SUBSYSTEM:WINDOWS)                 | Windows (/SUBSYSTEM:WINDOWS)                 |
| Configuration Properties > Linker > Advanced > Entry Point   | mainCRTStartup                               | mainCRTStartup                               |
{:.centered-table}

Table 05: Full Service EXE Configuration Settings
{:.label}

Files to include:

- svcmain.c
- patch.c
- patch.h
- bypass-\<bypass_name\>.c (Where \<bypass_name\> is the name of the bypass you're using.)

Files to exclude:

- dllmain.c
- dllmain.def
- main.c
- start_thread.c
- injector.c

### Staged DLL All

| Configuration Setting                                        | Value                                      | Example                                    |
| ------------------------------------------------------------ | ------------------------------------------ | ------------------------------------------ |
| Configuration Properties > General > Configuration Type      | Dynamic Library (.dll)                     | Dynamic Library (.dll)                     |
| Configuration Properties > C/C++ > Preprocessor > Preprocessor Definitions | DATA_SIZE=1024;\_CRT\_SECURE\_NO\_WARNINGS | DATA_SIZE=1024;\_CRT\_SECURE\_NO\_WARNINGS |
| Configuration Properties > Linker > System > SubSystem       | Windows (/SUBSYSTEM:WINDOWS)               | Windows (/SUBSYSTEM:WINDOWS)               |
| Configuration Properties > Linker > Advanced > Entry Point   | _DllMainCRTStartup                         | _DllMainCRTStartup                         |
{:.centered-table}

Table 06: Staged DLL Configuration Settings For All
{:.label}

#### Staged DLL x86

| Configuration Setting                           | Value                                 | Example                               |
| ----------------------------------------------- | ------------------------------------- | ------------------------------------- |
| Configuration Properties > General >Target Name | $(ProjectName)$(PlatformArchitecture) | $(ProjectName)$(PlatformArchitecture) |
{:.centered-table}

Table 07: Staged x86 DLL Configuration Settings
{:.label}

#### Staged DLL x64

| Configuration Setting                           | Value                                             | Example                                           |
| ----------------------------------------------- | ------------------------------------------------- | ------------------------------------------------- |
| Configuration Properties > General >Target Name | $(ProjectName)$(PlatformArchitecture).$(Platform) | $(ProjectName)$(PlatformArchitecture).$(Platform) |
{:.centered-table}

Table 08: Staged x64 DLL Configuration Settings
{:.label}

Files to include:

- dllmain.c
- dllmain.def
- patch.c
- patch.h
- bypass-\<bypass_name\>.c (Where \<bypass_name\> is the name of the bypass you're using.)

Files to exclude:

- main.c
- svcmain.c
- start_thread.c
- injector.c

### Full DLL All

| Configuration Setting                                        | Value                                        | Example                                      |
| ------------------------------------------------------------ | -------------------------------------------- | -------------------------------------------- |
| Configuration Properties > General > Configuration Type      | Dynamic Library (.dll)                       | Dynamic Library (.dll)                       |
| Configuration Properties > C/C++ > Preprocessor > Preprocessor Definitions | DATA_SIZE=271360;\_CRT\_SECURE\_NO\_WARNINGS | DATA_SIZE=271360;\_CRT\_SECURE\_NO\_WARNINGS |
| Configuration Properties > Linker > System > SubSystem       | Windows (/SUBSYSTEM:WINDOWS)                 | Windows (/SUBSYSTEM:WINDOWS)                 |
| Configuration Properties > Linker > Advanced > Entry Point   | _DllMainCRTStartup                           | _DllMainCRTStartup                           |
{:.centered-table}

Table 09: Full DLL Configuration Settings for All
{:.label}

#### Full DLL x86

| Configuration Setting                           | Value                                    | Example                                  |
| ----------------------------------------------- | ---------------------------------------- | ---------------------------------------- |
| Configuration Properties > General >Target Name | $(ProjectName)$(PlatformArchitecture)big | $(ProjectName)$(PlatformArchitecture)big |
{:.centered-table}

Table 10: Full x86 DLL Configuration Settings
{:.label}

#### Full DLL x64

| Configuration Setting                           | Value                                                | Example                                              |
| ----------------------------------------------- | ---------------------------------------------------- | ---------------------------------------------------- |
| Configuration Properties > General >Target Name | $(ProjectName)$(PlatformArchitecture)big.$(Platform) | $(ProjectName)$(PlatformArchitecture)big.$(Platform) |
{:.centered-table}

Table 11: Full x64 DLL Configuration Settings
{:.label}

Files to include:

- dllmain.c
- dllmain.def
- patch.c
- patch.h
- bypass-\<bypass_name\>.c (Where \<bypass_name\> is the name of the bypass you're using.)

Files to exclude:

- main.c
- svcmain.c
- start_thread.c
- injector.c

# We Have Binaries, Now What?

If you were successful in getting everything added, the settings correct, everything should build. You can do a **Batch Build** and build all **12** binaries. If all has gone well you should have:

- artifact32.dll
- artifact32.exe
- artifact32big.dll
- artifact32big.exe
- artifact32svc.exe
- artifact32svcbig.exe
- artifact64.exe
- artifact64.x64.dll
- artifact64big.exe
- artifact64big.x64.dll
- artifact64svc.exe
- artifact64svcbig.exe

Let's take a look at what we have, at first glance, everything looks fine as you can see in **Figure 04**. There are **12** binaries and the file sizes look right. If you test them out, they'll do the job... if the right conditions are met.

![Figure 04: Binaries compiled with the above settings. Everything looks good... right?](/resources/images/2022-03-12-Battling-Imports-and-Dependencies/image-20220312194853488.png)
{: .figure}

Figure 04: Binaries compiled with the above settings. Everything looks good... right?
{: .label}

# Take a Closer Look

I wanted to compare the resulting binaries to some that were compiled the traditional way using **MinGW** from a Linux command line. The first tool that I used was **dumpbin.exe** from the Visual Studio Developer Command Prompt. As you can see in **Figure 05**, there are some extra imports that I wasn't pleased to see:

- Kernel32.dll
- User32.dll
- VCRUNTIME140.dll
- api-ms-win-crt-heap-l1-1-0.dll
- api-ms-win-crt-runtime-l1-1-0.dll
- api-ms-win-crt-math-l1-1-0.dll
- api-ms-win-crt-stdio-l1-1-0.dll
- api-ms-win-crt-locale-l1-1-0.dll

![Figure 05: Output of "dumpbin.exe /imports" ran against an EXE compiled by Visual Studio](/resources/images/2022-03-12-Battling-Imports-and-Dependencies/image-20220312195005605.png)
{: .figure}

Figure 05: Output of "dumpbin.exe /imports" ran against an EXE compiled by Visual Studio
{: .label}

Wow! That's a lot of garbage that I don't want risk potentially not being present on my target system. **MingGW** doesn't seem to have the same behavior and links to **msvcrt.dll** alone to meet all of the same requirements as the massive list of DLLs Visual Studio seems to want. The same source compiled with **MinGW** only uses imports from: **KERNEL32.DLL**, **USER32.DLL**, and **msvcrt.dll**. I did some few searches and it seems that Visual Studio does not support linking to **msvcrt.dll**. After a bit more searching and digging in the crevices of my mind I remembered that there is an option that can be changed to statically link DLL files. This doesn't exactly fix the problem completely as I will demonstrate but it does improve things. To statically link your binary, you will need to change the following setting:

| Configuration Setting                                        | Value                |
| ------------------------------------------------------------ | -------------------- |
| Configuration Properties > C/C++ > Code Generation > Runtime Library | Multi-threaded (/MT) |
{:.centered-table}

Table 12: Changing Linking Behavior to Statically Link DLLs
{:.label}

Recompiling with the above setting changed, the imports have been reduced to only require **KERNEL32.DLL** and **USER32.DLL**. That's better, right? The answer is no, with the everything statically linked the size of our binaries have increased by nearly 60k for the staged payloads and 100k for un-staged payloads, as you can see in **Figure 06**. This isn't at all what we want either. 

![Figure 06: Things look a little bloated with statically linked libraries](/resources/images/2022-03-12-Battling-Imports-and-Dependencies/image-20220312195103195.png)
{: .figure}

Figure 06: Things look a little bloated with statically linked libraries
{: .label}

# Conclusion

That's it for this blog post. I hope that by documenting my process and my observations others can learn from my experiences. As I said at the beginning, using Visual Studio will absolutely work. Your mileage may vary depending on the target environment and your payload requirements. If your target is missing some dependencies or your target application cannot handle a larger payload, using Visual Studio may not be the best option for your needs. For my needs, I was not willing to settle and chose to pursue an option that relied on only **MinGW** and **Nasm** which I will document in **Part 2** of this series.
