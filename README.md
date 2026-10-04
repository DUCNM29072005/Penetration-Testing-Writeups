# Pentest Writeups

Tổng hợp các **bài thực hành Penetration Testing, Cybersecurity Lab và CTF Writeup** được thực hiện trên các nền tảng:

* **Hack The Box (HTB)**
* **TryHackMe (THM)**
* **VulnHub**

Repository này được sử dụng để lưu lại quá trình học tập, phương pháp kiểm thử, kỹ thuật khai thác lỗ hổng và các công cụ được sử dụng trong quá trình thực hành.

> **Lưu ý:** Các bài thực hành trong repository được thực hiện trên những môi trường lab được phép sử dụng và nhằm mục đích học tập, nghiên cứu an toàn thông tin.

---

## 📂 Cấu trúc Repository

```text
pentest-writeups/
│
├── HackTheBox/
│   ├── Easy/
│   ├── Medium/
│   └── Hard/
│
├── TryHackMe/
│   ├── Easy/
│   ├── Medium/
│   └── Hard/
│
├── VulnHub/
│   
│
└── README.md
```

---

## 🎯 Mục tiêu

Repository tập trung vào việc thực hành quy trình **Penetration Testing**, từ giai đoạn thu thập thông tin cho đến khai thác và leo thang đặc quyền.

Quy trình cơ bản:

```text
Reconnaissance
      │
      ▼
Enumeration
      │
      ▼
Vulnerability Identification
      │
      ▼
Exploitation
      │
      ▼
Initial Access
      │
      ▼
Privilege Escalation
      │
      ▼
Post-Exploitation
      │
      ▼
Flag / Proof
      │
      ▼
Lessons Learned
```

---

## 🔎 Nội dung thực hành

### 1. Reconnaissance & Enumeration

* Port Scanning
* Service Enumeration
* Banner Grabbing
* DNS Enumeration
* Subdomain Enumeration
* Directory Enumeration
* Virtual Host Enumeration
* Technology Detection
* Network Service Enumeration

### 2. Web Application Security

* OWASP Top 10
* SQL Injection
* Cross-Site Scripting (XSS)
* IDOR / Broken Access Control
* Server-Side Request Forgery (SSRF)
* File Upload
* Path Traversal
* Local File Inclusion (LFI)
* Command Injection
* Authentication Vulnerabilities
* Security Misconfiguration

### 3. Network Security

* FTP
* SSH
* SMB
* HTTP / HTTPS
* DNS
* SMTP
* Network Service Enumeration
* Packet Analysis

### 4. Privilege Escalation

**Linux**

* SUID / SGID
* Sudo Misconfiguration
* Cron Jobs
* Writable Files
* PATH Hijacking
* Linux Capabilities
* Kernel Exploitation
* Credential Discovery

**Windows**

* Windows Service Misconfiguration
* Weak Permissions
* Scheduled Tasks
* PowerShell
* Token Privileges
* Credential Discovery

### 5. Password & Credential Attacks

* Brute Force
* Password Spraying
* Credential Reuse
* Hash Cracking
* Dictionary Attack
* John the Ripper
* Hashcat
* Hydra

### 6. Exploitation

* Manual Exploitation
* Metasploit Framework
* Reverse Shell / Bind Shell
* Exploit Development cơ bản
* Payload Analysis

---

## 🛠️ Công cụ

| Nhóm | Công cụ |
|---|---|
| Reconnaissance | Nmap, Gobuster, FFUF, Nikto, DNSRecon |
| Web Security | Burp Suite, SQLMap, FFUF, Gobuster |
| Exploitation | Metasploit, Netcat, Socat |
| Password Cracking | Hashcat, John the Ripper, Hydra |
| Analysis & RE | Wireshark, Ghidra, IDA, x64dbg |

---

## 📝 Cấu trúc một Writeup

```text
Machine Information
        │
        ▼
Reconnaissance
        │
        ▼
Enumeration
        │
        ▼
Vulnerability Identification
        │
        ▼
Exploitation
        │
        ▼
Initial Foothold
        │
        ▼
Privilege Escalation
        │
        ▼
Flags / Proof
        │
        ▼
Lessons Learned
```

Một writeup thường bao gồm: thông tin mục tiêu → reconnaissance → port scanning → service enumeration → web enumeration → phân tích lỗ hổng → khai thác → initial access → privilege escalation → flag/proof → kinh nghiệm rút ra.

---

## 📁 Ví dụ cấu trúc một Machine

```text
Machine-Name/
│
├── README.md
│
└── images/
    ├── nmap.png
    ├── enumeration.png
    ├── exploit.png
    └── privilege-escalation.png
```

Trong `README.md` của từng machine có thể sử dụng cấu trúc:

````markdown
# Machine Name

## Thông tin

- Platform: Hack The Box
- Difficulty: Easy
- OS: Linux

## 1. Reconnaissance

Tiến hành quét các port đang mở bằng Nmap.

## 2. Enumeration

Phân tích các service được phát hiện và xác định attack surface.

## 3. Vulnerability Identification

Xác định các lỗ hổng có khả năng khai thác.

## 4. Exploitation

Tiến hành khai thác lỗ hổng để đạt được initial access.

## 5. Privilege Escalation

Thực hiện enumeration trên hệ thống để tìm kiếm vector leo thang đặc quyền.

## 6. Flag

```text
flag{...}
```
````

---

## 📚 Kiến thức đã thực hành

Penetration Testing · Web Application Security · Network Security · Linux Security · Windows Security · Privilege Escalation · Vulnerability Assessment · Exploitation · Reconnaissance · Enumeration · CTF Methodology

---

## 🚧 Tiến độ

| Nền tảng | Trạng thái |
|---|---|
| Hack The Box | 🚧 Đang thực hiện |
| TryHackMe | 🚧 Đang thực hiện |
| VulnHub | 🚧 Đang thực hiện |

---

## ⚠️ Disclaimer

Tất cả các kỹ thuật và nội dung trong repository được thực hiện trên các **môi trường lab được phép**, các hệ thống cố tình có lỗ hổng hoặc các nền tảng dành cho việc học tập như Hack The Box, TryHackMe và VulnHub.

Repository này được xây dựng nhằm mục đích **học tập, thực hành và nghiên cứu an toàn thông tin**. Không sử dụng các kỹ thuật được trình bày trong repository để tấn công hoặc kiểm thử các hệ thống khi chưa có sự cho phép.

---

## 👤 Tác giả

**Nguyễn Minh Đức**

Quan tâm đến: Penetration Testing · Web Security · Reverse Engineering · Malware Analysis · Binary Exploitation · Cybersecurity Research