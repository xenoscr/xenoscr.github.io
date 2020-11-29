---
layout: post
title: "OffSecOps Basic Setup"
---
It's been over 8 months since my last post. It has been a very long year filled with personal struggles compounded by the challenges that 2020 has brought. I hope that I will be able to post new content a bit more frequently. We'll see how that goes. If you're having a hard time, hang in there, things will get better.

# Introduction
Recently at SpecterOps' SO-CON 2020, Will Schroeder ([@harmj0y](https://twitter.com/HarmJ0y/)) gave a talk titled: OffSecOps. Almost immediately after I finished watching the talk I started to research what it would take to setup my own Jenkins Pipeline. This blog post is the result of me documenting the work I have done to work on setup a PoC pipeline that I can later implement in production to aid in my day job. I decided to post my notes in case they can help others. The example pipeline that Will shared is located at the following URL and has served a template for the pipeline I am attempting to setup:

* [https://gist.github.com/HarmJ0y/a8a5fe987f2b7751eb2ce55c2eec155d](https://gist.github.com/HarmJ0y/a8a5fe987f2b7751eb2ce55c2eec155d)

So far, I have completed a very basic pipeline that is capable of cloning a project from a GitLab repository, builds it, and  upload's the compiled EXE to an Artifactory repository. I have not worked on any obfuscation, IOC generation, or Unit tests. I will continue to pursue these capabilities and may blog about them in the future. In the meantime, I hope that this helps someone start their journey to build their own OffSecOps pipeline.

# Install Notes
## Sections
- [Overview](#overview)
- [Resources](#resources)
- [Windows System Stup](#windows-system-setup)
  - [Installing Visual Studio Community](#installing-visual-studio-community)
  - [Installing Java](#installing-java)
  - [Installing Artifactory](#installing-artifactory)
  - [Installing Git](#installing-git)
  - [Installing Jenkins](#installing-jenkins)
    - [Change the Folder Permissions](#change-the-folder-permissions)
    - [Change to a Virtual Account](#change-to-a-virtual-account)
    - [Finish the Jenkins Configuration](#finish-the-jenkins-configuration)
    - [Install the GitLab Plugin](#install-the-gitlab-plugin)
    - [Install the PowerShell Plugin](#install-the-powershell-plugin)
    - [Install the NUnit Plugin](#install-the-nunit-plugin)
    - [Setup a Defender Exception](#setup-a-defender-exception)
    - [Basic Pipeline Setup](#basic-pipeline-setup)
- [Conclusion](#conclusion)
    
## Overview
So far I have used two systems, one running Windows and another running GitLab. Based on the example Jenkins Pipeline that Will shared, the build system appears to be running Windows. In his talk, he mentioned maintaining an internal SCM system that houses the source code for projects that his team uses. The example he shared uses one of their own repositories, Seatbelt, which is hosted on GitHub. You can skip setting up your own GitLab instance if you like. I will not be covering its installation but, if you're interested the instructions can be found here:

* [https://about.gitlab.com/install/](https://about.gitlab.com/install/)

## Resources
- Windows 10 Pro (Minimum version that supports RDP) to install:
  - Jenkins 
  - Artifactory 
  - Visual Studio Community 2017 (for older builds) 
  - Visual Studio Community 2019 
  - Git 
- Ubuntu 20.04.1 to install: 
  - GitLab

## Windows System Setup
### Installing Visual Studio Community
Install the versions of Microsoft's Visual Studio Community that you will need to compile your target projects. I chose to install 2017 and 2019 to provide the ability to compile for older systems if necessary. This is a fairly common task and I will not go into detail. You can obtain Visual Studio from the following URL:

- [https://visualstudio.microsoft.com/downloads/](https://visualstudio.microsoft.com/downloads/)

### Installing Java
1. Download the latest version of JDK 11 from:
   - [https://download.java.net/java/GA/jdk11/9/GPL/openjdk-11.0.2_windows-x64_bin.zip](https://download.java.net/java/GA/jdk11/9/GPL/openjdk-11.0.2_windows-x64_bin.zip)
2. Extract the zip file to **C:\\Program Files\\OpenJDK\\jdk-11\\**

<p align="center">
    <img src="/resources/images/2020-11-28-OffSecOps-Basic-Setup/figure01.png" alt="Figure 1: Extract OpenJDK">
</p>
<p align="center">
    <em>Figure 1: Extract OpenJDK</em>
</p>

{:start="3"}
3. Add the path **C:\\Program Files\\OpenJDK\\jdk-11\\bin** to the System Path.

<p align="center">
    <img src="/resources/images/2020-11-28-OffSecOps-Basic-Setup/figure02.png" alt="Figure 2: Add OpenJDK to the System PATH variable">
</p>
<p align="center">
    <em>Figure 2: Add OpenJDK to the System PATH variable</em>
</p>

{:start="4"}
4. Add a new System Environment Variable named **JAVA_HOME** and a value that matches the path of your JDK.

<p align="center">
    <img src="/resources/images/2020-11-28-OffSecOps-Basic-Setup/figure03.png" alt="Figure 3: Add a JAVA_HOME System Environment Variable">
</p>
<p align="center">
    <em>Figure 3: Add a JAVA_HOME System Environment Variable</em>
</p>

### Installing Artifactory
1. Download the JFrog Artifactory OSS version from the download link.

   - [https://jfrog.com/open-source/#artifactory](https://jfrog.com/open-source/#artifactory)

2. Extract the ZIP file to the **C:\\** folder.

| **NOTE**: Do not try to install Artifactory in a directory with spaces. The scripts cannot handle spaces and your installation will not work. |

{:start="3"}
3. Add a new System Environment Variable called **ARTIFACTORY_HOME** and the path to where you extracted Artifactory to.

<p align="center">
    <img src="/resources/images/2020-11-28-OffSecOps-Basic-Setup/figure04.png" alt="Figure 4: Add a ARTIFACTORY_HOME System Environment Variable">
</p>
<p align="center">
    <em>Figure 4: Add a ARTIFACTORY_HOME System Environment Variable</em>
</p>

{:start="4"}
4. Open an Administrative Command prompt and run the **InstallService.exe** file in the **app\bin** folder.

5. Stop the **Artifactory** service.

6. Adjust the permissions on the **C:\artifactory-oss-7.11.2** folders in the following ways:

   - Break inheritance (Copy the permissions)
   - Remove the **Write** permission from these users:
     - **Users**
     - **Authenticated Users**
   - Add the **NT SERVICE\artifactory** user and grant them **Full Access**
   
7. Modify the **Artifactory** service to run as the **NT SERVICE\artifactory** user.
    **Change the Service Account to a Virtual Service Account**
    Windows 10 supports Virtual Accounts for services. Instead of leaving the service running as **NT AUTHORITY\SYSTEM**, we can change the service to run with a Virtual Account instead. More details about Virtual Service Accounts can be found here:

    - [https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2008-R2-and-2008/dd548356(v=ws.10)?redirectedfrom=MSDN#to-configure-a-service-to-use-a-virtual-accoun](https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2008-R2-and-2008/dd548356(v=ws.10\)?redirectedfrom=MSDN#to-configure-a-service-to-use-a-virtual-account)
    
    1. Open the **Services** dialogue and open the **Properties** for the **Artifactory** service.
    
    2. Select the **Log On** tab.
    
    3. Click **This account** and then type the following into the box.
    
        - **NT SERVICES\\artifactory**
        
    4. Delete the values from the **Password** and **Confirm Password** boxes.
    
    5. Click **OK**.
    
    6. Click **OK**.
    
    7. Restart the **Artifactory** service.

8. Start the **Artifactory** service.

9. Access the Artifactory logon page:

   - http://localhost:8082/ui/login/
   
10. Login with the default username and password.

11. Click **Get Started**.

<p align="center">
    <img src="/resources/images/2020-11-28-OffSecOps-Basic-Setup/figure05.png" alt="Figure 5: Click Get Started">
</p>
<p align="center">
    <em>Figure 5: Click Get Started</em>
</p>

{:start="12"}
12. Change the admin password.

<p align="center">
    <img src="/resources/images/2020-11-28-OffSecOps-Basic-Setup/figure06.png" alt="Figure 6: Change the admin password">
</p>
<p align="center">
    <em>Figure 6: Change the admin password</em>
</p>

{:start="13"}
13. Set the base URL.

<p align="center">
    <img src="/resources/images/2020-11-28-OffSecOps-Basic-Setup/figure07.png" alt="Figure 7: Set the base URL">
</p>
<p align="center">
    <em>Figure 7: Set the base URL</em>
</p>

{:start="14"}
14. Configure a proxy if you have one.

15. Click **Finish**.

#### Artifactory Configuration & Repository Creation
1. Log into Artifactory.

2. Click on Repositories.

3. Click **Add Repositories** then **Local Repository**.

<p align="center">
    <img src="/resources/images/2020-11-28-OffSecOps-Basic-Setup/figure08.png" alt="Figure 8: Create a Local Repository">
</p>
<p align="center">
    <em>Figure 8: Create a Local Repository</em>
</p>

{:start="4"}
4. Select **Generic**.

<p align="center">
    <img src="/resources/images/2020-11-28-OffSecOps-Basic-Setup/figure09.png" alt="Figure 9: Select Generic">
</p>
<p align="center">
    <em>Figure 9: Select Generic</em>
</p>

{:start="5"}
5. Type a name for your repository into the **Repository Key** field that ends in **-local**.

<p align="center">
    <img src="/resources/images/2020-11-28-OffSecOps-Basic-Setup/figure10.png" alt="Figure 10: Enter a Repository Key">
</p>
<p align="center">
    <em>Figure 10: Enter a Repository Key</em>
</p>

{:start="6"}
6. Click **Save & Finish**.

### Installing Git
1. Download the Git package from:

   - [https://git.scm.com/downloads](https://git.scm.com/downloads)
   
2. Run the Git installer and complete the installation.

### Installing Jenkins
1. Download Jenkins LTS from:

   - [https://www.jenkins.io/download/#downloading-jenkins](https://www.jenkins.io/download/#downloading-jenkins)
   
2. Open the installer MSI, then click **Next**.

<p align="center">
    <img src="/resources/images/2020-11-28-OffSecOps-Basic-Setup/figure11.png" alt="Figure 11: Run the Jenkins MSI Installer">
</p>
<p align="center">
    <em>Figure 11: Run the Jenkins MSI Installer</em>
</p>

{:start="3"}
3. Accept the default path and click **Next**.

<p align="center">
    <img src="/resources/images/2020-11-28-OffSecOps-Basic-Setup/figure12.png" alt="Figure 12: Accept the default path">
</p>
<p align="center">
    <em>Figure 12: Accept the default path</em>
</p>

{:start="4"}
4.Select **Run service as LocalSystem (not recommended)** then click **Next**.

<p align="center">
    <img src="/resources/images/2020-11-28-OffSecOps-Basic-Setup/figure13.png" alt="Figure 13: Select the LocalSystem installation option">
</p>
<p align="center">
    <em>Figure 13: Select the LocalSystem installation option</em>
</p>

{:start="5"}
5. Click **Test Port** then click **Next**.

<p align="center">
    <img src="/resources/images/2020-11-28-OffSecOps-Basic-Setup/figure14.png" alt="Figure 14: Test the port">
</p>
<p align="center">
    <em>Figure 14: Test the port</em>
</p>

{:start="6"}
6. Browse to the folder that OpenJDK is installed in then click **Next**.

<p align="center">
    <img src="/resources/images/2020-11-28-OffSecOps-Basic-Setup/figure15.png" alt="Figure 15: Specify the path to your OpenJDK installation">
</p>
<p align="center">
    <em>Figure 15: Specify the path to your OpenJDK installation</em>
</p>

{:start="7"}
7. Click **Next**.

<p align="center">
    <img src="/resources/images/2020-11-28-OffSecOps-Basic-Setup/figure16.png" alt="Figure 16: Click Next">
</p>
<p align="center">
    <em>Figure 16: Click Next</em>
</p>

{:start="8"}
8. Click **Install**.

<p align="center">
    <img src="/resources/images/2020-11-28-OffSecOps-Basic-Setup/figure17.png" alt="Figure 17: Click Install">
</p>
<p align="center">
    <em>Figure 17: Click Install</em>
</p>

{:start="9"}
9. Click **Finish**.

<p align="center">
    <img src="/resources/images/2020-11-28-OffSecOps-Basic-Setup/figure18.png" alt="Figure 18: Click Finish">
</p>
<p align="center">
    <em>Figure 18: Click Finish</em>
</p>

{:start="10"}
10. Ignore the **Getting Started** window for now. Once we setup a Virtual Service account, you will would be prompted to setup Jenkins a second time.

#### Change the Folder Permissions
1. Open the **File Explorer** and navigate to the **C:\Program Files\** folder.

2. Open the properties for the **Jenkins** folder.

3. Click the **Security** tab.

4. Click the **Advanced** button.

5. Click the **Continue** button.

6. Click the **Add** button.

7. Click **Select principle name**.

8. Type **NT SERVICE\jenkins** into the **Enter the object name to select** box.

<p align="center">
    <img src="/resources/images/2020-11-28-OffSecOps-Basic-Setup/figure19.png" alt="Figure 19: Enter NT SERVICE\jenkins">
</p>
<p align="center">
    <em>Figure 19: Enter NT Service\jenkins</em>
</p>

{:start="9"}
9. Click **OK**.

10. Select **Full Control**.

11. Click **OK**.

12. Click **OK**.

#### Change to A Virtual Account
1. Open the **Services** dialogue and open the **Properties** for the **Jenkins** service.

<p align="center">
    <img src="/resources/images/2020-11-28-OffSecOps-Basic-Setup/figure20.png" alt="Figure 20: Open the Jenkins service properties">
</p>
<p align="center">
    <em>Figure 20: Open the Jenkins service properties</em>
</p>

{:start="2"}
2. Select the **Log On** tab.

3. Click **This account** and then type the following into the box:

   - **NT SERVICE\\jenkins**
   
<p align="center">
    <img src="/resources/images/2020-11-28-OffSecOps-Basic-Setup/figure21.png" alt="Figure 21: Enter the Virutal Account name">
</p>
<p align="center">
    <em>Figure 21: Enter the Virtual Account name</em>
</p>

{:start="4"}
4. Delete the **Password** and **Confirm Password** values.

5. Click **OK**.

6. Click **OK**.

<p align="center">
    <img src="/resources/images/2020-11-28-OffSecOps-Basic-Setup/figure22.png" alt="Figure 22: Click OK to acknowledge the warning">
</p>
<p align="center">
    <em>Figure 22: Click OK to acknowledge the warning</em>
</p>

{:start="7"}
7. Restart the **Jenkins** service.

#### Finish the Jenkins Configuration
1. Open a web browser and open the following URL:

   - http://localhost:8080/
   
2. Locate the password in the referenced file and type it into the dialogue box, then click **Continue**.

<p align="center">
    <img src="/resources/images/2020-11-28-OffSecOps-Basic-Setup/figure23.png" alt="Figure 23: Enter the unlock password stored in the referenced file">
</p>
<p align="center">
    <em>Figure 23: Enter the unlock password stored in the referenced file</em>
</p>

{:start="3"}
3. Search for and select the MSBuild plugin.

<p align="center">
    <img src="/resources/images/2020-11-28-OffSecOps-Basic-Setup/figure24.png" alt="Figure 24: Select the MSBuild plugin">
</p>
<p align="center">
    <em>Figure 24: Select the MSBuild plugin</em>
</p>

{:start="4"}
4. Click **Install**.

5. Enter the information for your first admin account and click **Save and Continue**.

<p align="center">
    <img src="/resources/images/2020-11-28-OffSecOps-Basic-Setup/figure25.png" alt="Figure 25: Enter your admin account information">
</p>
<p align="center">
    <em>Figure 25: Enter your admin account information</em>
</p>

{:start="5"}
6. Click **Save and Finish**

7. Click **Start using Jenkins**.

#### Configure the MSBuild Plugin
1. Log into Jenkins.

2. Click **Manage Jenkins**.

3. Click **Global Tools Configuration**.

4. Scroll down to the **MSBuild** section and click **Add MSBuild**.

5. Fill in the **Name**, **Path to MSBuild**, and **Default parameters** fields.

<p align="center">
    <img src="/resources/images/2020-11-28-OffSecOps-Basic-Setup/figure34.png" alt="Figure 34: Enter the MSBuild details">
</p>
<p align="center">
    <em>Figure 34: Enter the MSBuild details</em>
</p>

{:start="6"}
6. Click **Save**.

#### Install the GitLab Plugin
1. Log into Jenkins.

2. Click **Manage Jenkins**.

3. Click **Manage Plugins**.

4. Click the **Available** tab.

5. Search for **GitLab**.

6. Place a check next to the **GitLab Plugin**.

7. Click **Install without restart**.

8. Let the installation complete and then restart Jenkins.

9. Log back into Jenkins and click **Manage Jenkins**.

10. Click **Configure System**.

11. Scroll down to the **GitLab** section and complete the **Connection Name**, **GitLab host URL**, and **Credentials** fields. I recommend setting up an API key for GitLab but you can use and ID and Password.

<p align="center">
    <img src="/resources/images/2020-11-28-OffSecOps-Basic-Setup/figure33.png" alt="Figure 33: Enter your GitLab server details">
</p>
<p align="center">
    <em>Figure 33: Enter your GitLab server details</em>
</p>

{:start="12"}
12. Click **Save**.

#### Install the PowerShell Plugin
The PowerShell Plugin will be needed to run the Pester Tests that the example pipeline is using to check the compiled binaries for IOCs.

1. Log into Jenkins.

2. Click **Manage Jenkins**.

3. Click **Manage Plugins**.

4. Click the **Available** tab.

5. Search for **PowerShell**.

6. Place a check next to the **PowerShell Plugin**.

7. Click **Install without restart**.

#### Install the NUnit Plugin
Pester's outputs its test results in NUnit format. This plugin will be needed once Pester tests are implemented.

1. Log into Jenkins.

2. Click **Manage Jenkins**.

3. Click **Manage Plugins**.

4. Click the **Available** tab.

5. Search for **NUnit**.

6. Place a check next to the **NUnit**.

7. Click **Install without restart**.

#### Setup a Defender Exception
I will be compiling binaries that Defender doesn't like very much. To keep things simple, I have chosen to setup an exception on the Jenkins service account profile's folder. This should be enough to keep Defender off my back, for now.

Setup an exception for the following folder:

- C:\Windows\ServiceProfiles\jenkins

#### Basic Pipeline Setup
Now to create a simple Jenkins Pipeline that will clone the source code from our local GitLab server, compile it, and then upload the compiled executable to the Artifactory repository. This is a very basic example that does not include much of the functionality that Will's example but, it will be possible to add additional stages that will allow us to get close to the same functionality that he demonstrated in his talk.

1. Log into Jenkins.

2. Click **New Item**.

<p align="center">
    <img src="/resources/images/2020-11-28-OffSecOps-Basic-Setup/figure26.png" alt="Figure 26: New Item">
</p>
<p align="center">
    <em>Figure 26: New Item</em>
</p>

{:start="3"}
3. Type a name for your pipeline into the **Enter an item name** field.

<p align="center">
    <img src="/resources/images/2020-11-28-OffSecOps-Basic-Setup/figure27.png" alt="Figure 27: Enter an item name">
</p>
<p align="center">
    <em>Figure 27: Enter an item name</em>
</p>

{:start="4"}
4. Select **Pipeline**.

<p align="center">
    <img src="/resources/images/2020-11-28-OffSecOps-Basic-Setup/figure28.png" alt="Figure 28: Select Pipeline">
</p>
<p align="center">
    <em>Figure 28: Select Pipeline</em>
</p>

{:start="5"}
5. Click **OK**.

<p align="center">
    <img src="/resources/images/2020-11-28-OffSecOps-Basic-Setup/figure29.png" alt="Figure 29: Click OK">
</p>
<p align="center">
    <em>Figure 29: Click OK</em>
</p>

{:start="6"}
6. Place a check next to **GitHub project** and fill in the **Project url** and select the **GitLab Connection** that was configured during the setup process from the drop down menu.

<p align="center">
    <img src="/resources/images/2020-11-28-OffSecOps-Basic-Setup/figure30.png" alt="Figure 30: Configure the GitLab Connection">
</p>
<p align="center">
    <em>Figure 30: Configure the GitLab Connection</em>
</p>

{:start="7"}
7. In the **Build Triggers** section, if you wish to automatically build your project based on a schedule, setup your criteria. For now I am using a manual trigger while I am working on developing a full solution.

8. Move down to the **Pipeline** section and enter the following skeleton pipeline script:

{% highlight Groovy linenos %}
pipeline { 
    agent { label 'master' } 

    environment { 
    } 
    stages { 
        stage('Checkout') { 
            steps { 
            } 
        } 
        stage('Build') { 
            steps { 
            } 
        } 
        stage('Publish') { 
            steps { 
            } 
        } 
    } 
} 
{% endhighlight %}

{:start="9"}
9. Our first stage, **Checkout**, is used to retrieve the source code from the repository. To complete this section click on the **Pipeline Syntax** link bellow the script dialogue. This will open a new tab. We're going to use the utility to help create an action to download the source code.

10. From the **Sample Step** drop-down menu, select **git: Git**. Complete the **Repository URL**, **Branch**, and **Credentials** fields with the URL of your target GitLab repository, the branch name, and credentials that will be used to clone the repository.

<p align="center">
    <img src="/resources/images/2020-11-28-OffSecOps-Basic-Setup/figure31.png" alt="Figure 31: Complete the required fields">
</p>
<p align="center">
    <em>Figure 31: Complete the required fields</em>
</p>

{:start="11"}
11. Click the **Generate Pipeline Script** button.

<p align="center">
    <img src="/resources/images/2020-11-28-OffSecOps-Basic-Setup/figure32.png" alt="Figure 32: Click Generate Pipeline Script">
</p>
<p align="center">
    <em>Figure 32: Click Generate Pipeline Script</em>
</p>

{:start="12"}
12. Copy the resulting script and paste it between the **steps** brackets of the **Checkout** section.

13. Next, copy and paste the following into the **environment** section. Be sure to update the values to match your solution's options. The **CONFIG** value should match the build configuration and the **PLATFORM** value should match the target CPU architecture of the solution.

{% highlight Groovy linenos %}
CONFIG = 'Release'
PLATFORM = 'Any CPU'
{% endhighlight %}

{:start="14"}
14. Copy and paste the following into the **steps** brackets of the **Build** section. Be sure to update the **Tool-Name** to match the name of the MSBuild instance required to compile your solution. The **Solution-Name** should match the name of the solution file in your project. You can modify this command to suit your needs. This example will work for most projects.

{% highlight Groovy linenos %}
bat "\"${tool 'Tool-Name'}\\MSBuild.exe\" /p:Configuration=${env.CONFIG} \"/p:Platform=${env.PLATFORM}\" /maxcpucount:%NUMBER_OF_PROCESSORS% /nodeReuse:false Solution-Name.sln"
{% endhighlight %}

{:start="15"}
15. Copy and paste the following script into the **steps** brackets of the **Publish** section. Be sure to update the **artifactory-repository** value to match the name of the Artifactory repository that you setup. Set the **Solution-Name** to match the name of your solution.

{% highlight Groovy linenos %}
script { 
    rtBuildInfo() 
    rtUpload ( 
        serverId: "artifactory-repository", 
        spec: 
        """{ 
            "files": [ 
                { 
                    "pattern": "*/bin/*/*.exe", 
                    "target": "artifactory-repository/Projects/Solution-Name/" 
                } 
            ] 
        }""" 
    ) 
} 
{% endhighlight %}

{:start="16"}
16. Copy and paste the following script into the **steps** brackets of the **Publish** section. Be sure to update the **artifactory-repository** value to match the name of the Artifactory repository that you setup. Set the **Solution-Name** to match the name of your solution.
17. The complete script should look something like this example:

{% highlight Groovy linenos %}
pipeline { 
    agent { label 'master' } 
    environment { 
        CONFIG = 'Release' 
        PLATFORM = 'Any CPU' 
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
        stage('Build') { 
          steps { 
              bat "\"${tool 'MSBuild-VS-2019'}\\MSBuild.exe\" /p:Configuration=${env.CONFIG} \"/p:Platform=${env.PLATFORM}\" /maxcpucount:%NUMBER_OF_PROCESSORS% /nodeReuse:false Seatbelt.sln" 
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
                                "target": "offsecops-local/Projects/Seatbelt/" 
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

{:start="18"}
18. Click **Save**.

19. Click **Build Now**.

# Conclusion
I still have much to do before this pipeline will ever be as complete and automated as the one that Will demonstrated in his talk but, this should serve as a good place to start. I have already begun to look into options to provide obfuscation and IOC extractions. I will share what I come up with as I have time. Until next time...
