# SECURITY ASSESSMENT REPORT - Noob

## 1. Reconnaissance

### 1.1. Port Scanning

Đầu tiên, sử dụng `nmap` để quét các cổng đang mở trên IP mục tiêu:

![img](img/nmap.png?raw=true)

Kết quả cho thấy hệ thống có 3 cổng đang mở. Trong đó, cổng `21/tcp` chạy dịch vụ FTP là đáng chú ý vì quá trình quét phát hiện hai file `cred.txt` và `welcome`.

Tiến hành truy cập FTP bằng tài khoản mặc định `anonymous`:

```bash
ftp <TARGET>
```

Sau khi đăng nhập thành công, sử dụng lệnh `get` để tải hai file về máy:

```text
get cred.txt
get welcome
```

![img](img/ftp.png?raw=true)

---

## 2. Information Disclosure thông qua FTP

Sau khi tải các file về, tiến hành kiểm tra nội dung của `welcome`.

![img](img/welcome.png?raw=true)

File `welcome` không chứa thông tin hữu ích. Tuy nhiên, file `cred.txt` chứa một chuỗi được mã hóa bằng Base64.

![img](img/decodeb64.png?raw=true)

Sau khi giải mã chuỗi Base64, kết quả có dạng username và password. Đây có thể là thông tin đăng nhập được sử dụng cho một dịch vụ khác trên máy chủ.

Tiếp theo, tiến hành kiểm tra web server trên cổng 80 để xác định xem có trang đăng nhập hay không.

---

## 3. Web Application Enumeration

Truy cập vào cổng 80:

![img](img/port80.png?raw=true)

Trang web hiển thị một giao diện đăng nhập. Sử dụng thông tin đăng nhập thu được từ file `cred.txt` để thử authentication.

![img](img/login_success.png?raw=true)

Đăng nhập thành công.

Trong ứng dụng, mục **About Us** là thành phần đáng chú ý. Khi truy cập vào mục này, server cung cấp một file RAR để tải xuống.

![img](img/download.png?raw=true)

Sau khi tải và giải nén file RAR, thu được ba file. Tiến hành sử dụng lệnh `file` để xác định loại file và kiểm tra nội dung file `sudo`.

![img](img/info.png?raw=true)

Kết quả cho thấy hai file là hình ảnh JPEG. File có tên `sudo` có thể là một gợi ý liên quan đến quá trình khai thác tiếp theo.

---

## 4. Steganography Analysis

Tiến hành mở hai file hình ảnh để kiểm tra:

![img](img/display.png?raw=true)

Hai hình ảnh có nội dung hiển thị giống nhau. Tuy nhiên, khi kiểm tra kích thước file, nhận thấy file JPEG có dung lượng lớn hơn đáng kể so với file BMP.

![img](img/cmp.png?raw=true)

Sự khác biệt về kích thước có thể cho thấy một trong các file chứa dữ liệu được ẩn bên trong.

Do đó, sử dụng `steghide` để kiểm tra dữ liệu được nhúng trong file JPEG:

```bash
steghide extract -sf <file.jpg>
```

Thử trích xuất dữ liệu mà không cung cấp passphrase và thu được file `hint.py`.

![img](img/hint.png?raw=true)

File `hint.py` cung cấp thêm một gợi ý liên quan đến phương pháp mã hóa/biến đổi chuỗi.

---

## 5. Phân tích Hint và khai thác Steghide

Quay lại kiểm tra file `sudo`, nhận thấy nội dung của file có thể là một gợi ý liên quan đến **tên file** và passphrase.

Dựa trên gợi ý từ `hint.py`, tiến hành kiểm tra dữ liệu ẩn trong file BMP bằng `steghide` với passphrase là:

```text
sudo
```

Quá trình trích xuất thành công:

![img](img/filebmp.png?raw=true)

Kết quả thu được một file `user.txt`.

Nội dung của file này có dạng một chuỗi đã được biến đổi. Dựa trên gợi ý trước đó, có thể nhận định chuỗi được mã hóa bằng **ROT13**.

Tiến hành giải mã bằng CyberChef:

![img](img/rot13.png?raw=true)

Sau khi giải mã ROT13, thu được một username và password có thể sử dụng để truy cập SSH.

---

# 6. Initial Access – SSH

Sử dụng thông tin đăng nhập vừa tìm được để SSH vào máy chủ:

```bash
ssh <username>@<TARGET>
```

![img](img/ssh.png?raw=true)

Đăng nhập SSH thành công với tài khoản `wtf`.

---

# 7. User Flag

Sau khi đăng nhập với user `wtf`, tiến hành kiểm tra các thư mục của người dùng.

Trong thư mục `Downloads` phát hiện file `flag-1`. Nội dung file đang được mã hóa dưới dạng Base64.

![img](img/flag1.png?raw=true)

Tiến hành decode Base64 để thu được **User Flag**.

---

# 8. Privilege Escalation

Sau khi thu được user flag, tiến hành kiểm tra các quyền sudo của user `wtf`:

```bash
sudo -l
```

Kết quả cho thấy user `wtf` có quyền thực thi một command với quyền `root` mà không yêu cầu password.

![img](img/root1.png?raw=true)

Đây là một hướng trực tiếp để thực hiện privilege escalation.

Ngoài ra, trong quá trình enumeration còn phát hiện một hướng khai thác khác thông qua file backup.

---

# 9. Privilege Escalation – Method 1

Kết quả `sudo -l` cho thấy user `wtf` có thể sử dụng quyền sudo để thực thi command với quyền `root`.

Do đó có thể tận dụng quyền sudo này để chuyển trực tiếp lên quyền root mà không cần biết mật khẩu root.

Đây là một lỗi cấu hình quyền sudo, do tài khoản người dùng thông thường được cấp quyền thực thi một chương trình hoặc command với đặc quyền root.

---

# 10. Privilege Escalation – Method 2

Một hướng khai thác khác được phát hiện trong quá trình tìm kiếm file trên hệ thống.

Trong thư mục `Documents` tồn tại file `backup.sh`.

![img](img/backup.png?raw=true)

Tiến hành kiểm tra nội dung file và phát hiện thông tin username cùng password của tài khoản `n00b`.

Sử dụng thông tin này để chuyển sang user `n00b`:

```bash
su n00b
```

![img](img/n00b.png?raw=true)

Sau khi chuyển sang user `n00b`, tiếp tục kiểm tra quyền sudo:

```bash
sudo -l
```

Kết quả cho thấy user `n00b` có thể thực thi `/bin/nano` với quyền `root` mà không cần password.

---

# 11. Exploiting Sudo nano

Do `nano` có khả năng thực hiện các thao tác tương tác với hệ thống, việc cho phép thực thi `nano` bằng `sudo` với quyền root có thể bị lợi dụng để thực thi command với đặc quyền cao hơn.

Tra cứu kỹ thuật khai thác trên GTFOBins:

![img](img/GTFO.png?raw=true)

Sau đó thực hiện kỹ thuật khai thác với `sudo`:

```bash
sudo nano
```

Khai thác thành công và chuyển sang shell với quyền `root`.

![img](img/root.png?raw=true)

---

# 12. Root Flag

Sau khi có quyền root, tiến hành truy cập thư mục `/root` và đọc file `root.txt`:

```bash
cd /root
cat root.txt
```

![img](img/root_flag.png?raw=true)

Đã hoàn thành quá trình khai thác và thu được root flag.

---

# 13. Attack Chain

Toàn bộ quá trình khai thác có thể được tóm tắt như sau:

```text
Nmap
  ↓
FTP Anonymous Login
  ↓
Download cred.txt
  ↓
Base64 Decode
  ↓
Web Login
  ↓
Download RAR
  ↓
Steganography Analysis
  ↓
Extract hint.py
  ↓
Steghide + passphrase "sudo"
  ↓
Extract user.txt
  ↓
ROT13 Decode
  ↓
SSH Login as wtf
  ↓
User Flag
  ↓
Privilege Enumeration
  ↓
sudo / backup.sh
  ↓
n00b
  ↓
sudo nano
  ↓
Root
  ↓
Root Flag
```