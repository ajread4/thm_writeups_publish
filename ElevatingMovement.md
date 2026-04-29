# Elevating Movement

Room link [here](https://tryhackme.com/room/elevatingmovement)

1. When did the attacker perform RDP login on the server?
*Answer*: 2025-06-30 16:33:18
* Use: ```.\EvtxeCmd\EvtxECmd.exe -f C:\Windows\System32\winevt\Logs\Microsoft-Windows-TerminalServices-LocalSessionManager%4Operational.evtx --inc 21,22,25,41 --csv C:\Users\Administrator\Desktop\Evidence\``` and find the initial connection using emily.ross account that you discovered in the previous room and with the background information provided for context. 

2. What is the full path to the binary that was replaced for persistence and privesc?
*Answer*: C:\Users\emily.ross\Documents\Coreinfo64.exe
* Look at the file system to find the file in the user Documents after finding that the "info check" scheduled task uses Coreinfo64.exe. In addition, can look at UserAssist in registry. The scheduled task can be viewed using the Task Scheduler application in Windows. 

3. What is the type or malware family of the replaced binary?
*Answer*: meterpreter
* Take the file hash of the Coreinfo64.exe and send to Virustotal: https://www.virustotal.com/gui/file/3ae834b3d24e56a5216ca69f8210c4ba92f4315197babeb6e34eacf06952c7de/detection

4. Which full command line was used to dump the OS credentials?
*Answer*: ```pcd.exe /accepteula -ma lsass.exe text.txt```
* Look within the powershell transcript that is saved in the administrator documents. They are base64 encoded commands. 

5. Using the stolen credentials, when did the attacker perform lateral movement?
*Answer* : 2025-06-30 19:47:14
* Look for PSEXEC execution within the prefetch files and match the date with the execution time using Eric Zimmermans prefetch tool. 

6. What is the NTLM hash of matthew.collins' domain password?
*Answer*: eb3d2de2f21b31933fb4a4fd7a7d314d
* Find the temp file with the dmp within C:\Windows\System32\ since the command was run in that folder and results saved in the folder as well. 