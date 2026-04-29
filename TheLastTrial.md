# The Last Trial

Room link [here](https://tryhackme.com/room/thelasttrial)

1. What was the website from which the user downloaded the malicious application's installer?
*Answer*: developai.thm 
* Use ```python3 mac_apt.py DD /home/ubuntu/Lucas_Disk.img SAFARI -c -o /home/ubuntu/evidence/``` and look for the DOWNLOAD event in the SAFARI CSV. 

2. What is the name of the malicious application's installer?
*Answer*: DevelopAIInstaller.pkg
* Use ```python3 mac_apt.py DD /home/ubuntu/Lucas_Disk.img RECENTITEMS -c -o /home/ubuntu/evidence/recentitems``` and look for recent documents. 

3. When was the malicious application installed in the system?
*Answer*:2025-07-04 10:09:03
* Use ```python3 mac_apt.py DD /home/ubuntu/Lucas_Disk.img INSTALLHISTORY -c -o /home/ubuntu/evidence/installhistory/``` and look for installers! 

4. Which TCC permission did the application request first?
*Answer*: kTCCServiceSystemPolicyDesktopFolder
* Use ```python3 mac_apt.py DD /home/ubuntu/Lucas_Disk.img TCC -c -o /home/ubuntu/evidence/tcc/``` and sort them based on date. 

5. What is the full C2 URL to which the application exfiltrated data?
*Answer*: http[:]//c7.macos-updatesupport.info[:]8080
* Look at the LaunchAgent for DevelopAI.sh and find the C2 URL. 

6. Which persistence mechanism did the application use?
*Answer*: LaunchAgents 
* Use ```python3 mac_apt.py DD /home/ubuntu/Lucas_Disk.img AUTOSTART -c -o /home/ubuntu/evidence/autostart/``` and look for strings with DevelopAI. 