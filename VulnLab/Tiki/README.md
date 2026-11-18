# SECURITY ASSESSMENT REPORT – Tiki Wiki CMS

# 1. Tổng quan

## 1.1. Thông tin mục tiêu

| Thông tin | Giá trị |
|---|---|
| Target | Máy chủ mục tiêu trong môi trường thực hành |
| Web Application | Tiki Wiki CMS Groupware |
| Assessment Type | Black-box Penetration Testing |
| Objective | Gain Initial Access and Escalate Privileges to Root |

## 1.2. Tóm tắt

Quá trình kiểm thử bắt đầu bằng việc sử dụng Nmap để thu thập thông tin về các dịch vụ đang hoạt động trên máy chủ mục tiêu. Kết quả quét cho thấy hệ thống đang mở bốn cổng dịch vụ, bao gồm SSH (22), HTTP (80), NetBIOS (139) và SMB (445).

Trong quá trình kiểm tra ứng dụng web, tôi sử dụng Gobuster để tìm kiếm các đường dẫn ẩn và phát hiện thư mục `/tiki/`, nơi triển khai nền tảng Tiki Wiki CMS Groupware. Tuy nhiên, ứng dụng yêu cầu xác thực tài khoản để truy cập các chức năng bên trong.

Tiếp theo, thông qua việc kiểm tra dịch vụ SMB, tôi phát hiện tệp `Mail.txt` chứa thông tin liên quan đến tài khoản Silky. Sử dụng thông tin thu thập được, tôi đăng nhập thành công vào ứng dụng Wiki và phát hiện thông tin về một lỗ hổng Authentication Bypass trong lịch sử chỉnh sửa trang.

Sau khi nghiên cứu và sử dụng mã khai thác PoC, tôi đã vượt qua cơ chế xác thực để truy cập vào tài khoản quản trị `admin` mà không cần mật khẩu. Từ quyền quản trị, tôi tiếp tục phát hiện thông tin đăng nhập SSH, truy cập vào hệ điều hành với tài khoản Silky và lợi dụng cấu hình sudo để leo thang đặc quyền lên root.

# 2. Reconnaissance

## 2.1. Port Scanning

Đầu tiên, tôi sử dụng công cụ Nmap để quét các cổng TCP đang mở trên địa chỉ IP mục tiêu. Mục đích của bước này là xác định những dịch vụ đang hoạt động, từ đó lựa chọn các bề mặt tấn công tiềm năng để tiến hành kiểm thử.

![img](img/nmap.png?raw=true)

Kết quả quét cho thấy hệ thống đang mở bốn cổng dịch vụ:

| Cổng | Dịch vụ | Mô tả |
|---|---|---|
| 22/tcp | SSH | Dịch vụ truy cập hệ thống từ xa |
| 80/tcp | HTTP | Dịch vụ Web Server |
| 139/tcp | NetBIOS-SSN | Dịch vụ hỗ trợ chia sẻ tài nguyên mạng |
| 445/tcp | SMB | Dịch vụ chia sẻ tệp và tài nguyên |

Trong số các dịch vụ được phát hiện, HTTP và SMB là hai mục tiêu đáng chú ý. Dịch vụ HTTP có thể chứa ứng dụng web tồn tại lỗ hổng bảo mật, trong khi SMB có khả năng làm lộ các tệp tin hoặc thông tin xác thực nếu được cấu hình không an toàn.

Vì vậy, tôi quyết định bắt đầu bằng việc truy cập dịch vụ HTTP đang hoạt động trên cổng 80.

## 2.2. HTTP Enumeration

Sau khi xác định cổng 80 đang mở, tôi sử dụng trình duyệt để truy cập vào địa chỉ IP của máy mục tiêu nhằm quan sát giao diện và tìm hiểu các chức năng được cung cấp.

![img](img/port80.png?raw=true)

Tiếp theo, tôi tiến hành liệt kê các thư mục và đường dẫn trên ứng dụng web để tìm kiếm những tài nguyên không được hiển thị trực tiếp trên giao diện.

# 3. Web Enumeration

## 3.1. Directory Enumeration

Tôi sử dụng công cụ Gobuster để thực hiện kỹ thuật Directory Enumeration, nhằm phát hiện các thư mục và tệp tin tồn tại trên Web Server.

Kết quả quét cho thấy hai đường dẫn đáng chú ý:

```text
/robots.txt
/tiki/
```

Đầu tiên, tôi truy cập vào tệp `robots.txt` và thu được nội dung:

```text
User-Agent: *
Disallow:
Disallow: /tiki/
```

Tệp `robots.txt` thường được sử dụng để hướng dẫn các công cụ tìm kiếm về những đường dẫn không nên thu thập dữ liệu. Tuy nhiên, nội dung của tệp cũng có thể vô tình tiết lộ các thư mục không được liên kết trực tiếp từ giao diện chính.

Trong trường hợp này, tệp `robots.txt` không chứa thông tin xác thực hoặc dữ liệu nhạy cảm, nhưng đã tiết lộ sự tồn tại của thư mục `/tiki/`.

## 3.2. Phát hiện Tiki Wiki CMS

Tiếp tục truy cập đường dẫn `/tiki/`, tôi phát hiện hệ thống đang triển khai nền tảng **Tiki Wiki CMS Groupware**, một hệ quản trị nội dung mã nguồn mở được sử dụng để xây dựng và quản lý các trang Wiki.

![img](img/tiki.png?raw=true)

Qua quá trình khảo sát, tôi nhận thấy ứng dụng cung cấp chức năng đăng nhập dành cho người dùng. Tuy nhiên, tại thời điểm này chưa có thông tin xác thực hợp lệ để truy cập vào các chức năng bên trong.

Do đó, tôi chuyển sang kiểm tra dịch vụ SMB đã được phát hiện trong quá trình quét Nmap, với mục tiêu tìm kiếm các tệp chia sẻ có thể chứa thông tin liên quan đến tài khoản người dùng.

# 4. SMB Enumeration

## 4.1. Kiểm tra tài nguyên chia sẻ SMB

Dịch vụ SMB hoạt động trên cổng 139 và 445, cho phép các thiết bị trong mạng chia sẻ tệp tin và tài nguyên. Nếu quyền truy cập không được cấu hình phù hợp, người dùng có thể đọc được những tệp chứa thông tin nội bộ.

Tôi tiến hành kết nối đến dịch vụ SMB trên máy mục tiêu và kiểm tra các tài nguyên được chia sẻ.

Trong quá trình này, tôi phát hiện một tệp có tên `Mail.txt`.

![img](img/getMail.png?raw=true)

Sau khi phát hiện tệp, tôi sử dụng lệnh `get` để tải tệp về máy kiểm thử và tiến hành đọc nội dung.

## 4.2. Phân tích Mail.txt

Sau khi tải xuống thành công, tôi sử dụng lệnh `cat` để kiểm tra nội dung tệp.

![img](img/catMail.png?raw=true)

Qua phân tích, tôi nhận thấy nội dung tệp có dạng một bức thư được gửi đến người dùng có tên **Silky**. Bức thư chứa thông tin có thể liên quan đến tài khoản đăng nhập trên hệ thống Wiki.

Từ dữ liệu thu thập được, tôi đặt giả thuyết rằng `Silky` là tên tài khoản hợp lệ trên Tiki Wiki CMS và mật khẩu được đề cập trong bức thư có thể được sử dụng để xác thực.

Tôi quay lại giao diện đăng nhập của ứng dụng, sử dụng tên tài khoản `Silky` kết hợp với mật khẩu thu thập được từ tệp Mail.txt.

Kết quả cho thấy quá trình xác thực thành công, cho phép tôi truy cập vào các chức năng dành cho người dùng đã đăng nhập.

Điều này cho thấy việc lưu trữ thông tin xác thực trong các tệp chia sẻ SMB có thể dẫn đến nguy cơ lộ lọt dữ liệu và tạo điều kiện cho kẻ tấn công mở rộng quyền truy cập vào hệ thống.

# 5. Vulnerability Discovery

## 5.1. Khảo sát các trang nội bộ

Sau khi đăng nhập thành công bằng tài khoản Silky, tôi tiếp tục khám phá các trang và chức năng có trong hệ thống Wiki.

Trong mục **List Pages**, tôi phát hiện một trang có tên **Silkys Homepage**.

![img](img/list%20page.png?raw=true)

Tôi tiến hành truy cập trang Silkys Homepage và kiểm tra lịch sử chỉnh sửa thông qua chức năng History.

Trong quá trình so sánh các phiên bản nội dung, tôi phát hiện thông tin đề cập đến mã lỗ hổng `CVE-2020-15906`.

![img](img/compare.png?raw=true)

Thông tin này là một dấu hiệu đáng chú ý, cho thấy ứng dụng có thể đang sử dụng phiên bản Tiki Wiki CMS tồn tại lỗ hổng bảo mật đã được công bố.

Từ đó, tôi tiếp tục nghiên cứu thông tin kỹ thuật và tìm kiếm các mã khai thác công khai liên quan đến lỗ hổng.

## 5.2. Nghiên cứu lỗ hổng Authentication Bypass

Tôi tiến hành tìm kiếm thông tin về CVE được phát hiện và tìm thấy một Proof of Concept (PoC) trên Exploit Database.

![img](img/exploit%20database.png?raw=true)

Theo nội dung nghiên cứu được sử dụng trong bài thực hành, lỗ hổng khai thác là **CVE-2020-15806**, thuộc nhóm Authentication Bypass trên nền tảng Tiki Wiki CMS Groupware.

Lỗ hổng được mô tả có điểm CVSS 3.1 là 9.8/10, thuộc mức độ nghiêm trọng Critical.

Nguyên nhân của lỗ hổng xuất phát từ lỗi xử lý logic trong tệp `tiki-login.php`, liên quan đến cơ chế bảo vệ tài khoản trước các cuộc tấn công dò mật khẩu.

Thông thường, khi một tài khoản nhận quá nhiều yêu cầu đăng nhập thất bại, hệ thống sẽ áp dụng cơ chế khóa tài khoản hoặc giới hạn số lần đăng nhập nhằm ngăn chặn brute-force.

Tuy nhiên, trong trường hợp bị ảnh hưởng bởi lỗ hổng, cơ chế xử lý có thể hoạt động không đúng. Sau khi tài khoản quản trị nhận đủ số lần đăng nhập thất bại, hệ thống có thể đặt lại mật khẩu của tài khoản thành giá trị rỗng thay vì khóa tài khoản.

Kẻ tấn công có thể lợi dụng hành vi này để vượt qua cơ chế xác thực bằng cách đăng nhập với tên tài khoản `admin` và mật khẩu rỗng.

Như vậy, thay vì ngăn chặn brute-force, cơ chế bảo vệ bị lỗi lại tạo điều kiện cho việc chiếm quyền truy cập tài khoản quản trị.

# 6. Exploitation – Authentication Bypass

## 6.1. Thực hiện khai thác bằng PoC

Sau khi nghiên cứu cơ chế hoạt động của lỗ hổng, tôi sử dụng mã PoC bằng Python để kiểm tra khả năng khai thác trên hệ thống mục tiêu.

Mã khai thác thực hiện các yêu cầu đăng nhập thất bại liên tiếp đối với tài khoản `admin`, nhằm kích hoạt lỗi xử lý trong cơ chế xác thực của Tiki Wiki CMS.

Theo mô tả của lỗ hổng, quá trình khai thác liên quan đến việc tạo ra khoảng 50 lần đăng nhập thất bại để khiến hệ thống chuyển sang trạng thái xử lý không an toàn.

![img](img/python.png?raw=true)

Sau khi thực thi mã khai thác, tôi quay lại giao diện đăng nhập để kiểm tra xem tài khoản quản trị có thể được truy cập mà không cần mật khẩu hay không.

## 6.2. Kiểm tra Authentication Bypass bằng Burp Suite

Để kiểm soát nội dung yêu cầu đăng nhập, tôi sử dụng Burp Suite nhằm chặn và chỉnh sửa HTTP request trước khi gửi đến máy chủ.

Tại đây, tôi giữ nguyên tên tài khoản `admin` và loại bỏ giá trị của trường mật khẩu, sau đó gửi yêu cầu đăng nhập đã chỉnh sửa.

![img](img/burpwithoutpass.png?raw=true)

Kết quả cho thấy máy chủ chấp nhận yêu cầu xác thực mặc dù không cung cấp mật khẩu hợp lệ.

Điều này chứng minh cơ chế xác thực của ứng dụng đã bị vượt qua và tài khoản quản trị có thể bị truy cập trái phép.

## 6.3. Compromise Admin Account

Sau khi gửi yêu cầu đăng nhập đã chỉnh sửa, tôi truy cập thành công vào ứng dụng Tiki Wiki CMS với quyền của tài khoản `admin`.

![img](img/login_to_admin.png?raw=true)

Kết quả khai thác cho thấy lỗ hổng Authentication Bypass có thể dẫn đến việc chiếm quyền quản trị ứng dụng.

Với quyền admin, người dùng có thể truy cập các chức năng quản lý và những nội dung không được phép xem bởi tài khoản thông thường.

Từ quyền truy cập này, tôi tiếp tục kiểm tra các trang nội bộ để tìm kiếm những thông tin nhạy cảm khác có thể hỗ trợ quá trình truy cập vào hệ điều hành.

# 7. Credential Discovery

## 7.1. Phát hiện thông tin xác thực SSH

Sau khi chiếm được quyền quản trị Tiki Wiki CMS, tôi tiếp tục khảo sát các trang và dữ liệu được lưu trữ trong ứng dụng.

Trong quá trình này, tôi phát hiện một trang có tên **Credential**, chứa thông tin được nghi ngờ là thông tin xác thực của một tài khoản trên hệ thống.

![img](img/credential.png?raw=true)

Dựa trên nội dung thu thập được, tôi đặt giả thuyết rằng thông tin này có thể được sử dụng để đăng nhập vào dịch vụ SSH đang hoạt động trên cổng 22.

Việc lưu trữ thông tin đăng nhập hệ điều hành trong ứng dụng Wiki là một vấn đề bảo mật đáng chú ý, bởi khi ứng dụng bị chiếm quyền, những thông tin này cũng có nguy cơ bị lộ.

## 7.2. SSH Access

Để kiểm tra giả thuyết, tôi sử dụng thông tin xác thực tìm được để kết nối đến dịch vụ SSH trên máy mục tiêu.

Kết quả cho thấy tôi đã đăng nhập thành công vào hệ thống với tài khoản `Silky`.

![img](img/ssh_success.png?raw=true)

Việc truy cập SSH thành công cho thấy tôi đã chuyển từ quyền truy cập ứng dụng web sang quyền truy cập trực tiếp vào hệ điều hành.

Tại thời điểm này, tôi có thể thực hiện các lệnh dưới quyền tài khoản Silky. Tuy nhiên, đây chưa phải là tài khoản có đặc quyền cao nhất, vì vậy tôi tiếp tục kiểm tra các quyền được cấp để tìm kiếm khả năng leo thang đặc quyền.

# 8. Privilege Escalation

## 8.1. Kiểm tra quyền sudo

Sau khi đăng nhập thành công qua SSH, tôi tiến hành kiểm tra quyền thực thi của tài khoản Silky trên hệ thống.

Lệnh được sử dụng:

```bash
sudo -l
```

Lệnh `sudo -l` cho phép liệt kê những chương trình hoặc lệnh mà người dùng hiện tại được phép thực thi thông qua sudo.

Kết quả kiểm tra cho thấy tài khoản Silky được cấp quyền sudo mà không yêu cầu nhập mật khẩu.

Đây là một cấu hình tiềm ẩn rủi ro bảo mật nghiêm trọng nếu quyền được cấp quá rộng, bởi người dùng có thể thực thi các thao tác với đặc quyền cao hơn mà không cần xác thực bổ sung.

## 8.2. Root Privilege Escalation

Dựa trên kết quả kiểm tra quyền sudo, tôi tiến hành sử dụng quyền được cấp để chuyển sang tài khoản có đặc quyền cao nhất trên hệ thống.

Quá trình leo thang đặc quyền thành công, cho phép tôi thực thi các thao tác với quyền `root`.

![img](img/root.png?raw=true)

Sau khi đạt được quyền root, tôi tiến hành kiểm tra kết quả và đọc dữ liệu mục tiêu theo yêu cầu của bài thực hành.

Như vậy, quá trình Privilege Escalation đã hoàn tất. Từ một tài khoản người dùng thông thường, tôi đã đạt được quyền quản trị cao nhất trên hệ điều hành.

# 9. Attack Chain

Toàn bộ quá trình khai thác được thực hiện theo chuỗi sau:

```text
Network Enumeration
        |
        v
Nmap Port Scanning
        |
        v
HTTP Enumeration
        |
        v
Gobuster Directory Enumeration
        |
        v
Discover /tiki/
        |
        v
Tiki Wiki CMS
        |
        v
SMB Enumeration
        |
        v
Discover Mail.txt
        |
        v
Recover Silky Credentials
        |
        v
Login to Tiki Wiki CMS
        |
        v
Discover CVE Information
        |
        v
Authentication Bypass
        |
        v
Compromise Admin Account
        |
        v
Credential Discovery
        |
        v
SSH Access as Silky
        |
        v
Sudo Enumeration
        |
        v
Privilege Escalation
        |
        v
ROOT
```