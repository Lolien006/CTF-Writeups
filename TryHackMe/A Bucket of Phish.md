# 🎣 TryHackMe — A Bucket of Phish

> **Room:** [A Bucket of Phish](https://tryhackme.com/room/abucketofphish)  
> **Difficulty:** Easy  
> **Tags:** `S3` `AWS` `Misconfiguration` `OSINT` `Phishing`
>
> A phishing operation got sloppy and left its loot in a **publicly accessible AWS S3 bucket**. This walkthrough shows how to discover the misconfiguration and dump the captured credentials using nothing but the AWS CLI.

---

## 📖 Table of Contents

1. [Scenario](#-scenario)
2. [Reconnaissance](#-reconnaissance)
3. [Enumerating the Bucket](#-enumerating-the-bucket)
4. [Exfiltrating the Data](#-exfiltrating-the-data)
5. [Flag](#-flag)
6. [Key Takeaways](#-key-takeaways)

---

## 🎯 Scenario

While investigating a phishing campaign, we come across the attacker's fake login page hosted on AWS S3:

```
http://darkinjector-phish.s3-website-us-west-2.amazonaws.com
```

S3 static websites are served **anonymously** — and if the bucket policy is misconfigured, the *contents* of the bucket can be just as public as the website itself. Let's see if the attacker made that classic mistake.

---

## 🔍 Reconnaissance

The URL already gives us everything we need to identify the bucket:


| Field           | Value                           |
| --------------- | ------------------------------- |
| **Bucket name** | `darkinjector-phish`            |
| **Region**      | `us-west-2`                     |
| **Access**      | Static website hosting (public) |


Unlike a normal `https://s3.amazonaws.com/bucket` URL, a `-s3-website-` URL tells us the bucket has **static website hosting enabled**. Website-enabled buckets don't respond to the usual S3 API calls the same way — but the underlying objects may still be listable and readable if ACLs/policies allow it.

> 💡 **Tip:** For CTF-style rooms, the `--no-sign-request` flag is your best friend. It tells the AWS CLI to make **anonymous, unauthenticated** requests — exactly like any random internet user (or victim) would.

---

## 📂 Enumerating the Bucket

First, let's check whether we can list the bucket's contents without any credentials:

```bash
aws s3 ls s3://darkinjector-phish --no-sign-request --region us-west-2
```

**Breakdown of the flags:**


| Flag                 | Purpose                                               |
| -------------------- | ----------------------------------------------------- |
| `s3 ls s3://...`     | List objects inside the bucket                        |
| `--no-sign-request`  | Make the request **anonymously** (no AWS credentials) |
| `--region us-west-2` | Target the correct region                             |


**Output:**

```
2024-01-15 10:32:01      13842 captured-logins-093582390
2024-01-15 09:47:22       2345 index.html
2024-01-15 09:47:22       1120 login.css
...
```

The listing works — the bucket is **world-readable**. And one file immediately catches the eye: `captured-logins-093582390`. That's not part of any website — that's the phishing kit's **harvested credentials**.

---

## 📥 Exfiltrating the Data

Let's download the stolen logins to our local machine:

```bash
aws s3 cp s3://darkinjector-phish/captured-logins-093582390 ./victims.txt --no-sign-request --region us-west-2
```

**Breakdown of the flags:**


| Flag                           | Purpose                                |
| ------------------------------ | -------------------------------------- |
| `s3 cp s3://... ./victims.txt` | Copy the remote object to a local file |
| `--no-sign-request`            | Again — fully anonymous access         |
| `--region us-west-2`           | Region must match the bucket's home    |


Then inspect the loot:

```bash
cat victims.txt
```

The file contains credentials submitted by real victims of the phishing page — usernames, passwords, and timestamps.

---

## 🏁 Flag

The flag is sitting in the harvested logins:

```text
THM{*************}
```
