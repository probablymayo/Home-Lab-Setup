<h1>Home-Lab-Setup</h1>

This lab is an **Active Directory** environment that I modeled this lab after the show "Mr. Robot" so if you watch the show you'll see a lot of familiar names 

<img width="434" height="224" alt="image" src="https://github.com/user-attachments/assets/79ce34d5-2891-4327-a136-fa049d1e67c3" />

<h2>Summary</h2>

The envrionment is housed on a Proxmox host, which is a refurbished Dell Poweredge r730xd I picked up off amazon. 

Current VMs:
- Windows 10 Enterprise / FlareVM Workstation (1)
- WIndows 10 Enterprise Workstation (2)
- pfSense Firewall
- Kali Linux 
- Kali Purple 
- Domain Controller / Windows Server 2022
- Ubuntu Server

<h3>Win10E Workstations</h3>

My 2 workstations for domain users

These are both configured as FlareVMs

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

BEFORE ANYTHING:
- I enabled the local admin account and set a password
- I opened powershell as that local admin using the .\Administrator and respective credentials


Went to this link in my browser: https://ln5.sync.com/dl/670e23aa0/view/default/39754955422012#xt8jghas-j8utjk35-yms4wyt2-rbpxfyfw 
The link is a “SYSMON” logging download script 
Meant to automate the deployment of Sysmon (System Monitor) on a Windows system
How the script works: 

I downloaded it, then i opened a powershell window as an admin
After opening it, i navigated to the following directory where i downloaded the script → 	C:\Users\pprice\Downloads

Commands: 
Set-ExecutionPolicy
ExecutionPolicy: Unrestricted
Select A for yes to all 
Get-ExecutionPolicy to check if it worked
 .\Install-Sysmon-m122configv2_1.ps1
Select R to finish it 

I logged into Event Viewer as an admin to see if it worked 
Directory =    Applications and Services → Microsoft → Windows → Sysmon → Operational
This directory showed all the events from the install, no events would mean something went wrong

Went ahead and downloaded some Atomic Red Team scripts which allow for emulating attacks to test logging and detections

- Attack Script:
  - https://ln5.sync.com/dl/da40042f0/view/default/23948580772012#qax9aer8-rah2u6i3-sbws4xhr-6sryqqgf 

- Cleanup Script:
  - https://ln5.sync.com/dl/e5d8e9540/view/default/23948585022012#wb6akfvr-tefjzqhx-bgabz4gx-ua9nwjkr 

I downloaded the scripts but didn’t use them yet (used them later for labs n shit) 

For troubleshooting I went into the actual script itself and found a -password version of it to get it to run 
Would not run despite disabling defender and real time protection multiple times via settings and gpedit 

I did this on both windows virtual machines
