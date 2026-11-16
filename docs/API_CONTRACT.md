# Hợp đồng REST API của QuizPulse

Tài liệu mô tả giao tiếp HTTP giữa frontend và backend theo mã nguồn tham chiếu QuizPulse.

## Lấy phiên hiện tại

`GET /api/auth/session`

Guest, User và Admin đều gọi được. Request không có body. Trình duyệt gửi cookie hiện có cùng request. Server có thể cấp cookie `QP_GUEST` khi cần nhận diện khách.

**HTTP 200 cho Guest**

```json
{
  "authenticated": false,
  "role": "GUEST",
  "csrfToken": "<token>"
}
```

**HTTP 200 cho tài khoản đã đăng nhập**

```json
{
  "authenticated": true,
  "username": "quizplayer",
  "role": "USER",
  "csrfToken": "<token>"
}
```

Với tài khoản quản trị, `role` là `ADMIN`. Phản hồi Guest không có trường `username`.

Frontend gửi giá trị `csrfToken` trong header `X-CSRF-TOKEN` cho các yêu cầu `POST`, `PUT` và `DELETE`. Sau khi đăng nhập hoặc đăng ký thành công, frontend gọi lại endpoint này để lấy token cho phiên HTTP mới.
