---
title: A Beginner's Guide to the OSCP
---

I recently passed the OSCP after a month of intense preparation! Here is the structure I would recommend using to prepare for the exam:
1. Review the exam guidelines
2. Read through the PEN-200 course material, but don't fall into tutorial hell
3. Work on the PEN-200 Challenge Labs to get a feel for the exam
4. Work through PG Practice boxes
5. Work through HTB boxes

In this blog post, I will attempt to walk through each of these steps, and aim to help you develop a clear mental model, which should hopefully make it easier for you to get started on your exam preparation journey. I try to assume as little prior knowledge as possible. I do hold a degree in computer science, which may have helped me to absorb the course material more quickly, but by no means is it necessary.

### 1. Exam Guidelines

The exam has the following format:
- Active Directory set (40p: 10p for Box 1, 10p for Box 2, 20p for Domain Controller)
- Standalone box 1 (20p: 10p for initial access, another 10p for privilege escalation)
- Standalone box 2 (20p: 10p for initial access, another 10p for privilege escalation)
- Standalone box 3 (20p: 10p for initial access, another 10p for privilege escalation)

Active Directory is just terminology for a group of networked Windows machines. The standalone boxes can be either Windows or Linux.

The first 24 hours are for solving the boxes and will be proctored. **70 points** are required to pass. The subsequent 24 hours will be for report writing, and VPN access to the exam environment will be switched off, so you'll need to get used to screenshotting your work while doing it. This can be difficult when you're going down rabbit hole after rabbit hole looking for the solution. For the report, just refer to Offsec's official template. I got 90 points, and my report was about 50 pages long including screenshots.

**Initial access and privilege escalation**

To better understand what initial access is, imagine you're visiting a website, `http://www.abc.com`. This website is being served from a machine somewhere in the world. Machines have the concept of ports. These ports are virtual and run from `0` to `65535`. The idea is that each port can speak a different language/protocol and can be used for a different purpose. Websites are usually served from the default `http` port `80`, or `https` port `443`. If we wanted to visit `http://www.abc.com`, our machine would send a request to the host machine's port `80`. 

Another very important concept is the shell. A machine has a shell/terminal which can be used to issue commands to it. For example, we can use the shell to programatically move files around our Desktop. One way to access a machine's shell is to use the `ssh` protocol, which has a default port of `22`. If a machine decides not to "open" port `22`, then `ssh` will not be available.

The concept of initial access refers to gaining a shell on a machine. There are a number of ways to do this: you could get hold of some `ssh` credentials and exec in, but more often access is gained via a technique known as reverse shell, which is running a command/injecting a payload to trick the machine to connect back to us, and we'll have a listener ready to accept that connection. A successful initial access means being able to locate a `local.txt` flag on an unprivileged user's Desktop.

Once we're in, privilege escalation means taking an unprivileged user and either escalating them to root/admin privileges, or finding a way to switch to the `root` user. For Windows boxes, the `proof.txt` flag will be in `/Users/Administrator/Desktop` and for Linux boxes it is usually in `/root`.

**Flag requirements**

The full set of exam instructions can be found [here](https://help.offsec.com/hc/en-us/articles/360040165632-OSCP-Exam-Guide). For the flag specifically, ensure you also run `pwd` and `ifconfig`/`ipconfig`/`ip a` and `whoami` and include everything in a single screenshot. Get used to doing this during your practice, so you don't forget to do it during the exam. 

**Restrictions on allowed tooling**

The OSCP restricts the use of auto-exploitation tools. Tools like Metasploit are taught in the course, but it's important to know that you're only allowed to use it on one box during the exam and thus you should aim to not develop a reliance on the tool during your practice. Personally, I completed the course and labs without touching Metasploit at all. There are some fine-grained details that are useful to keep at the back of your mind. They may not make much sense now but will as the course goes on, which is why I am recommending familiarising yourself with the exam guidelines before even starting on the course material. For example, Metasploit has a payload generator module called `msfvenom` which you're allowed to use liberally.

Other tools that are similarly restricted are tools like `sqlmap`, which automates the process of SQL injection. Again, read the exam instructions for the full list.

The OSCP does not restrict the use of auto enumeration tools. Enumeration is the process of discovering what's running and what's configured as what on a system. This can be anything from ports and running services to configured permissions. The OSCP is frequently described as an "enumeration exam", and really 90% of it is about enumerating as thoroughly as possible -- the exploits themselves are usually elementary once you've discovered the right attack surface. The most fundamental enumeration tool is `nmap`, and you're allowed to use its scripting engine. Other tools like `WinPEAS` and `LinPEAS` are taught in the course and recommended for the exam as they help to automate a lot of the enumeration stuff. I'll talk about this in more detail later. You can also prepare your own scripts, though I didn't due to laziness.

### 2. PEN-200 course overview

The course is divided into a number of modules, but not all of them are required for the exam. 

The course begins by covers initial access techniques. You will learn to use `nmap` to enumerate common ports. There are actually quite a number of things to take note for each port. For example, if you find `snmp` (usually udp port 161), you should look at `nsExtendObjects`, OffSec likes to hide stuff there. See the [SNMP entry](https://hacktricks.wiki/en/network-services-pentesting/pentesting-snmp/index.html) on HackTricks.

Afterwards, there is a slightly deeper dive into web-based techniques. Broadly, they are:
- Cross-site scripting (XSS)
- File inclusion
- Directory traversal
- SQL injection

These techniques can also be used later on for privilege escalation but more likely they are used to gain an initial foothold. If any of these techniques are involved in the solution, the attack path is usually qutie elementary. Don't expect some super complicated SQL injection, for example. 

Next, there are modules on **locating and using public exploits**. Often, the entrypoint could be a known vulnerability on the version of certain server software. Basically, it could be just a public exploit that already implements your XSS/file inclusion for you. Slight modifications may sometimes be needed to redirect the exploit to the right IP/port of your Kali machine so your listener can catch the reverse shell. There may be a need to cross-compile the executable. There may also be a need to adjust some syntax to suit new Python versions. Don't overlook this section. There are subsequent Offsec courses designed to teach you how to write payloads and exploits from scratch but this course does not require that. Instead, the focus is on knowing how to use existing exploits and building a confidence in doing so. As such, you should expect to see it fairly frequently on this course. 

A successful initial foothold usually concludes with a listener on our Kali machine catching a reverse shell from the target machine. Learn how to do this. For reverse shell payloads, [revshells.com](http://revshells.com/) and the `msfvenom` payload generator will be your good friends.

The focus next shifts to privilege escalation. Broadly, this can be divided into:
- Windows privilege escalation
- Active Directory
  - Setting up a tunnel
  - Privilege escalation and lateral movement techniques
- Linux privilege escalation

Let me now explain each of these in further detail. Note that exact commands have been omitted as this is meant to be a high-level guide.

#### Windows privilege escalation

We will first focus on local privilege escalation techniques for Windows, outside of AD.

**Privileges**

We can often start by looking at the privileges the user has been granted and look for a few that perhaps shouldn't be there:
- An account with `SeImpersonatePrivilege` can abuse the [SigmaPotato](https://github.com/tylerdotrar/SigmaPotato) or [PrintSpoofer](https://github.com/itm4n/printspoofer) exploits to privesc to SYSTEM. The underlying mechanism involves tricking a higher-privileged process running as SYSTEM into authenticating to a listener you control. You then catch the SYSTEM token and `SeImpersonatePrivilege` lets you use that token, allowing you to spawn a process as SYSTEM.
- `SeBackupPrivilege` allows us to dump the `SYSTEM` and `SAM` registry hives, which together contain local account password (NTLM) hashes. NTLM is an authentication mechanism used for Windows. It is elaborated later in the AD section.
- `SeShutdownPrivilege` allows us to restart a machine. 
- `SeRestorePrivilege` allows us to write to any file regardless of access control lists.

**Services**

Sometimes, the privileges on their own will not be sufficient for privilege escalation. For instance, `SeShutdownPrivilege` usually needs to be paired with some other thing we have planted. This leads us to the general technique of tampering with services. There are a few ways to tamper with services. The core thread is that **a privileged process loads or executes something from a location you can influence.**
- If you can write to the actual `.exe` or `.dll` the service loads, we can just replace it. For example, if we have `SeRestorePrivilege`.
- If not, we can check the path to the `.exe` that the service uses. If it is unquoted and with spaces, such as `C:\Program Files\My App\Service Folder\app.exe`, Windows will try `C:\Program.exe`, then `C:\Program Files\My.exe`, then `C:\Program Files\My App\Service.exe`, and so on. If one of these paths is writable, we can drop our malicious `.exe` there.
- Similarly, we may be able to exploit the search order of DLLs that the service loads to inject a malicious one.

Clearly, all these require some trigger. It could be a:
- **scheduled task**, the Windows equivalent of a cron job
- service that we have privileges to restart
- service that restarts on reboot and we have `SeShutdownPrivilege`

As another example, what can we do with the NTLM hashes obtained from `SYSTEM` and `SAM`? We should understand that there are a few core Windows services that allow remote access. They are `winrm`, `wmi`, `rdp`. RDP in particular gives us a UI on the system, which can be nice. A technique known as **pass-the-hash** can allow us to authenticate to one or more of these services, and gain a shell.

> [!note] **Pass the hash**
> - NTLM authenticate to a service by passing in the IP.
> - Supported by all of the common services like ldap (389), smb (445), winrm (5985), mssql (1433)

However, note this mechanism is a lot more common in AD environments because the scenarios where you have enough access to dump hashes but not enough to just become SYSTEM directly are rare. In an AD environment, if we can compromise one machine and dump its local admin (or other domain user) hashes, the same local admin credentials may be used in another machine.

**Looking for credentials**

If none of the above vectors is present, we should take some time to look for credentials hidden on the machine:
- Look for any interesting files, especially in `C:\Users`. Check if anything is hidden in files' metadata. For example, `exiftool file.pdf` on Kali.
- Look for databases, especially if database services are running on the machine.
- Look at PuTTY, a SSH/Telnet client for Windows that may store credentials.
- Look for autologon creds.
- Look at PowerShell history.
- Look at Windows Credential Manager.
- Look inside any config or log files.

This step is the most painful but Offsec loves it, so be patient, thorough and systematic.

**Other: `AlwaysInstallElevated` policy**

Policies are system-level configurations. For example, "Disable USB storage" could be a policy. One policy that we can abuse is `AlwaysInstallElevated`. This allows any `.msi` installer to run as SYSTEM, even if launched by an unprivileged user. We could generate a `.msi` reverse shell using msfvenom and run it.

**WinPEAS**

There is an automated enumeration tool, [WinPEAS](https://github.com/peass-ng/PEASS-ng/blob/master/winPEAS/winPEASbat/winPEAS.bat), which helps with identifying some of these. It's suggested to run it at the very start but don't over analyse the output. Also, learn to recognise false positives.

#### Active Directory

**Authentication mechanisms**

Unlike standalone Windows machines which use only NTLM authentication, Active Directory services often use another type of authentication known as *Kerberos*. A high-level understanding of these two mechanisms is thus required.

NTLM uses a challenge-response mechanism. It is older, and weaker. It is used as a fallback when Kerberos isn't available.

Kerberos involves a trusted third party, the Key Distribution Center (KDC), which runs on the Domain Controller. The flow is simple:
1. Client authenticates to KDC, gets a Ticket Granting Ticket (TGT).
2. When client needs to access a service, they present the TGT to get a Service Ticket, then present that to the service. 

Step 2 is contingent on the service having a *Service Principal Name* (SPN) on the domain. Without a SPN, Windows will fall back to trying NTLM authentication on the service.

**AD environment taxonomy**

There are three machines in the exam AD environment. Machine 1 is the starting point. We are given an initial set of domain credentials and this should work with at least one of the remote access services on Machine 1.

After Machine 1, we usually need to *laterally* move to Machine 2, and then finally reach the *Domain Controller*, Machine 3. Machine 1 is generally directly accessible by Kali (in the same subnet) but Machines 2 and 3 are in the AD subnet. Thus, to make Machines 2 and 3 accessible from our Kali machine so we can run commands against them more easily, we need a technique known as **tunneling** and we are fortunate to have two tools, [Chisel](https://github.com/jpillora/chisel) and [Ligolo-ng](https://github.com/nicocha30/ligolo-ng), that automate this very well. Personally, I used Ligolo-ng. 

**LDAP**

The initial set of domain credentials given should work on the LDAP port on the Domain Controller. LDAP is the AD protocol and will be able to reveal lots of key information. We would inject a *collector*, SharpHound, and analyse the results collected back on our Kali using BloodHound.

**Constructing the attack chain**

We can now commence our discussion on the attack chains used in AD. Since Active Directory is just terminology for networked Windows machines, the techniques for privilege escalation explained above for Windows machines all apply. In general, we need to compromise Machine 1 first. Then make use of these admin privileges to *laterally* move to Machine 2. Repeat, until we reach Machine 3, the *domain controller*. The latter two steps may be different depending on what our BloodHound output is. 

Suppose we have compromised Machine 1. How do we use admin privileges to get useful information that we could use to move to Machine 2? 
- We can usually run [Mimikatz](https://github.com/ParrotSec/mimikatz). Mimikatz automates the extraction of credentials:
  - We can dump credentials from LSASS memory (the running process that handles authentication). This gets plaintext passwords (on older systems), NTLM hashes, and Kerberos tickets for currently/recently logged-in users. 
  - We can also dump the local SAM database.

After we have dumped the credentials on Machine 1, it's a good time to look at what we've managed to get and also take a look at the BloodHound output, and try to come up with some sort of coherent idea of what the attack chain is likely to be. Some examples of things BloodHound flags are:
- `GenericAll` - full control over an AD object. Can change password, set SPN to Kerberoast, modify group memberships, and many others
- If a user has "Do not require Kerberos pre-authentication" enabled - AS-REP roast

> [!note] **AS-REP roasting**
> If a user has "Do not require Kerberos preauthentication" enabled, anyone can request a ticket for that account from the KDC without proving identity. The response contains data encrypted with the user's password hash and we can crack it offline. 

>[!note] **Kerberoasting**
> Any authenticated domain user can request a service ticket (TGS) for any account with an SPN. The ticket is encrypted with that account's password hash, crack it offline. We can also use the obtained credentials to forge a **silver ticket**, which allows us to access a service with any privileges we want.

Example attack chain:
- Bloodhound shows `svc_db` has an SPN → Kerberoast it → crack the password
- `svc_db` has `GenericAll` over Domain Admins group → add `svc_db` to `Domain Admins`
- Domain Admins by default have RDP access, so RDP to the domain controller as `svc_db`.

The attack chains aren't limited to these, but they always look something like this. Remember, frequently simplify and zoom out.

**On the Domain Controller**

Let's say we have a weak foothold on the DC. Domain Controllers have a database file, `ntds.dit`, which stores all of the domain creds. If we have `SeBackupPrivilege`, we can retrieve the contents of this file, and use the retrieved hash to log in as Domain Admin. We can also retrieve the `krbtgt` hash, the hash of the service account that the KDC uses to sign TGTs.

There could be two domains in a parent-child relationship. We can use the **golden ticket** attack in combination with **DCSync**. It works as follows:

>[!note] **Compromising the parent domain**
> - DCSync is used to dump all domain creds by "pretending" to be a domain controller and asking the real domain controller for creds. It requires the calling account to have *replication permissions* on the domain. Domain Admins have this right by default. So we can use DCSync after we become domain admin. This allows us to get the `krbtgt` hash.
> - We can then use the `krbtgt` hash to create a **golden ticket** to the parent domain, which we do not yet have replication rights on. It allows us to say "I'm an enterprise admin in the parent", and then we can run rogue.

#### Linux privilege escalation

Now, on to Linux. The key techniques for Linux are:
- credentials hidden in files. In particular, check `/etc` and `/opt`.
- `suid` binaries
  - The `suid` bit runs a binary with root permissions. Certain binaries if accidentally accorded this privilege can be abused. 
  - We can refer to [gtfobins](https://gtfobins.org/) for the vectors.
- misconfigured `sudo` permissions 
  - Similar to `suid`, if certain commands are allowed to run with `sudo`, they can be abused. For example, if I can run `vim` as root something has to be wrong. Also refer to gtfobins.
- running processes, specifically cron jobs 
  - Use [pspy64](https://github.com/dominicbreuker/pspy) to monitor. 
- service hijacking. This is essentially the same technique as in Windows, but with some extra ways to exploit it, such as tricks with [wildcards](https://hacktricks.wiki/en/linux-hardening/interesting-files-permissions/wildcards-spare-tricks.html).
- kernel exploits

To some extent, scripts like [linux-smart-enumeration](https://github.com/diego-treitos/linux-smart-enumeration) can help with identifying some of these. Another alternative is [linPEAS](https://github.com/peass-ng/PEASS-ng/tree/master/linPEAS) but I prefer `linux-smart-enumeration` for its gradual information reveal mechanism that is easier to parse. Do keep in mind just like WinPEAS, the tools are designed to suggest any potential vulnerability but not every vulnerability suggested by the tool will work. So you will need to gain an intuition as to what usually works. For example, I've found that public exploits for most of the identified kernel vulnerabilities fail to work, and so that's not the thing I usually try first.

#### Tooling

Some things to learn:
- You will frequently need to transfer files between Kali and your machines.
- You may need to learn how to upgrade a shell to be fully interactive, for both Linux and Windows.
- Some degree of familiarity with PowerShell and bash. Basically, don't need to be an expert, but don't be fumbling around in the exam either. 
- You may sometimes need `hydra` to brute force. Offsec likes password reuse and also using the same password as the username. If you get a list of users from BloodHound for example, spray it against every service you find. Also `netexec`, to spray one set of credentials against different services
- `hashcat` or `john` to crack hashes.
- How to exec into machines. `impacket-psexec` is the most common, from the `impacket` suite. Also `evil-winrm` for winrm and `xfreerdp` for RDP. Also, learn `ssh`. 
- `impacket-secretsdump` to extract hashes from SAM/`ntds.dit`

### 3. PEN-200 Challenge Labs

I started with OSCP A, which may seem quite ambitious, but I was short on time and wanted to get a feel for what the exam would be like. I allowed myself to use hints liberally. Afterwards, I did OSCP B, OSCP C, Medtech, Poseidon, Zeus, Secura, in a random order, trying to use fewer hints each time. I found the walkthroughs at this [Korean blogger's website](https://xn--ec6b17t.com/OSCP/Challenge-Labs/Challenge-4---OSCP-A) nice. I can't even read Korean but I was still able to understand the walkthrough, which is quite telling. I noticed that most of the boxes had strong focus on Active Directory and I was as a result quite nervous about initial access portion of the exam. This led me to move on to PG Practice labs.

### 4. PG Practice Labs

These were a tremendous resource. I used the very popular [TJNull list](https://docs.google.com/spreadsheets/u/1/d/1dwSMIAPIam0PuRBkCiDI88pU3yzrqqHkDtBngUHNCw8/htmlview) and worked through about 20 PG Practice boxes. I subsequently ran out of time so I just read through the walkthroughs of the remainder (readily available online on blogs), noting down the key points of each box. 

### 5. HackTheBox Labs

I've heard these are more challenging than the PG Practice Labs and so if you have more time I would highly recommend attempting them. I mean, it's always better to be overprepared.

### 6. Conclusion

I hope this blog post gave you a good understanding of what the OSCP is about and how to prepare for it. It is by no means comprehensive but provides a good high-level mental model you can use before commencing your preparation. All the best, and Try Harder!