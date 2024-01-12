# Windows-Forensics

## Introduction

This project consisted of six main steps: lab preparation, the attack, data acquisition, disk analysis, memory analysis, and super timeline analysis. The steps involved using a very wide array of technologies implemented in virtual environments. The following technologies were used:
  - Oracle VirtualBox
  - Windows 10/Server 2019 VM
  - Sysmon
  - Ubuntu on Windows
  - Arsenal Image Mounter
  - KAPE (Kroll Artifact Parser and Extractor)
  - Event Log Explorer
  - RegRipper 3.0
  - Eric Zimmerman Tools
      - Amcache Parser
      - MFTECmd
      - PECmd
      - ShellBags Explorer
      - Timeline Explorer
  - Volatility3

## Lab Preparation

This was a simple step, I downloaded VHDs of Windows 10 and Windows 2019 Server, with the Windows 10 VM acting as the victim and the Windows 2019 Server VM serving as the forensic workstation. Most of the tools above were then installed on the forensic workstation after setting up Windows Defender exemptions on the folder that would be used to prevent any tool issues. Once all tools were installed, I took a snapshot of the forensic VM in case of any issues so I could revert back to a clean install of all the tools before any actions took place.

## The Attack

Another easy step, I installed Sysmon on the Windows 10 VM to have access to some extra information down the road, disable some Windows Defender features and then ran the Atomic Red Team executable, which is a collection of tests and fake malware. Whatever Atomic Red Team left behind is what I will be trying to find as the evidence of an attack throughout the project. 

![compromisedSystem](https://github.com/k-mcgeary/Windows-Forensics/assets/129139672/435cf999-fecc-481c-9dca-324d629a191b)


## Data Acquisition

After the Atomic Red Team executable was finished, I paused the VM so I wouldn't lose any data due to random processes activating and potentially writing over valuable information. This was followed by taking a snapshot of the victim VM to preserve the data. I then used the vboxmanage utility to export the raw file of the victim VM and created a hash of the file. While this "evidence" isn't changing hands, the hash would be used to verify the integrity of the file if it were to be exchanged with someone else.

![win10MemDump](https://github.com/k-mcgeary/Windows-Forensics/assets/129139672/63d6c066-9343-4e48-89b5-4db5a29444c4)


The next step was copying the after-attack snaphsot VDI into a VHD, once again using the vboxmanage utility. Just like the raw file from earlier, I created a hash of the VHD file for the purposes of file integrity.

With the VHD copy created, I was able to load it into Arsenal Image Mounter on my forensic station VM. This mounted it onto the F drive, allowing me to view the files from the victim VM. This process also created E and G drives but they didn't hold any information.

![mountVHDArsenal](https://github.com/k-mcgeary/Windows-Forensics/assets/129139672/f330fca7-68ff-4dfd-99c6-43526dbef185)

The final step of the data acquisition process involved the KAPE tool. This has a lot of options to pull different files from whatever source you're pulling from, but in this project I used the Kape Triage option on the F drive created earlier. This option grabbed a number of useful files that can be seen in KAPE's configuration files and placed them into a folder I had created.

![triageKape](https://github.com/k-mcgeary/Windows-Forensics/assets/129139672/ab704f4f-0932-484d-81dd-088682255223)

## Disk Analysis

On to the best part!

I uploaded the Windows Registry hives (DEFAULT, SAM, SYSTEM, SECURITY, and SOFTWARE) into Registry Explorer. This allows you to parse through the hives and look at the keys and subkeys that grab your attention. This can also provide some information on whatever you're looking at on the right side of the application, but the information will vary widely depending on the keys/subkeys highlighted.

![registryExplorer](https://github.com/k-mcgeary/Windows-Forensics/assets/129139672/a1ce04e4-9051-4b80-a685-b9876eb38c26)

After Registry Explorer, I utilized RegRipper. This tool allows you to parse hives quickly and efficiently. RegRipper has a number of plugins that everyone can find useful when collecting information. With this tool I was able to pull the computer name, version of Windows being used, timezone the user was in, network information, shutdown time, and the Windows Defender settings by parsing through the SYSTEM and SOFTWARE hives. Lastly, I was able to run the tool and export the results to text documents that I could look at for key information through the rest of the project. 

![regRipper](https://github.com/k-mcgeary/Windows-Forensics/assets/129139672/54eed9bd-fcc5-4e02-8cd2-463cdecca759)

The next step was to analyze ShellBags, which stores the entries of directories accessed by the user among other things. I used the information in the ShellBags to determine the user executed a malicious script (Atomic Red Team).

![shellBags](https://github.com/k-mcgeary/Windows-Forensics/assets/129139672/c63c49b9-ade1-46be-8e6b-c2a6987392d7)

Now it's time to find evidence of deleted files. This was accomplished by using the MFTEcmd.exe in the Eric Zimmerman tools to parse through the USN Journal file, which provides a persistent log of all changes made to files on the volume. I found the file I needed but discovered in had been overwritten when parsing through the $MFT file.

![deletedFileOverwritten](https://github.com/k-mcgeary/Windows-Forensics/assets/129139672/248ccaa4-0628-428a-adec-c1cf97e4138d)

Next, I moved back to the Registry Explorer so I could look at the BAM (Background Activity Monitor). The BAM provides information on executables performed by the user and is found within the SYSTEM hive. This information can also be found in the RegRipper text files from earlier.

![bam](https://github.com/k-mcgeary/Windows-Forensics/assets/129139672/761c55df-4a5d-4fca-b584-345e5e4e1d0d)

Looking at more executable information, I looked at prefetch files. These contain information about the source from which an executable was launched, how many times it executed, what files it touched, and when it was launched. I was able to find multiple powershell, command prompt, scheduled task, and mavinject executables as well as the Atomic Red Team executable.

![prefetchTimeline](https://github.com/k-mcgeary/Windows-Forensics/assets/129139672/a1156400-327b-456e-bd12-271116e690c7)

Moving on, it's time to find evidence of persistence. This was simple enough, I moved to the Ubuntu application I installed within my forensic VM and used the grep function on the $MFT output file looking for StartUp. This allowed me to find a "malicious" bash script left behind by the Atomic Red Team script.

![persistenceEvidence](https://github.com/k-mcgeary/Windows-Forensics/assets/129139672/7b6caf77-448d-4ea5-a3b5-51a4834a86ef)

Next was investigating the Windows Event Logs. In this section I found evidence of alterations to Windows Defender settings, installation of services, authentication events (attacker logging in), and PowerShell execution events. I also utilized Sysmon event logs to find malicious process creation, network connection, file creation, registry events, and DNS queries.

![exeInEventLogs](https://github.com/k-mcgeary/Windows-Forensics/assets/129139672/768ba734-a93c-4178-aa14-cccda0a72dea)
![powershellLogs](https://github.com/k-mcgeary/Windows-Forensics/assets/129139672/bfe7402e-fb46-4226-b8ab-a017066e01ec)

## Memory Analysis

This section was accomplished solely through the Ubuntu on Windows application using the Volatility3 tool and the VM memory raw file created earlier. Information that was extracted included OS/Kernel details, Windows processes/parent processes, evidence of injected DLLs, and malicious registry key entries.

## Super Timeline

Lastly, a super timeline was configured by creating a memory timeline with Volatility3 and creating a disk image timeline with Plaso tools and Log2Timeline. Those timelines will then be merged with the mactime parser tool to create a super timeline, which contains all of the information that was manually parsed out through the course of this project, This is a lot of information of course, but very valuable and convenient if you know what you're looking for. Using Timeline Explorer you will have access to the timestamp, source description, where it falls in MACB (Modified, Accessed, Changed(($MFT Modified)), Born), and a long description of the file.

## Conclusion

This was a very information dense project that taught me a lot not only about popular forensics tools and practices, but also gave me a look into the processes that Windows goes through that most aren't aware are even happening. The information written above isn't even close to the amount of stuff that was found within the material but I found it to be very well-delivered by a knowledgeable teacher. I was able to access this through the Practical Windows Forensics course found on TCM Academy and taught by Markus Schober. This was a great introduction into the world of Digital Forensics and I'm excited to dive into the subject more in the future.













