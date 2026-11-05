# Active-Directory-Home-Lab
Built an Active Directory home lab integrated with Splunk SIEM for centralized log monitoring, Sysmon telemetry collection, RDP brute-force attack detection, and Atomic Red Team simulations using MITRE ATT&amp;CK techniques.
**ACTIVE DIRECTORY HOME LAB
Deployment, Security, Attack Simulation & SIEM Monitoring
Project Owner: Akshay  |  Domain: akshay.local**
**1. Project Overview**
I undertook this project to deepen my understanding of securing and monitoring Microsoft Active Directory (AD), a widely used directory service in corporate environments. I built a simulated corporate network, configured a Windows Server as a Domain Controller, deployed Splunk Enterprise as a centralized SIEM, and collected endpoint telemetry using Sysmon and Splunk Universal Forwarder.
I also simulated repeated RDP authentication failures from Kali Linux using Hydra and performed controlled adversary-emulation tests with Atomic Red Team. The objective was to understand what activity is logged, how to investigate it in Splunk, and where monitoring coverage can be improved.
2. Project Objectives
**•	Deploy and secure an Active Directory domain in a virtual lab.
•	Create domain users and Organizational Units (OUs).
•	Deploy Splunk Enterprise on Ubuntu Server as the centralized SIEM.
•	Install Sysmon and Splunk Universal Forwarder on Windows machines.
•	Collect Windows Application, Security, System, and Sysmon events.
•	Simulate RDP authentication failures and investigate Event ID 4625.
•	Run selected Atomic Red Team tests mapped to MITRE ATT&CK.
•	Identify telemetry gaps and recommend improvements to detection coverage.**
3. Lab Architecture
The lab uses four virtual machines connected to a dedicated Oracle VirtualBox NAT Network. The firewall concept in this design refers to each machine's local host firewall; there is no separate firewall virtual machine.
Virtual machine	Purpose	Example IP
Ubuntu Server	Splunk Enterprise SIEM	192.168.10.10
Windows Server	Active Directory Domain Controller (ADDC01)	192.168.10.7
Windows 10	Domain-joined endpoint	DHCP reservation/static address
Kali Linux	Authorized attack simulation	DHCP reservation/static address
Network: 192.168.10.0/24 | Domain: akshay.local | Domain DNS: 192.168.10.7 | Splunk receiving port: TCP 9997 | RDP: TCP 3389.
Reserve addresses outside the DHCP allocation range or create DHCP reservations to prevent address conflicts. The gateway shown below is an example; use the gateway configured for the VirtualBox NAT Network.
4. Virtual Machine and Network Setup
1.	In Oracle VirtualBox, create a NAT Network for the lab and configure its IPv4 network as 192.168.10.0/24.
2.	Enable DHCP if dynamic addressing is required, and note the actual gateway and DHCP range.
3.	Create four VMs: Ubuntu Server, Windows Server, Windows 10, and Kali Linux.
4.	Connect each VM's network adapter to the same NAT Network.
5.	Install the operating systems from their official installation media and update them.
5. Ubuntu Server – Splunk Enterprise
5.1 Configure a static IP
Identify the interface name with `ip -br address` and edit the appropriate Netplan YAML file. The interface name and gateway below are examples.
sudo nano /etc/netplan/00-installer-config.yaml

**network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: no
      addresses:
        - 192.168.10.10/24
      routes:
        - to: default
          via: 192.168.10.1
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]**

sudo netplan try
sudo netplan apply
ip -br address
Use the actual VirtualBox NAT Network gateway instead of assuming it is 192.168.10.1. In a domain lab, Windows clients and the Domain Controller should use the AD DNS server (192.168.10.7), not public DNS, for domain name resolution.
5.2 Optional VirtualBox shared folder
sudo apt-get update
sudo apt-get install -y virtualbox-guest-utils
sudo reboot
sudo adduser ubuntu vboxsf
mkdir -p ~/shared
sudo mount -t vboxsf -o uid=1000,gid=1000 SharedFolder ~/shared
Replace `SharedFolder` with the exact shared-folder name configured in VirtualBox. Log out and back in after adding the user to the vboxsf group, if required.
5.3 Install and start Splunk
Download the Splunk Enterprise Linux package from the official Splunk website and transfer it to the Ubuntu VM. Install the downloaded .deb package:
sudo dpkg -i splunk-*.deb
sudo -u splunk /opt/splunk/bin/splunk start
sudo /opt/splunk/bin/splunk enable boot-start -user splunk
Complete the license acceptance and administrator account setup. Splunk Web is normally available at http://192.168.10.10:8000.
6. Windows Endpoint – Sysmon and Universal Forwarder
6.1 Configure Windows networking
Assign Windows 10 a unique address in 192.168.10.0/24, either through DHCP reservation or a static address outside the DHCP pool. Do not reuse the Domain Controller or SIEM address.
6.2 Install Splunk Universal Forwarder
Download the Windows Splunk Universal Forwarder from Splunk's official website. During installation, configure the deployment to send data to the Splunk server at 192.168.10.10 on TCP port 9997. Confirm that the Windows host can reach that port.
6.3 Install Sysmon
Download Sysmon from Microsoft Sysinternals and obtain a reviewed configuration such as the Sysmon Modular configuration maintained by Olaf Hartong. Review the configuration before applying it.
# Run in an elevated PowerShell window from the extracted Sysmon directory
.\Sysmon64.exe -accepteula -i .\sysmonconfig.xml

# Verify the service
Get-Service Sysmon*
Sysmon can record process creation, network connections, file and registry activity, and other endpoint events depending on the configuration. Its operational log is available under Applications and Services Logs > Microsoft > Windows > Sysmon > Operational.
6.4 Configure event collection
Create or edit the local inputs.conf file at:
C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf
[WinEventLog://Application]
index = endpoint
disabled = 0

[WinEventLog://Security]
index = endpoint
disabled = 0

[WinEventLog://System]
index = endpoint
disabled = 0

[WinEventLog://Microsoft-Windows-Sysmon/Operational]
index = endpoint
disabled = 0
renderXml = true
source = XmlWinEventLog:Microsoft-Windows-Sysmon/Operational
Keep local configuration separate from the vendor-supplied default configuration. Ensure the Universal Forwarder service account has the required permissions to read the selected event channels. Apply least privilege and follow Splunk's guidance for granting event-log access.
Restart-Service SplunkForwarder
Get-Service SplunkForwarder
7. Configure the Splunk Index and Receiving Port
6.	Sign in to Splunk Web at http://192.168.10.10:8000.
7.	Go to Settings > Indexes > New Index.
8.	Create an index named endpoint.
9.	Go to Settings > Forwarding and Receiving > Configure Receiving.
10.	Create a receiving port using TCP 9997.
11.	Confirm that the Ubuntu host firewall and VirtualBox network permit the required traffic.
A receiving port accepts data from forwarders; it does not by itself guarantee that events are correctly parsed or indexed. Validate the forwarder's outputs.conf, the receiving configuration, index existence, and Splunk internal logs if data is missing.
**8. Configure Windows Server as the Active Directory Domain Controller**
8.1 Set the server identity and network
Computer name: ADDC01
IP address:     192.168.10.7
Subnet mask:    255.255.255.0
Preferred DNS:  127.0.0.1
After installing AD DS and DNS, configure the server to use its own DNS service. Ensure the NAT Network gateway and any DNS forwarders are configured appropriately for external name resolution.
8.2 Install AD DS
12.	Open Server Manager > Manage > Add Roles and Features.
13.	Choose Role-based or feature-based installation.
14.	Select the local server.
15.	Select Active Directory Domain Services and add required features.
16.	Complete the wizard and install the role.
8.3 Promote the server to a Domain Controller
17.	Select the notification flag in Server Manager and choose Promote this server to a Domain Controller.
18.	Choose Add a new forest.
19.	Enter the root domain name akshay.local.
20.	Configure Domain Controller options and set a strong Directory Services Restore Mode password.
21.	Review the DNS, NetBIOS name, and database/log/SYSVOL paths.
22.	Run prerequisite checks and select Install. The server will restart.
After promotion, sign in using the domain administrator account, for example AKSHAY\Administrator. The AD database is commonly stored at C:\Windows\NTDS\NTDS.dit. Protect the database and its backups because they contain sensitive directory data, including password-verifier material.
8.4 Create Organizational Units and users
23.	Open Server Manager > Tools > Active Directory Users and Computers.
24.	Expand akshay.local.
25.	Right-click the domain and create an Organizational Unit named IT.
26.	Create another OU named HR if desired.
27.	Right-click the IT OU, select New > User, and create a dedicated lab user such as akshay.
28.	Create a separate test user for authentication exercises.
Use non-privileged test accounts for simulations. Do not use the built-in Domain Administrator account for routine testing.
**9. Install Monitoring on the Domain Controller**
Install Sysmon and Splunk Universal Forwarder on ADDC01 using the same general approach as the Windows 10 endpoint. Configure the forwarder to send to 192.168.10.10:9997 and collect the Security, System, Application, and Sysmon Operational channels into the endpoint index. Validate permissions and confirm data arrival before beginning attack simulations.
10. Join Windows 10 to akshay.local
29.	Set the Windows 10 preferred DNS server to 192.168.10.7.
30.	Verify DNS resolution for the domain and Domain Controller.
31.	Open System Properties > Computer Name > Change.
32.	Select Domain and enter akshay.local.
33.	Provide authorized domain-join credentials.
34.	After the welcome message, restart the Windows 10 machine.
35.	At sign-in, choose Other user and authenticate with the test domain account, for example AKSHAY\akshay.
# Useful checks from Windows PowerShell
ipconfig /all
nslookup akshay.local
nltest /dsgetdc:akshay.local
11. Controlled RDP Authentication-Failure Simulation
This exercise generates a small number of failed logon events against a Windows 10 test machine in the isolated lab. Perform it only against systems and accounts you own or are explicitly authorized to test.
11.1 Prepare the endpoint
•	Enable Remote Desktop only for the duration of the lab exercise.
•	Add the dedicated test user to the Remote Desktop Users group.
•	Allow inbound RDP through Windows Firewall only from the Kali VM or lab subnet.
•	Confirm that TCP 3389 is listening and that the endpoint is reachable.
11.2 Prepare Kali Linux
mkdir -p ~/adproj
cd ~/adproj
# If the RockYou list is compressed:
sudo gzip -dk /usr/share/wordlists/rockyou.txt.gz
head -n 20 /usr/share/wordlists/rockyou.txt > passwords.txt
Use a purpose-built short list of known incorrect test passwords for a failed-login exercise. Avoid using real users' passwords or large public password lists.
11.3 Verify RDP connectivity
nmap -sT -p 3389 192.168.10.20
Replace 192.168.10.20 with the Windows 10 endpoint's actual address. If the port is closed, troubleshoot RDP configuration and firewall rules before proceeding.
11.4 Generate a limited set of failed attempts
Hydra can be used in an authorized lab to validate that failed RDP authentication is logged. Keep the test small and controlled, and stop if the account lockout policy is triggered.
hydra -l akshay -P passwords.txt rdp://192.168.10.20
The command structure uses -l for one username, -P for the password file, and rdp:// to identify the RDP service and target. Replace the example IP with the endpoint address. Do not add a correct password if the specific goal is to generate failed logons.
**12. Analyze Authentication Events in Splunk**
Open Splunk Search & Reporting and search the endpoint index. Windows Security Event ID 4625 indicates a failed logon. The event can include the target account, logon type, failure status/substatus, source network address, workstation name, and timestamp.
12.1 Confirm event ingestion
index=endpoint
| head 20
12.2 Find failed logons
index=endpoint EventCode=4625
Depending on the event format and field extraction, the event code may appear as EventCode, EventID, or within the raw XML. Inspect a raw event if the search returns no results.
12.3 Count failures by source address
index=endpoint EventCode=4625
| stats count by IpAddress
| sort - count
If IpAddress is not extracted, inspect the raw event and use the actual field name present in your data, such as Source_Network_Address or IpAddress.
12.4 Visualize failures over time
index=endpoint EventCode=4625
| timechart span=1m count
12.5 Example detection for repeated failures
index=endpoint EventCode=4625
| bin _time span=5m
| stats count as failures by _time, IpAddress, TargetUserName
| where failures >= 5
| sort - failures
This is a simple starting point, not a complete brute-force detection rule. Tune the threshold to the environment and account for NAT, shared workstations, service accounts, and legitimate authentication errors. Correlate with successful logons (Event ID 4624) and relevant RDP logon types where appropriate.
13. Atomic Red Team – Adversary Emulation
Atomic Red Team is an open-source framework for executing small, controlled tests mapped to MITRE ATT&CK techniques. It helps validate whether endpoint telemetry is generated, forwarded, searchable, and useful for detection.
Reference: https://github.com/redcanaryco/atomic-red-team and https://attack.mitre.org/.
13.1 Install Invoke-AtomicRedTeam
Follow the current official installation instructions. Run tests only in the isolated lab, review each test's prerequisites and cleanup commands, and keep endpoint protection enabled.
# Temporary execution-policy change for this PowerShell process only
Set-ExecutionPolicy Bypass -Scope Process -Force

# Follow the official Invoke-AtomicRedTeam installation instructions:
IEX (IWR 'https://raw.githubusercontent.com/redcanaryco/invoke-atomicredteam/master/install-atomicredteam.ps1' -UseBasicParsing)
Install-AtomicRedTeam -GetAtomics
The tests are commonly stored under C:\AtomicRedTeam\Atomics. Check the official repository for current installation syntax and dependencies, as commands and prerequisites can change.
13.2 T1136.001 – Create Account: Local
MITRE ATT&CK T1136.001 describes creation of a local account. This test can help validate monitoring for account creation. Review the selected Atomic test before running it, especially because it creates a local account.
Invoke-AtomicTest T1136.001
After execution, inspect Windows Security logs for account-management events, such as Event ID 4720 (a user account was created), and inspect Sysmon or process telemetry if configured. Search in Splunk:
index=endpoint (EventCode=4720 OR NewLocalUser)
A test may create an account with a specific name depending on the selected atomic and its parameters. Confirm the test output and search for the actual account name rather than assuming it is always NewLocalUser.
13.3 T1140 – Deobfuscate/Decode Files or Information
T1140 covers decoding or deobfuscating files or information. Some Atomic tests use utilities such as certutil to encode or decode test files.
Invoke-AtomicTest T1140
Review the test output and the process-creation telemetry. Sysmon Event ID 1 can record process creation and command-line arguments when configured. A search for certutil may help:
index=endpoint certutil
| table _time host user EventCode Image CommandLine
| sort - _time
If no event appears, check whether Sysmon Event ID 1 is enabled in the configuration, whether the forwarder is collecting the Sysmon Operational channel, and whether the event reached the endpoint index. Absence of a result does not prove that the activity was not logged locally.
13.4 T1201 – Password Policy Discovery
T1201 describes discovery of password policy. The Atomic tests may query local or domain password policy information.
Invoke-AtomicTest T1201
Review the selected test's output and expected telemetry. Depending on the test, useful evidence may include process creation, command-line activity, PowerShell logging, or other Windows events. Search for the executed commands and process names rather than expecting a single universal event.
index=endpoint (EventCode=1 OR EventCode=4104)
| search "*net accounts*" OR "*Get-ADDefaultDomainPasswordPolicy*" OR "*password*"
| table _time host user EventCode Image CommandLine Message
| sort - _time
Event ID 4104 is PowerShell Script Block Logging and is available only when appropriately configured. Consider enabling and forwarding relevant PowerShell operational logs in a controlled way.
13.5 T1202 – Indirect Command Execution
T1202 describes abuse of utilities that can indirectly execute commands and potentially bypass restrictions on command interpreters. The Atomic tests may use utilities such as pcalua.exe, forfiles.exe, conhost.exe, or scriptrunner.exe.
Invoke-AtomicTest T1202
Inspect process-creation events and parent-child process relationships. Sysmon Event ID 1 is particularly useful when configured to capture process creation and command-line information.
index=endpoint EventCode=1
| search Image="*pcalua.exe" OR Image="*forfiles.exe" OR Image="*conhost.exe" OR Image="*scriptrunner.exe"
| table _time host user ParentImage Image CommandLine
| sort - _time
Field names can vary depending on parsing and data format. Inspect raw events and adjust the search to match the fields available in your Splunk deployment.
**14. SIEM Validation and Detection Engineering**
The exercises revealed an important distinction: executing a test successfully does not guarantee that Splunk can detect it. A complete monitoring pipeline requires the event to be generated, retained locally, collected by the forwarder, transmitted, indexed, parsed, and searchable.
•	Validate Sysmon configuration and confirm expected event IDs are generated locally.
•	Check Windows Event Viewer when an expected event is missing from Splunk.
•	Confirm the forwarder is running and has permission to read the relevant event channels.
•	Check outputs.conf, receiving port 9997, index name, and Splunk internal logs.
•	Collect additional telemetry where justified, such as PowerShell Operational logs and appropriate process-creation auditing.
•	Build and test detections using known-good lab activity before relying on them.
•	Document false positives, data gaps, test prerequisites, and cleanup steps.
15. Security Improvements and Future Enhancements
•	Configure appropriate domain password and account-lockout policies.
•	Use least privilege and separate standard user and administrative accounts.
•	Enable and monitor relevant Windows audit policies.
•	Forward Domain Controller security events and endpoint Sysmon telemetry to Splunk.
•	Develop detections for repeated failed logons, suspicious process execution, account creation, and unusual administrative activity.
•	Add dashboards for authentication failures, endpoint process activity, and account changes.
•	Test alert quality and tune thresholds to reduce false positives.
•	Consider adding a second Domain Controller, Group Policy baselines, Windows Defender telemetry, and centralized time synchronization.
•	Document remediation actions for each identified monitoring gap.
**16. Conclusion**
This Active Directory Home Lab project helped me gain practical experience in deploying and managing a Windows domain environment, configuring endpoint monitoring, and centralizing security logs using Splunk Enterprise.
By integrating Sysmon and Splunk Universal Forwarder, I created visibility into endpoint and authentication activity. I used Kali Linux and Hydra to generate controlled RDP authentication failures and investigated the resulting Windows Event ID 4625 records in Splunk.
I also used Atomic Red Team to emulate selected MITRE ATT&CK techniques, including T1136.001 (Create Account: Local), T1140 (Deobfuscate/Decode Files or Information), T1201 (Password Policy Discovery), and T1202 (Indirect Command Execution). These exercises helped me understand the importance of validating telemetry coverage and identifying detection gaps.
The next phase of this project would focus on detection engineering: improving log collection, building correlation searches and dashboards, validating alerts, and documenting remediation steps to strengthen the security posture of the lab environment.
**17. Tools and Technologies**
•	Oracle VirtualBox – Virtual machine and isolated network setup
•	Windows Server – Active Directory Domain Services and DNS
•	Windows 10 – Domain-joined endpoint
•	Ubuntu Server – Splunk Enterprise SIEM
•	Splunk Universal Forwarder – Log forwarding
•	Sysmon – Windows endpoint telemetry
•	Kali Linux and Hydra – Controlled authentication testing
•	Atomic Red Team / Invoke-AtomicRedTeam – Adversary emulation
•	MITRE ATT&CK – Adversary technique mapping
•	PowerShell and Splunk SPL – Administration and log investigation
**18. Important Lab Notes**
•	All attack simulations must remain within systems and accounts you own or are authorized to test.
•	Use unique IP addresses and verify the actual VirtualBox NAT gateway and DHCP range.
•	Use the Domain Controller's DNS service for domain-joined Windows clients.
•	Do not disable Microsoft Defender globally or exclude the entire C: drive to run Atomic Red Team.
•	Review each Atomic test's prerequisites, expected behavior, impact, and cleanup instructions before execution.
•	Hydra syntax, Splunk field extractions, Sysmon configuration, and Atomic test parameters may vary by version. Validate against the versions installed in the lab.
•	Do not claim a detection succeeded unless the expected event was actually received and validated in Splunk.
