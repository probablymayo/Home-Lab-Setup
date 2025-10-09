<h1>Home-Lab-Setup</h1>

This lab is an **Active Directory** environment that I modeled this lab after the "Mr. Robot" series, so if you watch the show you'll see a lot of familiar names 

By no means is this an "exhaustive" list of what I did to setup the environment. I drew from a bunch of youtube videos, online courses (like Udemy) and tutorials as well to try and make this lab a little unique. I will include some software and tools I use here to give an idea of what's making the lab "tick". 

I tend to document what I install since the windows licenses are not permanent. I can re-arm the licenses a certain number of times but I will have to periodically rebuild my vms. I included some install instructions for those who might want some inspiration for their own labs or maybe it can help them on their own install of things like Wazuh or Velociraptor. 

<img width="434" height="224" alt="image" src="https://github.com/user-attachments/assets/79ce34d5-2891-4327-a136-fa049d1e67c3" />

<h2>Summary</h2>

The envrionment is housed on a Proxmox host, which is a refurbished Dell Poweredge r730xd I picked up off amazon. 
- Some say this is overkill but I got a full sized rack for free from work so at that point... why tf not

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

I created a file share and purposefully misconfigured it since one of my labs is to parse a misconfigured file share for stored credentials 

I also have some red team scripts for log analysis, this lets me better simulate investigations without having to go through the headache of actually performing attacks. It also makes it more realistic since these log files have thousands of logs that I would have to search through like in an enterprise environment

I integrated **Splunk** into the environment. Not super complicated especially because I'm running a free plan but I still use **EventViewer** sometimes to touch up on creating custom XML scripts and finding quick info on logs 

<h2>SPICE Configuration</h2>

Installed the SPICE file from the official page  

I turned off and edited every VM i used SPICE for as such:
- **Display** = SPICE
- **Memory** = 128 MiB
- **Machine** = q35
<img width="766" height="360" alt="image" src="https://github.com/user-attachments/assets/606358bb-bf89-4799-ab3a-08d28f02cf77" />

I restarted the VM and then went to the virtio storage drive (in the VMs file explorer) i installed earlier during initial setup and ran the **win-guest-tools** executable file that was in that “drive” 

From this point on I used SPICE instead of NoVNC for console

<h2>Networking</h2>

I installed **openvswitch**. I opened a shell in the server’s node: 
```
apt install openvswitch-switch
```

Went to network section and created an **OVS Bridge ** with this command:
```
VLAN Interface
```

Then I created 3  OVS IntPorts and gave them vlan tags like so:

<img width="594" height="206" alt="image" src="https://github.com/user-attachments/assets/a9159c8c-5337-43ac-8e2d-76fbef0d728b" />

After this, I needed to go to the pfsense vm and linked at the network adapters to the vswitch adapters

<img width="591" height="222" alt="image" src="https://github.com/user-attachments/assets/40f4da05-3c29-4c23-a8e0-2aea3c055492" />

Now i have a virtual switch consisting of 3 virtual network adapters which are linked to 3 adapters on my pfsense server, successfully creating 3 VLANS 

I setup the VLANs for each machine (i went to each network adapter, selected the OVS bridge, then gave each a VLAN tag)

ECorp VLAN
- Win10E-1
- Win10E-2
- Domain controller 

AllSafe VLAN
- Kali Purple
- Ubuntu Server

Attacker VLAN
- Kali Linux

<h2>SOAR & Case Management</h2>

This is currently a work-in progress, I plan on implementing **Shuffle** for SOAR capabilities and **Hive** for case management.  

<h2>Wazuh</h2>

Wazuh is a security platform for networks and endpoints for monitoring, incident response and threat detection
- Integrates log analysis (in real time), threat intelligence, file integrity monitoring and endpoint protection
- Automatically checks for standards like GDPR, HIPAA, PCI DSS, NIST etc. 
- Tracks unauthorized modifications to files and directories 

It does way more but you get the gist of it

I installed this on my Kali Purple vm and then went though initial config

I dropped this powershell script in the windows vm notepad. It allows the Wazuh configuration to read Sysmon logs from the Windows hosts

I went into the windows vm and edited the `C:\\Program Files (x86)\\ossec-agent\\ossec.conf`. I did this with **notepad++ **

<img width="754" height="455" alt="image" src="https://github.com/user-attachments/assets/0ad7e0e4-0e60-4598-88ca-2f20d8f986ee" />

After I saved the edit to the file I restarted the service: 
```
NET STOP WazuhSvc
NET START WazuhScv
```

<h4>Sysmon Event Config</h4>

I went back into the Kali-purple machine and edited the SSH configuration file to allow my purple vm to SSH into my windows VM  
```
sudo nano /etc/ssh/sshd_config 
```

I then edited an entry: 

Original = `#PermitRootLogin prohibit-password`

New = `#PermitRootLogin yes`

<img width="674" height="342" alt="image" src="https://github.com/user-attachments/assets/b4e60943-9391-422b-ac7b-0d6fad13ceb4" />

After restarting the ssh service with `sudo service ssh restart`, I planned to test this by running mimikatz. I went into the Wazuh Rules file and added an entry:
```
<group name="windows, sysmon, sysmon_process-anomalies,">
   <rule id="100000" level="12">
     <if_group>sysmon_event1</if_group>
     <field name="win.eventdata.image">mimikatz.exe</field>
     <description>Sysmon - Suspicious Process - mimikatz.exe</description>
   </rule>
   <rule id="100001" level="12">
     <if_group>sysmon_event8</if_group>
     <field name="win.eventdata.sourceImage">mimikatz.exe</field>
     <description>Sysmon - Suspicious Process mimikatz.exe created a remote thread</description>
   </rule>
   <rule id="100002" level="12">
     <if_group>sysmon_event_10</if_group>
     <field name="win.eventdata.sourceImage">mimikatz.exe</field>
     <description>Sysmon - Suspicious Process mimikatz.exe accessed $(win.eventdata.targetImage)</description>
   </rule>
</group>

```
**Entry Breakdown**

First line creates a group of rules named windows, sysmon, sysmon_process-anomalies, this allows Wazuh to classify and organize rules for matching Sysmon events on Windows systems

The entry has 3 rules included in it. 

1. Mimikatz Execution
```
<rule id="100000" level="12">
     <if_group>sysmon_event1</if_group>
     <field name="win.eventdata.image">mimikatz.exe</field>
     <description>Sysmon - Suspicious Process - mimikatz.exe</description>
   </rule>
```
- The Rule ID number is somewhat arbitrary and the levels range from 0–15 with higher numbers representing higher severity. The `sysmon_event1` is a “**process creation**” event. The rest of this rule just says that if the event matches **mimikatz.exe** then there is a notification with severity of 12. So if mimikatz is run at all, the event is seen and the alert is triggered 

2. Mimikatz Remote Thread
```
<rule id="100001" level="12">
     <if_group>sysmon_event8</if_group>
     <field name="win.eventdata.sourceImage">mimikatz.exe</field>
     <description>Sysmon - Suspicious Process mimikatz.exe created a remote thread</description>
   </rule>
```
- The _sysmon_event8_ is a **Remote Thread Creation** event, so basically an event where mimikatz is creating a thread in another process. A Thread is a simple and small piece of executable programmed instructions (multiple threads make up a Process). The remote thread is a thread running within the cpu / memory space of another process. The rule detects this bc remote threads are usually used for credential theft or injection 

3. Mimikatz Process Access
```
<rule id="100002" level="12">
     <if_group>sysmon_event_10</if_group>
     <field name="win.eventdata.sourceImage">mimikatz.exe</field>
     <description>Sysmon - Suspicious Process mimikatz.exe accessed $(win.eventdata.targetImage)</description>
   </rule>
```

The _sysmon_event10_ is a **Process Access** event for when mimikatz accesses another process. 

Now that that's done, I downloaded mimikatz on my windows vm and ran some commands: 

<img width="686" height="319" alt="image" src="https://github.com/user-attachments/assets/ebe55d91-e969-4688-9750-0a8e743809d5" />

`privilege::debug` 
- This command instructs Mimikatz to enable debug privileges. Debug privileges are powerful permissions that allow a user to interact directly with system processes and memory, potentially bypassing security mechanisms.

`log mimikatz.log`
- This command sets up logging, directing the output of Mimikatz commands to a file named mimikatz.log. This can be useful for later analysis or auditing purposes.

`sekurlsa::logonpasswords`
- This command instructs Mimikatz to retrieve and display plaintext passwords from the Windows Security Account Manager (SAM) database, which contains user account information including passwords. Mimikatz's sekurlsa module specifically deals with credentials.
  - I obviously expected to see a couple hashes but this fuckin DUMPED almost every user and password on the domain. I didnt't expect this many hashes to be dropped but I found afterwards that those were all the users active on the local machine but nevertheless every username and password was exposed and outputted to the mimikatz log file i made with the previous command. And, even better, I was able to see this in eventviewer when i investigated

<h4>Malware Detection</h4>

Here’s what I did on the kali purple vm to configure the server with a CDB list with malware hashes and configure rules to prompt alerts when detecting a file with this hash: 
1. I made a CDB list file in the /ossec/etc/lists directory. The name for the file with the malware hashes is malware-hashes and i used this script:
```
sudo nano /var/ossec/etc/lists/malware-hashes
```
I added the following key:value pair to the file (malware hash & name):
```
85061FB539F0E118805729C0D9EFA99E:mimikatz 
```
2. Ran a script to change the file permissions to allow for read, write, execute
```
sudo chmod 777 /var/ossec/etc/lists/malware-hashes
```

3. I added the CDB list under the default ruleset block. Inputting the location of the list in the **<ruleset>** block allows me to add a reference to the new CDB list in the Wazuh manager config file/ directory `/var/ossec/etc/ossec.conf` 
- To access the **ossec.conf** file → `sudo nano /var/ossec/etc/ossec.conf`
- Pasted this into the file → `<list>etc/lists/malware-hashes</list>`

<img width="629" height="471" alt="image" src="https://github.com/user-attachments/assets/a6afb949-685d-4682-b88d-b650cc7bcff8" />

4. I added a custom rule to the servers `/var/ossec/etc/rules/local_rules.xml` file: 

<img width="733" height="268" alt="image" src="https://github.com/user-attachments/assets/7d0d2045-764f-4d77-895b-340cb539885d" />

Rule Breakdown:
- When Wazuh finds a match between the MD5 hash of a recently created or updated file and a malware hash in the CDB list, this rule triggers. When an event occurs that indicates a newly created or modified file exists, rules 554 and 550 will be triggered

5. After i saved, I restarted the Wazuh Manager to apply the changes:
``` 
sudo systemctl restart wazuh-manager
```

I visited the windows VMs and configured them to monitor file alterations, which lets them activate the CDB list for hash comparisons.

**On the Windows VMs:**

I edited the `C:\\Program Files (x86)\\ossec-agent\\ossec.conf` file and then added this entry to allow the directory monitoring / tracking file changes for the Downloads directory: 

<img width="986" height="114" alt="image" src="https://github.com/user-attachments/assets/c4839cf8-75bf-449e-aa80-b04b0253512b" />

- I changed the m122 to the respective user for that machine (either tcolby or pprice)

Entry Breakdown: 
- `check_all="yes"` --> This ensures that Wazuh verifies every aspect of the file, like its size, permissions, owner, last modification date, inode, and hash sums
- `realtime="yes”` --> Wazuh will perform real-time monitoring and trigger alerts.

<img width="869" height="641" alt="image" src="https://github.com/user-attachments/assets/6d36f576-b557-4030-b693-ef6759759e99" />

- Same here I replaced the m122 with the actual username

After this i restarted the wazuh svc service 
```
NET STOP WazuhSvc
NET START WazuhSvc
```

I tested that my modification rules worked by attempting to download mimikatz again on the windows vms 

<h2>Velociraptor</h2>



<h2>Windows 10 Workstations</h2>

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

<h3>**FlareVM & Sysmon Installation**</h3>

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

At this point I opened powershell as an admin and then changed the executed policy to be able to run the script, Next was to set the **execution policy** 
Commands: 
```
Set-ExecutionPolicy unrestricted
```
- Select A for yes to all

`Get-ExecutionPolicy` to check if it worked. If so then run the config: 
```
.\Install-Sysmon-m122configv2_1.ps1
```
- Select `R` to finish it 

Login to Event Viewer as an admin to see if it worked 
- Directory = `Applications and Services → Microsoft → Windows → Sysmon → Operational`
 - This directory showed all the events from the install, no events would mean something went wrong

Now I can download FlareVM via powershell or from downloading the ps1 file from github. I did the script on 1 vm and the file on the other: 
- I can install it from it's github page [HERE](https://github.com/mandiant/flare-vm?tab=readme-ov-file)
- Have powershell download it for me:
```
New-Object net.webclient).DownloadFile('https://raw.githubusercontent.com/mandiant/flare-vm/main/install.ps1',"$([Environment]::GetFolderPath("Desktop"))\install.ps1"
```

Went ahead and downloaded some Atomic Red Team scripts which allow for emulating attacks to test logging and detections later on in exercises

- Attack Script:
```
  https://ln5.sync.com/dl/da40042f0/view/default/23948580772012#qax9aer8-rah2u6i3-sbws4xhr-6sryqqgf 
```

- Cleanup Script:
```
https://ln5.sync.com/dl/e5d8e9540/view/default/23948585022012#wb6akfvr-tefjzqhx-bgabz4gx-ua9nwjkr 
```

For troubleshooting I went into the actual script itself and found a `-password` version of it to get it to run. I was having issues since it would not run despite disabling defender and real time protection multiple times via **settings** and **gpedit** 

