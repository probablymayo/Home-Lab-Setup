<h1>Home-Lab-Setup</h1>

This lab is an **Active Directory** environment that I modeled this lab after the "Mr. Robot" series, so if you watch the show you'll see a lot of familiar names 

<img width="434" height="224" alt="image" src="https://github.com/user-attachments/assets/79ce34d5-2891-4327-a136-fa049d1e67c3" />

<h2>Summary</h2>

The envrionment is housed on a Proxmox host, which is a refurbished Dell Poweredge r730xd I picked up off amazon. 
- Some say this is overkill but one of the clients I decomissioned a server room for at work offered me one of their full sized racks. The deal was too good to pass up so I bought the r730 to give me excuse to have it

I have a cisco **CBS-350** 16 port managed switch as well which is the uplink for the server but currently I haven't done much other than intial setup via **puTTy**. I will tinker with this more when I start to persue the CCNA 

There are 3 VLANs that I configured in pfSense for an Attack LAN, Allsafe LAN, and Ecorp LAN

Current VMs:
- Windows 10 Enterprise / FlareVM Workstation (1)
- WIndows 10 Enterprise Workstation (2)
- pfSense Firewall
- Kali Linux 
- Kali Purple 
- Domain Controller / Windows Server 2022
- Ubuntu Server

All of the ISO files I used are here in **lvm**, NOT **local-lvm**:
- <img width="591" height="290" alt="image" src="https://github.com/user-attachments/assets/8f919b07-c458-44a5-ad4d-9d73a9ffc016" />

<h3>Win10E Workstations</h3>

Both of my domain user workstations are running Windows 10 Enterprise and are configured as FlareVMs

**FLARE** = FireEye Labs Advanced Reverse Engineering
- Open-source and meant for windows virtual machines

This is a vm configuration & toolkit Designed for incident response, malware analysis & reverse engineering. The install is modular, but I decided to select all of the available tools

I automated the install with “**Chocolatey**”

A jist of the tools: 
- Decompilers 
- IDA free, Ghidra, x64dbg
- Disassemblers 
- dnSpy, JD-GUI
- Static analysis tools
- PE-bear, Detect-it-Easy
- Dynamic analysis tools
- Process monitors, wireshark
- Forensics utilities for memory dumps, disk images, network traffic etc. 

<h4>**Enabling Local Administrator**</h4>

- I enabled the local admin account and set a password
- I opened powershell as that local admin using the ```.\Administrator``` and respective credentials

<h4>**FlareVM & Sysmon Installation**</h4>

<img width="462" height="442" alt="image" src="https://github.com/user-attachments/assets/c4c6a01d-e759-4100-8c25-11fa6cd4df94" />

<img width="475" height="173" alt="image" src="https://github.com/user-attachments/assets/02a897b6-316a-498f-aeaa-6fe539a3fe58" />

Sysmon is a tool part of Microsoft **Sysinternals** that monitors and logs system-level events such as process creation, network connections, and registry changes. Once downloaded, usually it’s stored in ```C:\ProgramData``` directory
- I downloaded it, then i opened a powershell window as an admin
- After opening it, i navigated to the following directory where i downloaded the script → 	```C:\Users\pprice\Downloads```

I downloaded **Sysmon** via This link to a download script:
- ```https://ln5.sync.com/dl/670e23aa0/view/default/39754955422012#xt8jghas-j8utjk35-yms4wyt2-rbpxfyfw``` 
 - This script is meant to automate the deployment of Sysmon.

I didn't write this script but here is how it works: 
- The script begins by defining a parameter for the Sysmon configuration file, pointing to a github URL where a custom XML-based configuration can be downloaded. It then prints a message indicating that the installation process has started and performs DNS lookups for several domains including ```download.sysinternals.com```, ```github.com```, and ```raw.githubusercontent.com```. This step helps confirm that the system has internet access and can reach the necessary resources.
- Next, the script downloads the official Sysmon ZIP archive from Microsoft and stores it in the `C:\ProgramData` directory. It extracts the contents of the ZIP file and identifies the path to the Sysmon executable. After ensuring the binary is available, it proceeds to download the Sysmon configuration file from the provided URL and saves it locally as an XML file. This configuration dictates which system events Sysmon should monitor and log.
- Once both the Sysmon binary and configuration file are available, the script installs Sysmon as a Windows service using the `-accepteula` flag to automatically accept the license agreement. It then configures the service to start automatically at system boot. To make sure there's appropriate access to the Sysmon event logs, the script sets permissions on the "**Microsoft-Windows-Sysmon/Operational**" channel so that system processes, administrators, and event log readers can access it.
- Finally, the script checks whether the Sysmon service is running. If not, it attempts to start it and waits until the service reaches the "**Running**" state. A confirmation message is displayed once Sysmon is verified to be running. 

Now I can download FlareVM via a powershell script: 
- I can insall if from it's gihub page [HERE](https://github.com/mandiant/flare-vm?tab=readme-ov-file)
- I can also use this command:
```
New-Object net.webclient).DownloadFile('https://raw.githubusercontent.com/mandiant/flare-vm/main/install.ps1',"$([Environment]::GetFolderPath("Desktop"))\install.ps1"
```

At this point I opened powershell as an admin and then changed the executed policy to be able to run the script, Next was to set the **execution policy** 
Commands: 
```
Set-ExecutionPolicy unrestricted
```
- Select A for yes to all

`Get-ExecutionPolicy` to check if it worked
- `.\Install-Sysmon-m122configv2_1.ps1`

Select `R` to finish it 

Login to Event Viewer as an admin to see if it worked 
- Directory = `Applications and Services → Microsoft → Windows → Sysmon → Operational`
 - This directory showed all the events from the install, no events would mean something went wrong

Went ahead and downloaded some Atomic Red Team scripts which allow for emulating attacks to test logging and detections later on in exercises

- Attack Script:
  - `https://ln5.sync.com/dl/da40042f0/view/default/23948580772012#qax9aer8-rah2u6i3-sbws4xhr-6sryqqgf `

- Cleanup Script:
  - `https://ln5.sync.com/dl/e5d8e9540/view/default/23948585022012#wb6akfvr-tefjzqhx-bgabz4gx-ua9nwjkr `

For troubleshooting I went into the actual script itself and found a `-password` version of it to get it to run. I was having issues since it would not run despite disabling defender and real time protection multiple times via **settings** and **gpedit** 
