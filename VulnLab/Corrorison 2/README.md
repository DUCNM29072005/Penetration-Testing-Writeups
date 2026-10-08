# SECURITY ASSESSMENT REPORT - Corrorison 2

## 1. Reconnaissance

### 1.1. Port Scanning

Sử dụng `nmap` để quét các cổng đang mở trên IP mục tiêu:

![img](img/nmap.png?raw=true)

Kết quả cho thấy hệ thống đang mở 3 cổng:

* `22/tcp` – SSH
* `80/tcp` – HTTP
* `8080/tcp` – HTTP / Apache Tomcat

Tiến hành truy cập vào hai cổng web `80` và `8080` để kiểm tra các dịch vụ đang chạy.

![img](img/port80.png?raw=true)

---

## 2. Web Enumeration

Tiến hành sử dụng Gobuster để tìm kiếm các directory và file ẩn trên cổng 80. Tuy nhiên, không phát hiện được endpoint đáng chú ý.

Do đó, chuyển sang kiểm tra dịch vụ trên cổng 8080.

![img](img/port8080.png?raw=true)

Cổng 8080 đang chạy **Apache Tomcat**. Trang Tomcat cung cấp khu vực quản trị `Manager`, tuy nhiên yêu cầu username và password để đăng nhập.

Ban đầu, thử sử dụng các username/password mặc định của Tomcat nhưng không thành công.

![img](img/hydra.png?raw=true)

Do không thể đăng nhập bằng credential mặc định, tiếp tục thực hiện directory enumeration trên web server.

---

## 3. Phát hiện Backup File

Tiến hành sử dụng Gobuster để tìm kiếm các file và directory trên cổng 8080:

![img](img/gobusterbackup.png?raw=true)

Kết quả phát hiện file:

```text
backup.zip
```

Truy cập vào file này khiến server tự động tải file backup về máy.

Khi thử giải nén, file yêu cầu password:

![img](img/unzip.png?raw=true)

Do đó, sử dụng `fcrackzip` kết hợp với wordlist `rockyou.txt` để tìm password của file ZIP:

```bash
fcrackzip -u -D -p /usr/share/wordlists/rockyou.txt backup.zip
```

![img](img/fcrackzip.png?raw=true)

Quá trình brute-force thành công và thu được password của file ZIP.

Tiến hành giải nén:

![img](img/unzip_success.png?raw=true)

Sau khi giải nén, phát hiện các file cấu hình chứa thông tin đăng nhập.

---

## 4. Compromise Tomcat Manager

Kiểm tra file XML thu được từ backup:

![img](img/tomcatxml.png?raw=true)

Trong file tồn tại username và password dùng để xác thực với Tomcat Manager.

Sử dụng credential này để đăng nhập vào Tomcat Manager:

![img](img/login_success.png?raw=true)

Đăng nhập thành công và có quyền truy cập dashboard quản trị Tomcat.

---

# 5. Remote Code Execution thông qua WAR Deployment

Tomcat Manager cung cấp chức năng:

```text
WAR file to deploy
```

Chức năng này cho phép administrator upload một file `.war` và triển khai nó trực tiếp thành một web application trên server.

Nếu attacker có quyền Tomcat Manager và có khả năng upload WAR tùy ý, chức năng này có thể bị lợi dụng để thực thi mã tùy ý trên server.

Do đã có quyền quản trị Tomcat, tiến hành tạo một WAR payload để thiết lập reverse shell.

Sử dụng `msfvenom` để tạo payload:

![img](img/msfvenom.png?raw=true)

Sau khi tạo payload, upload file WAR thông qua Tomcat Manager:

![img](img/upload.png?raw=true)

Upload thành công. Sau đó thiết lập listener trên máy attacker và truy cập vào web application vừa triển khai.

Kết quả nhận được reverse shell trên máy chủ mục tiêu:

![img](img/reverseshell.png?raw=true)

Để cải thiện môi trường shell, thực hiện nâng cấp shell thành TTY:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

---

# 6. User Flag

Sau khi có shell trên server, tiến hành kiểm tra các thư mục `/home` để xác định các tài khoản người dùng.

Trong hệ thống tồn tại hai user. Tiếp tục kiểm tra thư mục của user `randy` và phát hiện file `user.txt`.

![img](img/user_flag.png?raw=true)

Đọc file và thu được **User Flag**.

---

# 7. Privilege Escalation – Tomcat → jaye

Trong quá trình enumeration, phát hiện thêm file `note.txt`, có khả năng chứa thông tin liên quan đến privilege escalation.

Dựa trên thông tin thu được từ file backup trước đó, thử sử dụng credential để đăng nhập vào tài khoản `jaye` thông qua SSH.

![img](img/ssh_success.png?raw=true)

Đăng nhập SSH thành công với tài khoản `jaye`.

---

# 8. Privilege Escalation – jaye → randy

Sau khi đăng nhập với user `jaye`, phát hiện một thư mục `Files` chứa file `look`.

![img](img/look.png?raw=true)

File `look` có quyền thực thi với quyền `root`.

Tiến hành kiểm tra khả năng khai thác file này. Tra cứu trên GTFOBins cho thấy chương trình có thể được lợi dụng để đọc các file mà user hiện tại không có quyền đọc.

![img](img/GTFO.png?raw=true)

Sử dụng kỹ thuật này để đọc:

```text
/etc/shadow
```

Kết quả thu được hash mật khẩu của user `randy`.

![img](img/shadow1%20\(1\).png?raw=true)

![img](img/shadow1%20\(2\).png?raw=true)

Lưu hash và sử dụng John the Ripper cùng wordlist `rockyou.txt` để crack password:

```bash
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

![img](img/john.png?raw=true)

Sau khi crack thành công, sử dụng password vừa tìm được để đăng nhập vào tài khoản `randy`.

![img](img/login_success_ssh_randy.png?raw=true)

---

# 9. Privilege Escalation – randy → root

Sau khi đăng nhập với user `randy`, kiểm tra quyền sudo:

```bash
sudo -l
```

![img](img/sudo%20-l.png?raw=true)

Kết quả cho thấy user `randy` có thể thực thi file:

```text
randombase64.py
```

với quyền `root`.

Tiến hành kiểm tra file:

![img](img/base64.png?raw=true)

File sử dụng Python 3.8 và import thư viện `base64`.

Điểm đáng chú ý là Python sẽ tìm kiếm module `base64` trong các thư mục thư viện của Python. Trong quá trình kiểm tra, phát hiện file `base64.py` mà chương trình import có quyền ghi.

Điều này tạo ra khả năng **Python Library Hijacking**: thay thế nội dung của module `base64.py` bằng code do attacker kiểm soát.

---

# 10. Python Library Hijacking

Tiến hành chỉnh sửa file `base64.py` và thêm đoạn code thực thi shell:

```python
import os
os.system("/bin/bash")
```

![img](img/writebase64.png?raw=true)

Sau khi thay đổi thư viện, chạy `randombase64.py` bằng Python 3.8 với quyền sudo.

Do chương trình import module `base64`, phiên bản module đã bị thay đổi sẽ được thực thi với quyền `root`.

Kết quả là shell được cấp quyền root.

---

# 11. Root Flag

Sau khi có quyền root, kiểm tra quyền hiện tại:

```bash
whoami
```

Kết quả trả về:

```text
root
```

Tiến hành truy cập thư mục `/root` và đọc file `root.txt`:

![img](img/root_flag.png?raw=true)

Đã hoàn thành quá trình privilege escalation và thu được **Root Flag**.

---

# 12. Attack Chain

Toàn bộ quá trình khai thác có thể được mô tả như sau:

```text
Nmap
  ↓
Port 22 / 80 / 8080
  ↓
Apache Tomcat
  ↓
Directory Enumeration
  ↓
backup.zip
  ↓
Fcrackzip + Rockyou
  ↓
Extract Backup
  ↓
Tomcat Credentials
  ↓
Tomcat Manager
  ↓
WAR File Upload
  ↓
Remote Code Execution
  ↓
Reverse Shell
  ↓
User Flag
  ↓
Credential / Note Enumeration
  ↓
SSH as jaye
  ↓
SUID / Root File "look"
  ↓
Read /etc/shadow
  ↓
John the Ripper
  ↓
SSH as randy
  ↓
sudo randombase64.py
  ↓
Python Library Hijacking
  ↓
Root
  ↓
Root Flag
```
