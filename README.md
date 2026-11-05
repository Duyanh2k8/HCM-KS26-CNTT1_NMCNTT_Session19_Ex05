# Bài 5 : Xây dựng cơ chế xác thực an toàn cho Crypto Wallet
## *Thiết kế kiến trúc*
### 1. Thiết kế kiến trúc bảo mật MFA + TOTP
Hệ thống sử dụng 3 lớp bảo vệ phối hợp với nhau:
- **Lớp 1 – Password:** Sử dụng mật khẩu mạnh, được băm bằng Argon2id để bảo vệ thông tin đăng nhập.
- **Lớp 2 – TOTP (MFA):** Yêu cầu mã OTP gồm 6 chữ số, thay đổi sau mỗi 30 giây.
- **Lớp 3 – Device & Session Security:** Kiểm tra thiết bị đăng nhập, vị trí bất thường và phiên đăng nhập để phát hiện truy cập đáng ngờ.
```text
[User]
   |
   v
[Layer 1: Password / Passphrase]
   |  (1) Nhập mật khẩu
   v
[Auth Server]
   |  (2) Kiểm tra password + bảo mật brute-force
   v
[Layer 2: TOTP]
   |  (3) Yêu cầu mã 6 số từ Authenticator app
   |     Secret key đã được lưu trong app và server khi đăng ký MFA
   v
[TOTP Validator]
   |  (4) So khớp với thời gian hiện tại (30s window)
   v
[Layer 3: Device / Biometric / Secure Enclave]
   |  (5) Xác nhận trên thiết bị tin cậy
   |     hoặc xác thực bằng WebAuthn / Secure Enclave / hardware key
   v
[Wallet Backend]
   |  (6) Cho phép truy cập / mở ví
   v
[Crypto Wallet]
```
#### Giải thích cách 3 lớp phối hợp
**Lớp 1: Mật khẩu mạnh**
- Yêu cầu mật khẩu dài, phức tạp và không sử dụng lại ở tài khoản khác.
- Máy chủ không lưu mật khẩu dạng văn bản mà sử dụng thuật toán băm Argon2id.
- Nếu hacker lấy được mật khẩu, họ vẫn chưa thể đăng nhập.

**Lớp 2: Xác thực TOTP**
- Người dùng phải nhập mã OTP gồm 6 chữ số từ ứng dụng Google Authenticator hoặc Authy.
- Mã OTP thay đổi sau mỗi 30 giây.
- Máy chủ kiểm tra mã OTP dựa trên khóa bí mật đã đăng ký và thời gian hiện tại.
- Hacker có mật khẩu nhưng không có khóa bí mật TOTP thì khó có thể vượt qua lớp xác thực này.

**Lớp 3: Kiểm tra thiết bị và phiên đăng nhập**
- Kiểm tra thiết bị mới, địa chỉ IP và dấu hiệu đăng nhập bất thường.
- Nếu phát hiện rủi ro, hệ thống yêu cầu xác minh bổ sung hoặc tạm thời từ chối truy cập.
- Tự động hết hạn phiên đăng nhập khi không sử dụng trong thời gian quy định.
## *Xử lý bẫy (Edge Case)*
### 2. Xử lý tình huống Edge Case: Người dùng đang ở trên máy bay
**Vấn đề:** Người dùng không có Internet hoặc sóng điện thoại nhưng vẫn cần mã OTP để đăng nhập và thực hiện giao dịch.

**Giải pháp:** Sử dụng TOTP Offline.

TOTP không cần Internet hoặc sóng điện thoại để tạo mã OTP vì ứng dụng Authenticator sử dụng hai thành phần:
- Khóa bí mật (Secret Key) được thiết lập khi đăng ký MFA.
- Thời gian hiện tại trên thiết bị.

Ứng dụng kết hợp hai thành phần này bằng thuật toán HMAC để tạo mã OTP 6 chữ số, thường thay đổi sau mỗi 30 giây.

**Các bước thực hiện:**
1. Trước chuyến bay, người dùng đã cài đặt Google Authenticator hoặc ứng dụng TOTP tương thích và đăng ký MFA.
2. Khi lên máy bay, bật chế độ máy bay.
3. Mở ứng dụng Authenticator, ứng dụng vẫn tự tạo mã OTP dù không có Internet.
4. Nhập mã OTP vào ứng dụng Crypto Wallet.
5. Máy chủ xác thực kiểm tra mã OTP khi thiết bị có thể kết nối lại với hệ thống.
