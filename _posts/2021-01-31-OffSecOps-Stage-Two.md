---
layout: post
title: "OffSecOps Stage Two"
---
In my previous [blog post](https://blog.xenoscr.net/OffSecOps-Basic-Setup/), I demonstrated how to setup and configure a basic Jenkins Pipeline. Now, this post will demonstrate how to turn that basic DevOps pipeline into a basic OffSecOps pipeline. To take the pipeline from DevOps to OffSecOps, it will need to be capable of targeting specific .NET Frameworks, CPU types, and obfuscate the assemblies. I have chosen to develop this capability for C# projects in this blog post. The same techniques can be applied to other languages and compilers with a little more work and testing. I will leave that exercise to you, the reader.

# Visual Studio Solution & Project Manipulation
In [Will Schroeder's SO-CON talk](https://specterops.io/so-con2020/event-758918), he demonstrated a pipeline that was capable of:
- Compile Visual Studio C# Solutions 
- Targeting specific .NET Framework Versions 
- Obfuscating the Compiled Binary 
- Testing for OPSEC issues 

<p align="center">
    <img src="/resources/images/2021-01-31-OffSecOps-Stage-Two/figure01.png" alt="Figure 1: A slide from Will Schroeder's OffSecOps talk at SO-CON 2020">
</p>
<p align="center">
    <em>Figure 1: A slide from Will Schroeder's OffSecOps talk at SO-CON 2020</em>
</p>

At this stage, I will give the OPSEC checks a pass, and focus on the ability to compile and obfuscate my tools. To mimic what seems to be taking place and to add some additional capabilities, I wrote a new PowerShell module. The new module, **Update-Vsproject.psm1**, is part of a set of PowerShell modules that I needed to write to duplicate the functionality that it appears that SpecterOps' solution has. All of the modules and other scripts I have use can be downloaded from the following GitHub repository: 

 - [OffSecOps Pipeline Scripts](https://github.com/xenoscr/offsecops-shared-library)
 
This module has several functions but there is only need to call one to make the necessary modifications to a C# visual studio project. I will eventually expand this script to support other languages, other than C#. The functions in this module are:

| Function Name | Description |
| --- | --- |
| Update-CSProjects | Update a C# solution and project files to include the specified platform configuration |
| Update-CSProjDotNetVer | Update the target .NET Framework of C# projects. |
| Update-CSProjPlatform | Update C# project files to include the specified platform configuration. |
| Write-PrettyXml | Print formatted XML to the console. |
| Add-SlnConfig | Update Visual Studio Solution file to include the specified platform configuration. |
{:.datatable}

The **Update-CSProjects** function is the main function in this module, it will call the necessary functions needed to re-target a project. It will update the solution and project files to include a new CPU target and, if specified, modify the .NET Framework in all project files. It's not perfect, there are some aesthetic issues, but it gets the job done.  For example:

## Retarget the CPU Platform
{% highlight powershell %}
Update-CSProjects -slnPath "C:\foo\foo.sln" -TargetPlatform "x86"
{% endhighlight %}

## Retarget the CPU Platform and .NET Version
{% highlight powershell %}
Update-CSProjects -slnPath "C:\foo\foo.sln" -TargetPlatform "x86" -TargetFramework "3.5"
{% endhighlight %}

# Building Projects & Solutions
After the solution and project files have been manipulated, or if you choose to use them as they are, it's time to build the solution(s). To help out with this portion of the OffSecOps pipeline, I created another PowerShell module, called **Compile-CSProject**. As I add functionality for other languages, I may make other modules or modify and change this module's name to something more generic. The functions available in this module are:

| Function Name | Description |
| --- | --- |
| Invoke-CompileCSProject | Compile a target C# solution. |
| Invoke-CSProjectCleanup | Clean a target C# solution. |
| Invoke-DevEnvironment | Invokes a command and imports its environment variables. |
| Invoke-MSBuildEnv | Invokes the appropriate VsMsBuildCmd.bat and duplicates the environment variables. |
| Get-MSBuildPath | Returns the path of the appropriate MSBuild.exe to use based on the target language version. |
| Get-CSProjFrameworkVer | Get the .NET Framework version from a visual studio project. |
| Get-CSProjLangVer | Get the language version from a visual studio project. |
{:.datatable}

The **Invoke-CompileCSProject** function is the primary function to use. It will call the necessary functions to invoke the appropriate build environment for the targeted solution, then build the projects. If necessary, you can use the **Update-CSProjects** function to add a platform first. Once your solution has the desired targets, you can compile it. The following are examples of how **Invoke-CompileCSProject** can be used:

## Build a Project Using the Default .NET Framework Version
{% highlight powershell %}
Invoke-CompileCSProject –slnPath "C:\foo\foo.sln" -TargetPlatform "x86" -TargetConfiguration "Release"
{% endhighlight %}

## Build a Project Using a Specific .NET Framework Version
{% highlight powershell %}
Invoke-CompileCSProject –slnPath "C:\foo\foo.sln" -TargetPlatform "x86" -TargetConfiguration "Release" -TargetFrameworkVersion "3.5"
{% endhighlight %}

# Obfuscation (Confused?)
The final trick for this round is obfuscation. I have chosen to use the **[ConfuserEx 2](https://mkaring.github.io/ConfuserEx/)** project to obfuscate the compiled .NET Assemblies. For this task, I wrote another module called: **ConfuserEXProj**. I wrote the module to create a custom object that contains all of the current options available in **ConfuserEx 2** and to create a project file for **ConfuserEx 2**. The functions available in this module are:

| Function Name | Description |
| --- | --- |
| Get-ConfuserExProtections | Writes a ConfuserEX project file with the provided protections. |
| New-ConfuserEXProj | Returns a PSCustomObject populated with the default ConfuserEX protections settings. |
{:.datatable}

To use this module to create a **ConfuserEx 2** project file, you will need to first create a **ConfuserEx 2** protections object. Then, pass it to the **New-ConfuserEXProj** to create a new project. For example:

## Create and Setup ConfuserEx Options and Create a Project File
{% highlight powershell %}
# Setup the protections 
$protections = Get-ConfuserExProtections 
$protections.antiTamper .enabled = $true 
$protections.constants.enabled = $true 
$protections.ctrlFlow.enabled = $true 
$protections.refProxy.enabled = $true 
$protections.resources.enabled = $true 
$protections.antiILDasm.enabled = $true 
$protections.watermark.enabled = $true 
$protections.watermark.action = "remove" 
$protections.packer = $null 
 
# Create the new ConfuserEX project file 
New-ConfuserEXProj –Protections $protections –TargetAssembly "C:\foo\bin\Release\foo.exe"
{% endhighlight %}

# Putting it All Together
With the scripts, it's now possible to manipulate the Visual Studio solution and project files, build it, and obfuscate the resulting Assemblies. The next step is to integrate everything into the Jenkins pipeline that was built in the previous blog. Were we left off, the example pipeline had the following stages:

- Checkout
- Build
- Publish

After the modifications have been made to the OffSecOps pipeline, it will consist of the following stages:

- Checkout
- Modify Solution
- Assembly Clean-Up
- Compile Project
- Run ConfuserEx
- Publish

## Let's Get Groovy
Jenkins uses [Groovy](https://groovy-lang.org/) scripting, It's very similar to JavaScript but, not exactly. To expand the pipeline, and following the example from SpecterOps, I created a new Jenkins shared library. The functions in the shared library are running PowerShell scripts that load and execute the functions in the modules that I created and detailed above. The shared library is part of the repository I'm publishing with this blog post. The shared library has the following functions:

| Function Name | Description |
| --- | --- |
| modifySolution | Used by the Modify Solution stage of the Jenkins pipeline to make the necessary modifications to the Visual Studio project. |
| removeOldAssemblies | Used by the Assembly Clean-Up stage of the Jenkins pipeline to remove artifacts from previous builds. |
| compileSolution | Used by the Compile Project stage of the Jenkins pipeline to build the project. |
| confuseSolution | Used by the Run ConfuseEx stage of the Jenkins pipeline to obfuscate the compiled assemblies. |
{:.datatable}

### Adding a New Shared Library to Jenkins
There are a few ways to use a shared library from a Jenkins pipeline script. I have selected the option of using a Git library added to the Jenkins Global configuration. More information about shared libraries in Jenkins can be found here:

- [Extending with Shared Libraries (jenkins.io)](https://www.jenkins.io/doc/book/pipeline/shared-libraries/)

1. Log into Jenkins.

2. Click **Manage Jenkins**.

3. Click on **Configure System**.

4. Scroll down to the **Global Pipeline Libraries** section.

5. Click the **Add** button.

6. Complete the following fields:

   - **Name** – The name of your shared library
   - **Default version** – The Git branch that will be used, the default is: master
   - **Allow default version to be overridden** – Checked
   - **Include @Library changes in job recent changes** – Checked
   - **Modern SCM** – Selected
   - **Git** – Selected
   - **Project Repository** – The repository's URL. (I've use HTTP(S) in my PoC)
   - **Credentials** – The credentials to use if the repository is marked "Private". (API Keys do not work here)
   - **Within Repository** – Discover Branches
   
7. Click **Save** to apply the changes to Jenkins. 

After adding the library, you should now be able to address the shared library in your pipeline in the following way: 
{% highlight Groovy %}
@Library('<library-name>')_
{% endhighlight %}

### Update the Jenkins Pipeline Script
With the shared library added to Jenkins, it can now be added to a Jenkins Pipeline. I have elected to clone the pipeline from the previous blog post and modify it to use the newly added shared libraries. This example pipeline script will now do everything I have set out to do: Download the source, modify the project to add a x86 and x64 configuration, clean-up any previous build artifacts, compile the project for the targeted .NET and CPU versions, obfuscate them using ConfuserEX 2, and upload all of the assemblies to Artifactory.

The new Jenkins Pipeline will look something like this:
{% highlight Groovy linenos %}
@Library('jenkins-shared-library')_ 
def server = Artifactory.server 'offsecops-local' 
def psScriptPath = "C:\\scripts\\powershell\\" 

pipeline { 
   agent { label 'master' } 
   environment { 
   } 
   stages { 
      stage('Checkout') { 
          steps { 
              checkout([$class: 'GitSCM', 
              branches: [[name: '*/master']], 
              doGenerateSubmoduleConfigurations: false, 
              extensions: [], 
              submoduleCfg: [], 
              userRemoteConfigs: [[credentialsId: 'gitlab-jenkins', 
              url: 'http://gitlab.example.local/jenkins/Seatbelt.git']]]) 
          } 
      } 
      stage('Modify Solution') { 
          steps { 
              script { 
                  psScripts.modifySolution(psScriptPath, "all") 
              } 
          } 
      } 
      stage('Assembly Clean-up') { 
          steps { 
              script { 
                  psScripts.removeOldAssemblies() 
              } 
          } 
      } 
      stage('Compile Project') { 
          steps { 
              script { 
                  def platforms = ['x86', 'x64', 'Any CPU'] 
                  platforms.each { arch -> psScripts.compileSolution(psScriptPath, "${arch}", "3.5") } 
                  platforms.each { arch -> psScripts.compileSolution(psScriptPath, "${arch}", "4.0") } 
              } 
          } 
      } 
      stage('Run ConfuserEX') { 
          steps { 
              script { 
                  psScripts.confuseSolution(psScriptPath, "<ConfuserExCLIPath>") 
              } 
          } 
      } 
      stage('publish') { 
          steps { 
              script { 
                  rtBuildInfo() 
                  rtUpload ( 
                      serverId: "offsecops-local", 
                      spec: 
                      """{ 
                          "files": [ 
                          { 
                              "pattern": "*/bin/*/*.exe", 
                              "target": "offsecops-local/Projects/<ProjectName>/" 
                          }, 
                          { 
                              "pattern": "*/bin/*/Confused/*.exe", 
                              "target": "offsecops-local/Projects/<ProjectName>/" 
                          } 
                          ] 
                      }""" 
                  ) 
              } 
          } 
      } 
   } 
}
{% endhighlight %}

# Conclusion
The OffSecOps Jenkins pipeline now has, what I would consider the core functionality of the pipeline demonstrated by Will Schroeder. There are still a few features that are missing and I will continue to work on them. I will also be investigating the ability to apply similar techniques to compile and obfuscate C and C++ projects. (Like Mimikatz) Other features that are missing, that were present in the SO-CON talk are: 

- Comment Stripping 
- Artifact Fingerprinting 
- sRDI Conversion 
- Scans for tool IOCs presence in online submission/scanning tools 
- Slack Alerts 
- For new GitHub/GitLab commits 
- For alerts of detection of submitted artifacts 
- Opsec checks (Pester Tests) 
- C\C++ Support 

I will continue to work on my OffSecOps pipeline and possibly post some more blog entries as I make progress and add new features. Happy hacking!

# References
- [https://specterops.io/so-con2020/event-758918](https://specterops.io/so-con2020/event-758918)
- [https://raw.githubusercontent.com/specterops/presentations/master/Will%20Schroeder/OffSecOps_SO-CON2020.pdf](https://raw.githubusercontent.com/specterops/presentations/master/Will%20Schroeder/OffSecOps_SO-CON2020.pdf)
- [https://www.jenkins.io/doc/book/pipeline/shared-libraries/](https://www.jenkins.io/doc/book/pipeline/shared-libraries/)
- [https://www.jenkins.io/doc/book/pipeline/syntax/](https://www.jenkins.io/doc/book/pipeline/syntax/)
- [https://groovy-lang.org/](https://groovy-lang.org/)
- [https://mkaring.github.io/ConfuserEx/](https://mkaring.github.io/ConfuserEx/)
