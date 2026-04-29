# Lost in RAMslation

Room link [here](https://tryhackme.com/module/honeynet-collapse)

1. What is the absolute path to the initial malicious file executed on this host?
*Answer*: C:\Windows\Tasks\MicrosoftUpdate.dll
* Found the security-updat process which was interesting and backtracked using the cmdline and pstree output from Volatility to find the initial process. Can also find this activity looking at the console.txt output from the prepared files. You can clearly see the download of the exe. 
```
* 2680	conhost.exe	0x7ff7b930b828	_CONSOLE_INFORMATION.ScreenBuffer_0.Dump	N/A	[*] Listening on port 4443...
[*] Client connected.
[*] Received command: alive
[*] Sent alive response.
[*] Received command: alive
[*] Sent alive response.
[*] Received command: exfiltrate
[+] Hosts file exfiltrated to http://167.172.41.141/
[*] Received command: spawnchain
[!] Failed to launch secup1: 87
[*] Received command: dw_exec
[*] Downloading to: C:\Users\matthew.collins\Downloads\security-update.exe
[*] File downloaded successfully.
[*] Executed file. Waiting...
[*] Execution complete. Deleting file...
[*] File deleted.

```

2. Which process ID (PID) was assigned to the process used to execute the initial payload?
*Answer*: 2928
* From the pstree output of the answer above looking for the update exe. 

3. What was the full command line used by the attacker to launch initial execution on this host?
*Answer*: rundll32.exe  C:\Windows\Tasks\MicrosoftUpdate.dll, RunMe
* From the pstree output of the answers above with the PID 2928. 

4. The attack launched various processes. What is the name of the final process in the chain?
*Answer*: notepad.exe
* Looking at the full chain narrowing down on the initial processes with PID 1444 or 2676. From there, can see that the starting process was notepad. 

5. What are the first five bytes (in hex, e.g., 4d5a9000) of the Meterpreter shellcode injected into it?
*Answer*: fc4889ce48
* Look at the output of malfind for the notepad.exe process with PID 836. Malfind displays the initial bytes of any discovered malware. 

6. Which is the IP address that the hosts perform a lateral movement using port 3389?
*Answer*: 172.16.2.9
* Look within the netscan output to find traffic on port 3389: ```0xa68c461c42c0	TCPv4	172.16.8.15	49750	172.16.2.9	3389	ESTABLISHED	464	powershell.exe	2025-07-02 01:08:25.000000 UTC``` to the next machine based on the background given in the prompt. 