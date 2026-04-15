\# Kerberoast Mitigation — Wazuh Integration



\## Table of Contents

\* \[Introduction](#introduction)

\* \[Prerequisites](#prerequisites)

\* \[Installation and Configuration](#installation-and-configuration)

\* \[Integration Steps](#integration-steps)

\* \[Integration Testing](#integration-testing)

\* \[Sources](#sources)



\---



\## Introduction



This integration provides automated detection and response for Kerberoasting

attacks (MITRE ATT\&CK T1558.003) against Windows Active Directory environments.



Kerberoasting allows any authenticated domain user to request a Kerberos service

ticket for any SPN-registered account. The ticket is encrypted with the service

account password hash and can be cracked offline with no further network

interaction required after the initial request.



The only viable detection window is the ticket request itself: Windows Security

Event ID 4769. This integration filters on RC4-HMAC encryption type 0x17 (the

downgrade forced by Impacket and Rubeus), ignoring legitimate AES traffic

(0x11/0x12). Zero false positives against normal domain traffic.



When Rule 100002 fires at Level 12, Wazuh Active Response executes

kerb-block.ps1 on the Domain Controller, disabling the compromised service

account via Disable-ADAccount. MTTR: hours to under 2 seconds.



\---



\## Prerequisites



\- Wazuh Manager v4.x (tested on Wazuh Cloud v4.14.4)

\- Wazuh Agent active on the Windows Domain Controller

\- Windows Server 2019 Domain Controller with Active Directory Domain Services

\- Sysmon v15.15 deployed on the Domain Controller

\- RSAT (Remote Server Administration Tools) installed on the DC

\- PowerShell execution policy allowing scripts on the DC

\- Kerberos Service Ticket Operations auditing enabled via Group Policy:

&#x20;   Computer Configuration > Advanced Audit Policy > Account Logon

&#x20;   > Audit Kerberos Service Ticket Operations: Success and Failure

\- At least one SPN-registered service account in the domain



\---



\## Installation and Configuration



\### 1. Enable Kerberos Audit Policy



On the Domain Controller, open Group Policy Management Console (GPMC):

&#x20; Computer Configuration > Policies > Windows Settings > Security Settings

&#x20; > Advanced Audit Policy Configuration > Account Logon

&#x20; > Audit Kerberos Service Ticket Operations: Success and Failure



Enforce and verify:

&#x20; gpupdate /force

&#x20; auditpol /get /subcategory:"Kerberos Service Ticket Operations"

&#x20; # Expected output: Success and Failure



\### 2. Deploy the Detection Rule



Copy local\_rules.xml to the Wazuh Manager rules directory:

&#x20; /var/ossec/etc/rules/local\_rules.xml



Restart Wazuh Manager:

&#x20; systemctl restart wazuh-manager



\### 3. Deploy the Active Response Script



Copy kerb-block.ps1 to the Active Response bin on the Domain Controller:

&#x20; C:\\Program Files (x86)\\ossec-agent\\active-response\\bin\\kerb-block.ps1



Add to ossec.conf on the Wazuh Manager:

&#x20; <command>

&#x20;   <n>win-disable-user</n>

&#x20;   <executable>kerb-block.ps1</executable>

&#x20;   <timeout\_allowed>no</timeout\_allowed>

&#x20; </command>

&#x20; <active-response>

&#x20;   <command>win-disable-user</command>

&#x20;   <location>local</location>

&#x20;   <rules\_id>100002</rules\_id>

&#x20; </active-response>



Create audit log directory on the DC:

&#x20; New-Item -ItemType Directory -Path C:\\Security -Force



\---



\## Integration Steps



1\. Attacker requests a Kerberos ticket using Impacket GetUserSPNs,

&#x20;  forcing RC4-HMAC (0x17) encryption.

2\. Domain Controller generates Event ID 4769 with TicketEncryptionType 0x17.

3\. Sysmon v15.15 and Windows Security Auditing forward events to Wazuh Agent.

4\. Wazuh Agent forwards telemetry to Wazuh Manager.

5\. Rule 100002 matches EventID 4769 + EncType 0x17, fires at Level 12.

6\. Wazuh Active Response triggers kerb-block.ps1 via stdin on the DC.

7\. Script parses alert JSON, calls Disable-ADAccount, writes to SOAR.log.



\---



\## Integration Testing



\### Simulate the Attack



From a Kali Linux machine on the same subnet:

&#x20; impacket-GetUserSPNs DOMAIN/username:password -dc-ip DC\_IP -request



\### Verify Detection



Check Wazuh Manager for Rule 100002 firing at Level 12 (MITRE T1558.003).



\### Verify Automated Response



On the Domain Controller:

&#x20; Get-ADUser sql\_service | Select-Object Name, Enabled

&#x20; # Expected: Enabled : False



&#x20; Get-Content C:\\Security\\SOAR.log

&#x20; # Expected: \[timestamp] - SUCCESS: Disabled account: sql\_service



\---



\## Sources



\- MITRE ATT\&CK T1558.003: https://attack.mitre.org/techniques/T1558/003/

\- Wazuh Active Response: https://documentation.wazuh.com/current/user-manual/

&#x20; capabilities/active-response/

\- Windows Event 4769: https://learn.microsoft.com/en-us/windows/security/

&#x20; threat-protection/auditing/event-4769

\- Full lab implementation and evidence:

&#x20; https://github.com/R-Williams-Security/Kerberoast-Detection-Lab

\- Portfolio write-up:

&#x20; https://sites.google.com/view/williamsransom-portfolio/project-page/

&#x20; active-directory-soar-automated-kerberoast-mitigation



