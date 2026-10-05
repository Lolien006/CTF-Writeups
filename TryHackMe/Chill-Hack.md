Chill Hack — TryHackMe

Platform: TryHackMe
Machine: Chill Hack
Difficulty: Easy
Target IP: 10.128.184.149
Attacker IP: 192.168.129.228

1. Reconnaissance

I started with an Nmap scan to identify the exposed services and their versions.

nmap -sC -sV 10.128.184.149

Open ports
Port	Service	Version
21/tcp	FTP	vsftpd 3.0.5
22/tcp	SSH	OpenSSH 8.2p1
80/tcp	HTTP	Apache 2.4.41

One particularly interesting finding was that anonymous FTP access was enabled.

2. FTP Enumeration

I connected to the FTP service using the anonymous account:

ftp 10.128.184.149


Anonymous authentication was successful.

The FTP server contained a file named note.txt, which I downloaded and inspected:

Anurodh told me that there is some filtering on strings being put in the command -- Apaar


This provided an important clue about command filtering on the web application.

3. Web Enumeration

I then investigated the HTTP service running on port 80.

The web application exposed an endpoint that accepted user input and allowed commands to be executed.

Further testing revealed that the application was vulnerable to command injection.

The application filtered certain strings, but the filtering could be bypassed by using an absolute path.

For example, instead of relying on a command being resolved through the $PATH, I could explicitly reference the binary:

/bin/bash


This allowed me to bypass the filtering mechanism and execute a reverse shell payload.

4. Obtaining a Reverse Shell

I configured a listener on my attacking machine:

nc -lvnp <PORT>


I then supplied a Bash reverse-shell payload through the vulnerable parameter.

After triggering the request, I received a connection back to my machine.

At this point, I had obtained an initial shell on the target.

5. Privilege Escalation to Apaar

After obtaining the initial shell, I started enumerating the system and looking for ways to escalate privileges.

One interesting file was:

.helpfine.sh


I discovered that this script could be executed through sudo as the user apaar.

I verified the available sudo privileges with:

sudo -l


The script could then be executed in the context of apaar.

By exploiting this, I obtained a shell with the privileges of the apaar user.

User Flag
e8vpd3****************************

6. Investigating the Apaar Account

Once I had access as apaar, I inspected the user's home directory and hidden files.

The .ssh directory was present:

ls -la /home/apaar/


This indicated that SSH credentials could potentially be useful for further access.

I continued enumerating the web application's files.

7. Discovering Credentials in the Web Application

While examining the web application's source files, I found an interesting PHP file:

/var/www/files/index.php


The source code contained credentials stored in clear text.

This was an important finding because credentials exposed in application source code can often be reused elsewhere.

I also discovered an image directory containing files that appeared to be relevant to the challenge.

8. Steganography

One of the images contained hidden data.

After investigating the image, I identified an embedded ZIP archive.

The archive was password protected, so I extracted its hash and used John the Ripper to recover the password:

john hash.txt --wordlist=<wordlist>


The recovered password allowed me to extract the archive.

Inside the archive was a PHP file containing an encoded credential.

The credential was encoded using Base64:

IWQwbnRLbjB3bVlwQHNzdzByZA==


I decoded it with:

echo 'IWQwbnRLbjB3bVlwQHNzdzByZA==' | base64 -d


This revealed a password associated with the anurodh account.

9. SSH Access as Anurodh

I tested the recovered credentials against the SSH service:

ssh anurodh@10.128.184.149


The credentials were valid, giving me an interactive SSH session as anurodh.

I could now perform further local enumeration from a more privileged account.

10. Docker Privilege Escalation

During the enumeration of the system, I discovered that the current user had access to Docker.

Docker access can be highly privileged because containers can potentially be started with access to the host filesystem.

I used the corresponding Docker privilege-escalation technique to mount the host filesystem inside a container:

docker run -v /:/mnt --rm -it alpine chroot /mnt /bin/sh


The host filesystem was mounted under /mnt, and chroot allowed me to switch the root directory to the mounted host filesystem.

This resulted in a root shell on the target system.

I verified my privileges with:

whoami


Output:

root

11. Root Flag

The final flag was located after obtaining root access.

w18gfp****************************

12. Attack Path Summary

The complete attack chain was:

Anonymous FTP
      ↓
Information disclosure
      ↓
Web enumeration
      ↓
Command injection
      ↓
Reverse shell
      ↓
Sudo misconfiguration
      ↓
User: apaar
      ↓
Web application credential discovery
      ↓
Steganography
      ↓
ZIP password cracking
      ↓
Credential recovery
      ↓
SSH access as anurodh
      ↓
Docker privilege escalation
      ↓
Root

13. Key Takeaways

This room demonstrated several important penetration-testing concepts:

Service enumeration with Nmap

Anonymous FTP enumeration

Command injection

Reverse shells

Sudo privilege escalation

Local file and source-code enumeration

Steganography

Password cracking with John the Ripper

Base64 decoding

SSH credential reuse

Docker-based privilege escalation

The main lesson from this machine is the importance of chaining multiple small weaknesses together. None of the individual findings necessarily provided complete control, but combining them allowed the attack to progress from anonymous FTP access to a root shell.

14. Mitigation

From a defensive perspective, the following measures could reduce the attack surface:

Disable anonymous FTP access unless it is strictly required.

Properly validate and sanitize user-controlled input.

Avoid command execution from web applications whenever possible.

Implement strict sudo permissions and avoid allowing users to execute arbitrary scripts with elevated privileges.

Never store credentials in source code.

Avoid embedding sensitive information in publicly accessible files.

Protect archives and sensitive data with strong passwords.

Restrict Docker access to trusted administrators.

Apply the principle of least privilege to system and service accounts.
