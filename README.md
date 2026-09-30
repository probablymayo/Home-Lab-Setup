# Home SOC Lab: Mr. Robot Themed Active Directory Environment

This lab is an **Active Directory** environment that I modeled after the "Mr. Robot" series, so anyone who has watched the show will recognize a lot of the names. I use it to practice detection engineering and incident investigation by simulating attacks against the Windows hosts and then building and testing detections in **Wazuh** and **Splunk**.

By no means is this an exhaustive list of everything I did to set up the environment. I drew from YouTube videos, online courses (such as Udemy), and tutorials, and I adjusted the setup to make the lab my own. I included the software and tools that make the lab function.

I document what I install because the Windows licenses are not permanent. I can re-arm them a limited number of times, so I have to rebuild my VMs periodically. I included install instructions in case they are useful for anyone setting up their own lab or installing tools such as Wazuh or Velociraptor.

**Skills demonstrated:** network segmentation (pfSense, Open vSwitch VLANs), Active Directory, Sysmon telemetry, custom Wazuh rules, credential theft detection (MITRE ATT&CK T1003.001), hash-based malware alerting, log analysis in Splunk and Event Viewer, and malware analysis tooling (FlareVM).

## Contents

[Architecture](#architecture) | [Tool Status](#tool-status) | [Proxmox Setup](#proxmox-setup) | [Networking](#networking) | [Wazuh](#wazuh) | [Sysmon](#sysmon) | [Detection: Mimikatz](#detection-mimikatz) | [Detection: Malware Hashes](#detection-malware-hashes) | [Windows Workstations](#windows-workstations) | [Investigation Practice](#investigation-practice) | [Limitations and Next Steps](#limitations-and-next-steps) | [Credits](#credits)

![Lab overview](https://github.com/user-attachments/assets/79ce34d5-2891-4327-a136-fa049d1e67c3)

## Architecture

The environment is housed on a Proxmox host, which is a refurbished Dell PowerEdge R730xd that I purchased on Amazon. Some would consider it overkill, but I received a full-size rack from work at no cost, so it made sense to use the space.

I also have a Cisco **CBS-350** 16-port managed switch, which is the uplink for the server. So far I have only completed the initial setup through **PuTTY**, and I plan on configuring it further when I start studying for the CCNA.

There are three VLANs configured in pfSense: an Attacker LAN, an AllSafe LAN, and an ECorp LAN. They are carried on an Open vSwitch bridge in Proxmox.

| VLAN | Hosts | Purpose |
|---|---|---|
| ECorp | Win10E-1, Win10E-2, Domain Controller (Windows Server 2022) | Target environment |
| AllSafe | Kali Purple (Wazuh), Ubuntu Server | Monitoring and defense |
| Attacker | Kali Linux | Attack simulation |

Current VMs:

- Windows 10 Enterprise / FlareVM Workstation (1)
- Windows 10 Enterprise / FlareVM Workstation (2)
- pfSense Firewall
- Kali Linux
- Kali Purple
- Domain Controller / Windows Server 2022
- Ubuntu Server

## Tool Status

| Tool | Purpose | Status |
|---|---|---|
| Wazuh | SIEM and XDR (runs on Kali Purple) | Live |
| Sysmon | Endpoint telemetry on Windows hosts | Live |
| Splunk (free license) | Log analysis | Live |
| FlareVM | Malware analysis workstations | Live |
| Atomic Red Team | Attack emulation | Live |
| Velociraptor | Endpoint DFIR | In progress, documentation coming |
| Shuffle | SOAR | Planned |
| TheHive | Case management | Planned |
| Suricata | Network IDS | Planned |

## Proxmox Setup

All of the ISO files I used are stored in **lvm**, not **local-lvm**:

![ISO storage](https://github.com/user-attachments/assets/8f919b07-c458-44a5-ad4d-9d73a9ffc016)

I created a file share and purposely misconfigured it, since one of my labs is to parse a misconfigured file share for stored credentials.

### SPICE Configuration

I installed the SPICE guest tools from the official page. Then I shut down and edited every VM that I planned to use SPICE on:

- **Display** = SPICE
- **Memory** = 128 MiB
- **Machine** = q35

![SPICE settings](https://github.com/user-attachments/assets/606358bb-bf89-4799-ab3a-08d28f02cf77)

After restarting each VM, I opened the virtio storage drive that I attached during the initial setup (through the VM's file explorer) and ran the **win-guest-tools** executable. From this point on I used SPICE instead of noVNC for the console.

## Networking

I installed **Open vSwitch** by opening a shell on the Proxmox node and running:

```bash
apt install openvswitch-switch
```

In the Network section, I created an **OVS Bridge**. Then I created three **OVS IntPorts** and gave each one a VLAN tag:

![OVS IntPorts with VLAN tags](https://github.com/user-attachments/assets/a9159c8c-5337-43ac-8e2d-76fbef0d728b)

After that, I went to the pfSense VM and linked its network adapters to the virtual switch adapters:

![pfSense adapters](https://github.com/user-attachments/assets/40f4da05-3c29-4c23-a8e0-2aea3c055492)

At this point I had a virtual switch with three virtual network adapters, linked to three adapters on the pfSense VM, which successfully created the three VLANs.

I then set the VLAN for each machine by opening its network adapter, selecting the OVS bridge, and entering the VLAN tag:

**ECorp VLAN**
- Win10E-1
- Win10E-2
- Domain Controller

**AllSafe VLAN**
- Kali Purple
- Ubuntu Server

**Attacker VLAN**
- Kali Linux

## Wazuh

Wazuh is a security platform for networks and endpoints that supports monitoring, incident response, and threat detection.

- Integrates real-time log analysis, threat intelligence, file integrity monitoring, and endpoint protection
- Automatically checks for compliance standards such as GDPR, HIPAA, PCI DSS, and NIST
- Tracks unauthorized modifications to files and directories

It offers much more than what is listed here, but this covers the main features that I use.

I installed it on my Kali Purple VM and completed the initial configuration. To allow Wazuh to read Sysmon logs from the Windows hosts, I edited the agent configuration file at `C:\Program Files (x86)\ossec-agent\ossec.conf` using **Notepad++**:

![ossec.conf edit for Sysmon](https://github.com/user-attachments/assets/0ad7e0e4-0e60-4598-88ca-2f20d8f986ee)

After saving the file, I restarted the agent service:

```
NET STOP WazuhSvc
NET START WazuhSvc
```

## Sysmon

Sysmon is a tool that is part of Microsoft **Sysinternals**. It monitors and logs system-level events such as process creation, network connections, and registry changes.

I deployed it with an install script that `[credit the original author and link the source, or commit a copy to a scripts/ folder with attribution]`. I did not write this script, but here is how it works:

1. The script defines a parameter for the Sysmon configuration file, which points to a GitHub URL where a custom XML configuration can be downloaded.
2. It performs DNS lookups for `download.sysinternals.com`, `github.com`, and `raw.githubusercontent.com` to confirm that the system has internet access and can reach the necessary resources.
3. It downloads the official Sysmon ZIP archive from Microsoft into `C:\ProgramData` and extracts it.
4. It downloads the configuration file and saves it locally as an XML file. This file dictates which system events Sysmon monitors and logs.
5. It installs Sysmon as a Windows service with the `-accepteula` flag and configures the service to start automatically at boot.
6. It sets permissions on the **Microsoft-Windows-Sysmon/Operational** channel so that system processes, administrators, and event log readers can access the logs.
7. It checks that the service is running, starts it if it is not, and confirms once it reaches the **Running** state.

To run the script, I opened PowerShell as an admin and allowed scripts for the current session only:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
Get-ExecutionPolicy
.\Install-Sysmon-m122configv2_1.ps1
```

I selected `R` to finish. To confirm that it worked, I logged into Event Viewer as an admin and checked the following directory:

- Applications and Services Logs → Microsoft → Windows → Sysmon → Operational

This directory should show all of the events from the install. If there are no events, something went wrong.

## Detection: Mimikatz

To test credential theft detection, I connected to the Wazuh server and added an entry to the Wazuh rules file:

```xml
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

### Entry Breakdown

The first line creates a group of rules named `windows, sysmon, sysmon_process-anomalies`. This allows Wazuh to classify and organize rules that match Sysmon events on Windows systems. The entry contains three rules. The rule ID numbers are somewhat arbitrary, and the levels range from 0 to 15, with higher numbers representing higher severity.

**1. Mimikatz Execution (Rule 100000)**

`sysmon_event1` is a **process creation** event. If the event matches **mimikatz.exe**, Wazuh generates an alert with a severity of 12. In other words, if Mimikatz runs at all, the event is seen and the alert is triggered.

**2. Mimikatz Remote Thread (Rule 100001)**

`sysmon_event8` is a **remote thread creation** event. A thread is a small piece of executable instructions, and multiple threads make up a process. A remote thread is a thread that runs inside the memory space of another process. This rule detects it because remote threads are commonly used for credential theft and process injection.

**3. Mimikatz Process Access (Rule 100002)**

`sysmon_event_10` is a **process access** event. This rule triggers when Mimikatz accesses another process, and the alert includes the name of the target process.

### Testing

Once the rules were in place, I downloaded Mimikatz on my Windows VM and ran several commands:

![Mimikatz commands](https://github.com/user-attachments/assets/ebe55d91-e969-4688-9750-0a8e743809d5)

- `privilege::debug` instructs Mimikatz to enable the debug privilege. This privilege allows a user to interact directly with system processes and memory, which can bypass some security mechanisms.
- `log mimikatz.log` directs the output of Mimikatz commands to a file named mimikatz.log. This is useful for later analysis.
- `sekurlsa::logonpasswords` reads credential material from the memory of the LSASS process (MITRE ATT&CK T1003.001). It prints NTLM hashes, and in some cases plaintext passwords, for every account with a logon session on that machine.

I expected to see only a couple of hashes, but the output listed nearly every user that had logged on to that machine, and all of it was written to the log file I created with the previous command. These were the accounts active on the local machine rather than the entire domain, but every one of them was still exposed. I was also able to see the activity in Event Viewer when I investigated.

## Detection: Malware Hashes

On the Kali Purple VM, I configured Wazuh with a CDB list of malware hashes and a rule that alerts when a file matches one of those hashes.

**1. Create the CDB list.** I created a file named malware-hashes in the `/var/ossec/etc/lists` directory:

```bash
sudo nano /var/ossec/etc/lists/malware-hashes
```

I added a key:value pair to the file (the malware hash and its name):

```
85061FB539F0E118805729C0D9EFA99E:mimikatz
```

**2. Set the file permissions.** I limited access so that only root and the Wazuh service can modify the list:

```bash
sudo chown root:wazuh /var/ossec/etc/lists/malware-hashes
sudo chmod 660 /var/ossec/etc/lists/malware-hashes
```

**3. Register the list.** I added a reference to the new list in the Wazuh manager configuration file, `/var/ossec/etc/ossec.conf`, under the default ruleset block:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

I pasted this line into the file: `<list>etc/lists/malware-hashes</list>`

![List registered in ossec.conf](https://github.com/user-attachments/assets/a6afb949-685d-4682-b88d-b650cc7bcff8)

**4. Add a custom rule.** I added a rule to `/var/ossec/etc/rules/local_rules.xml`:

![Custom hash rule](https://github.com/user-attachments/assets/7d0d2045-764f-4d77-895b-340cb539885d)

Rule breakdown: when Wazuh finds a match between the MD5 hash of a recently created or modified file and a hash in the CDB list, this rule triggers. A newly created or modified file triggers the built-in rules 554 (file added) and 550 (file modified), and this custom rule builds on them.

**5. Restart the manager.** I saved the changes and restarted the Wazuh manager to apply them:

```bash
sudo systemctl restart wazuh-manager
```

**6. Configure the Windows VMs.** I configured the Windows VMs to monitor file changes, which allows them to use the CDB list for hash comparisons. I edited `C:\Program Files (x86)\ossec-agent\ossec.conf` and added an entry to monitor the Downloads directory:

![syscheck directories entry](https://github.com/user-attachments/assets/c4839cf8-75bf-449e-aa80-b04b0253512b)

I replaced the placeholder username with the respective user for that machine (either tcolby or pprice). The entry options work as follows:

- `check_all="yes"` has Wazuh verify every attribute of the file, such as its size, permissions, owner, last modification date, inode, and hash sums.
- `realtime="yes"` has Wazuh monitor in real time and trigger alerts as soon as a file changes.

![syscheck realtime entry](https://github.com/user-attachments/assets/6d36f576-b557-4030-b693-ef6759759e99)

After this, I restarted the agent service again:

```
NET STOP WazuhSvc
NET START WazuhSvc
```

To test that my rules worked, I attempted to download Mimikatz again on the Windows VMs.

## Windows Workstations

Both of my domain user workstations run Windows 10 Enterprise and are configured as FlareVMs.

**FLARE** stands for FireEye Labs Advanced Reverse Engineering. It is an open-source VM configuration and toolkit from Mandiant, designed for Windows virtual machines, and it is meant for incident response, malware analysis, and reverse engineering. The install is modular, and I decided to select all of the available tools. I automated the install with **Chocolatey**.

A summary of the tools:

- **Decompilers and debuggers:** IDA Free, Ghidra, x64dbg
- **Disassemblers:** dnSpy, JD-GUI
- **Static analysis tools:** PE-bear, Detect It Easy
- **Dynamic analysis tools:** Process Monitor, Wireshark
- **Forensics utilities:** memory dumps, disk images, and network traffic analysis

### Enabling the Local Administrator

I enabled the local administrator account and set a password. Then I opened PowerShell as that user with `.\Administrator` and the respective credentials.

### FlareVM Installation

![Install screenshot 1](https://github.com/user-attachments/assets/c4c6a01d-e759-4100-8c25-11fa6cd4df94)
![Install screenshot 2](https://github.com/user-attachments/assets/02a897b6-316a-498f-aeaa-6fe539a3fe58)

I installed FlareVM in two different ways, one on each VM. The first was to have PowerShell download the installer:

```powershell
(New-Object net.webclient).DownloadFile('https://raw.githubusercontent.com/mandiant/flare-vm/main/install.ps1',"$([Environment]::GetFolderPath("Desktop"))\install.ps1")
```

The second was to download the `install.ps1` file directly from the [FLARE-VM GitHub page](https://github.com/mandiant/flare-vm). The installer requires an unrestricted execution policy and Windows Defender to be disabled, which I consider acceptable on these isolated lab VMs.

### Atomic Red Team

I downloaded **Atomic Red Team** scripts, which allow me to emulate attacks and test logging and detections in later exercises. The attack and cleanup scripts came from `[course or source, with link]`.

For troubleshooting, the attack script would not run even after I disabled Defender and real-time protection multiple times through **Settings** and **gpedit**. I opened the script itself and found a `-password` version of it, which allowed it to run.

## Investigation Practice

I integrated **Splunk** into the environment. It is not especially complicated because I am running the free license, but I still use **Event Viewer** to practice writing custom XML queries and to find quick information in logs.

I also have some red team scripts for log analysis. These let me simulate investigations without performing a live attack each time, and they make the practice more realistic, since the log files contain thousands of events that I have to search through as I would in an enterprise environment.

## Limitations and Next Steps

- The Mimikatz rules match on the filename, so a renamed binary would evade them. I plan on adding a detection for Sysmon Event 10 where the target image is `lsass.exe`, and using Sysmon's OriginalFileName field.
- The hash detection relies on a single MD5 value, which is easy to bypass. I plan on adding reputation lookups such as VirusTotal.
- I am finishing the Velociraptor documentation, and I plan on implementing **Shuffle** for SOAR and **TheHive** for case management.
- I plan on adding **Suricata** for network detection.

## Credits

I built this lab with help from YouTube tutorials, Udemy coursework, `[Sysmon config source]`, Atomic Red Team, and Mandiant's FLARE-VM.

## Lab Safety

All attack tooling runs inside isolated VLANs. Windows Defender was disabled only on lab VMs, and all credentials shown belong to lab accounts.
