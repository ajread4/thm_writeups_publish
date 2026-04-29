# Shock and Silence

Room link [here](https://tryhackme.com/room/shockandsilence)

1. What is the full URL from which the ransomware was downloaded to the system?
*Answer*:https[:]//store5.gofile.io/download/web/e23cb33f-0e4d-4a5f-8c55-ea2d78057d40/HiddenFile.zip
* Look within the $MFT for Zone IDs that contain https. It singled out this one! $MFT analyzed using Eric Zimmermans tools (MFTECmd). 

2. What was the original file name of the ransomware executable downloaded to the host?
*Answer*: pb.exe
* Found within the $MFT in a Hidden Directory using Timeline explorer and the output of MFTECmd. Matched the creation dates with the expected activity dates and identified the suspicious executable. 

3. Which executable file initiated the encryption process on the system?
*Answer*: HpAgent.exe 
* Found within $MFT and $J, look for file creation or rename around the time that the zip file was downloaded. I did some research on the executable and saw that it is not a common executable, which raised my suspicion and interest. 

4. What file extension was appended to the encrypted files?
*Answer*: EeUfy
* Found within the $MFT output using Timeline Explorer. A ton of files showed up and appeared to be linked to user data folders. It also clearly is interacting with all major directories and locations on the local host. 

5. Go beyond the obvious - which ransomware group targeted the organisation?
*Answer*: BlackLock 
* Based on the ransomware note format and terminology! This was hard to find! A lot of ransomware notes online matched the style, demands, and setup. 