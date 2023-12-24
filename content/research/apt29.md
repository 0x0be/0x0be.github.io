+++
date = "2023-01-01"
title = "The curious case of SberAuto spear-phishing campaign"
slug = "the-curious-case-of-sberauto-spear-phishing-campaign"
+++



### Summary
On December 19, 2022 malware researcher known as **StopMalvertisin** twitted about a possible APT29 attack.

![Alt text](image-0.png)

The spear-phishing campaign targets **SberAuto**, a Russian online car trading platform associated with the state-owned banking and financial services company **Sberbank**.  
The analyzed attack displayed similar **TTPs** commonly attributed to **APT29** (aka **Cozy Bear**), even though it is unclear why a Russian-backed hacking group should be targeting a domestic web service.


## Technical Analysis
### Initial Stage
The first stage of this attack is represented by an **ISO** file (0b32bd907072d95223e5eb2dc5e3d9e0) named "Алкоголь_2023_zip.iso", uploaded on VirusTotal from Russia on December 19, 2022 and potentially delivered as an email attachment.

![Alt text](image-1.png) 

The archive content closely resembles the one of previous APT29 campaigns.   

![Alt text](image-2.png)


The only folder visible item is a shortcut file disguised as "Алкоголь_2023.pdf". 

```cmd
%windir%/system32/cmd.exe /c start update.exe & "%ProgramFiles(x86)%/Microsoft/Edge/Application/msedge.exe" %cd%/alcohol.pdf
```

Once clicked on the **LNK** file, **update.exe** is firstly executed, followed by the lure PDF document called "alcohol.pdf", which displays the alcohol catalog from the Russian market chain called **Globus Gourmet**.   

![Alt text](image-3.png)

The files "thumbcache.dll" and "update.exe" are actually two legit *signed binaries* of **Microsoft OneDrive**: the latter exploits Windows **search order hijacking** to load the malicious DLL named "version.dll," which has been modified by the threat actor to load an encrypted payload file.

![Alt text](image-4.png)


### Second stage: version.dll
The dynamic-linked library "version.dll" is a 64-bit DLL which is side-loaded by the legit Microsoft OneDrive binary.  
The *compile-timestamp* shows **Sunday, December 18 2022**. 

![Alt text](image-5.png)

The binary first employs ```GetComputerNameExA``` API function to retrieve the hostname of the infected machine and check whether it is equal to ```corp.sberauto[.]com```, an online russian service facilitating car sales website.
If the hostname does not match, the program terminates.

![Alt text](image-7.png)

After that, the malicious DLL iterates through the running processes using ```Process32Next``` to find the ID of **explorer.exe** and obtain its process handle.

![Alt text](image-8.png)

It then creates a *suspended-process* called **RuntimeBroker.exe** using ```CreateProcessA``` and sets "explorer.exe" as the parent process via ```UpdateProcThreadAttribute``` API.

The attribute parameter with the value 0x20007 corresponds to the definition of **PROC_THREAD_ATTRIBUTE_MITIGATION_POLICY**, while the value ```0x100000000000``` corresponds to **PROCESS_CREATION_MITIGATION_POLICY_BLOCK_NON_MICROSOFT_BINARIES_ALWAYS_ON**: it is typical for **CobaltStrike** to use CreateProcess API call along with a **STARTUPINFOEX** structure containing this mitigation policy to block DLLs that are not signed by Microsoft.

![Alt text](image-9.png)


Finally, the DLL decrypts the shellcode into "RuntimeBroker.exe" through ```NtAllocateVirtualMemory``` and ```NtWriteVirtualMemory```, sets it to an executable **APC routine** and run it via the ```NtAlertResumeThread```.
All this function API calls are done directly via **syscall** to hinder analysis. 

![Alt text](image-10.png)

The presence of specific debug strings shows that the program was created with **Shhhloader** framework, a "shellcode loader that takes raw shellcode as input and compiles a C++ stub that does a bunch of different things to try and bypass AV/EDR".  

![Alt text](image-11.png)


### Final stage: Cobalt Strike payload
The APC inject shellcode represents a **Cobalt Strike backdoor**, as evidenced by the multiple signature matches of the THOR APT Scanner.  
Firstly, the shellcode finds the "MZ" and "PE" header in the memory.

![Alt text](image-12.png)

Then it links the following Windows API function at **runtime** from parsing **PEB_LDR_DATA** to avoid detection.
* GetProcAddress
* GetModuleHandleW
* LoadLibraryW
* VirtualAlloc
* VirtualProtect

![Alt text](image-13.png)

![Alt text](image-14.png)

After allocating a **RWX** protected memory portion, it injects the shellcode into "RuntimeBroker.exe" process by calling once again the same initial entry function.

![Alt text](image-15.png)

 To extract the C2 configuration of **Cobalt Strike beacon**, SentinelOne CobaltStrikeParser Python script comes in handy. 

 ```bash
 python parse_beacon_config.py --json version.dll
 ```

```json
{
  "BeaconType": [
    "HTTPS"
  ],
  "Port": 443,
  "SleepTime": 46000,
  "MaxGetSize": 1398924,
  "Jitter": 32,
  "MaxDNS": "Not Found",
  "PublicKey": "MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQCSrC60Ubq+U90iLmHldoLVFW6Bc7vLsQ12BXGcc2c8TQJbnaf8I9dm/dhdZPEoCwQKRbjD/2xlR4Vr/S7IGj1Sh8gKHfJXh96lIhR5W85/+Fdi0weqGbrx9mbu70Ir7bA0ar1vwK17RFIla7B24ffVWNTfsO4fuagDSmR6MSKK2wIDAQABAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA==",
  "PublicKey_MD5": "ad07d632e310a66efeb503ee9089ad64",
  "C2Server": "adblockext.ru,/functionalStatus/hw7s8TE4f9GtrBHb8iiFT7RyIAuN",
  "UserAgent": "Mozilla/5.0 (Windows NT 6.3; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4433.0 Safari/537.36",
  "HttpPostUri": "/rest/2/meetingsfR4TG1Qpai0Oa1Q5fNxdUoAAfFn1"
}
```

The Cobalt Strike beacon uses **Base64 scheme** to communicates with the C2 domain ```adblockext[.]ru``` over **HTTPS**.


## Conclusion
This sample is interesting because a Russian-backed hacking group targeting a domestic website raises some suspicions.  

The similarities with previous APT29 campaigns (i.e., he use of ISO files containing binaries vulnerable to DLL hijacking) may lead to a couple of hypotheses about what this campaign could be:
* An internal Red Teaming exercise, aimed at enhancing cybersecurity measures inside SberAuto employees;
* An attack orchestrated by Ukrainian groups (particularly, The IT Army of Ukraine), trying to simulate Cozy Bear TTPs.  






























