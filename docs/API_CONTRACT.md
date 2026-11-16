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

## Quy ước HTTP

- Frontend và API dùng cùng origin. Trình duyệt gửi cookie phiên `JSESSIONID` và cookie khách `QP_GUEST` khi có. `QP_GUEST` là cookie phiên `HttpOnly`, `SameSite=Lax`, có cờ `Secure` khi server nhận biết kết nối HTTPS.
- Request có body dùng JSON với `Content-Type: application/json`. Các thao tác thành công trong tài liệu trả `200` trừ trường hợp ghi rõ mã khác.
- Lỗi nghiệp vụ, lỗi đọc JSON và các lỗi do cấu hình Spring Security xử lý có hai trường chuỗi `code` và `message`. Ví dụ `{"code":"INVALID_JSON","message":"Dữ liệu JSON không hợp lệ."}`. Thiếu header bắt buộc hoặc sai kiểu tham số đường dẫn có thể do Spring MVC xử lý với định dạng lỗi mặc định.
- Nhánh bảo mật trả `401 LOGIN_REQUIRED` khi cần xác thực và `403 FORBIDDEN` khi thiếu quyền hoặc CSRF không hợp lệ. Một số endpoint public kiểm tra quyền ở controller hoặc service và trả `403 FORBIDDEN`.

## Đăng ký đăng nhập và đăng xuất

| Method và đường dẫn | Ai được gọi | Body JSON | Phản hồi thành công |
|---|---|---|---|
| `POST /api/auth/register` | Guest chưa ở trong phòng | `{"username":"quizplayer","password":"demoPass123"}` | `{"username":"quizplayer","role":"USER","authenticated":true}`. Tạo tài khoản và đăng nhập ngay. |
| `POST /api/auth/login` | Guest chưa ở trong phòng | Cùng cấu trúc với đăng ký | `username`, `role` (`USER` hoặc `ADMIN`), `authenticated: true`. |
| `POST /api/auth/logout` | User hoặc Admin đã rời phòng | Không có | `{"status":"ok"}`. Phiên HTTP bị hủy. Gọi lại `/api/auth/session` để lấy phiên và CSRF token cho lần thao tác tiếp theo. |

- Đăng ký chấp nhận tên từ 3 đến 30 ký tự ASCII thuộc `A–Z`, `a–z`, `0–9` hoặc `_`. Tên được chuyển thành chữ thường khi đăng ký và đăng nhập. Mật khẩu đăng ký dài từ 8 đến 72 ký tự. Dữ liệu sai trả `400 INVALID_USERNAME` hoặc `400 INVALID_PASSWORD`. Tên đã tồn tại trả `409 CONFLICT`.
- Đăng nhập thiếu trường trả `400 INVALID_INPUT`. Sai tên hoặc mật khẩu trả `401 INVALID_CREDENTIALS`.
- Đăng ký hoặc đăng nhập khi đã đăng nhập hay đang ở trong phòng với tư cách khách trả `409 CONFLICT`. Đăng xuất khi còn ở phòng trả `409 CONFLICT`. Gọi logout với tư cách Guest và CSRF hợp lệ trả `403 FORBIDDEN`.
- Đăng ký giới hạn 8 yêu cầu mỗi phút, đăng nhập 12 yêu cầu mỗi phút theo địa chỉ nguồn. Vượt giới hạn trả `429 RATE_LIMITED`.

## Danh sách bộ đề và đồng hồ server

| Method và đường dẫn | Ai được gọi | Phản hồi |
|---|---|---|
| `GET /api/quizzes` | Mọi người | Mảng `QuizSet` đã xuất bản, theo `id` giảm dần. |
| `GET /api/time` | Mọi người | `{"serverNow":1760000000000}`. Thời gian Unix tính bằng mili giây. |

`QuizSet` gồm `id` (số), `title`, `description`, `category` (chuỗi), `secondsPerQuestion` (số nguyên), `published` (boolean) và `questionCount` (số nguyên).

## Quản lý phòng

`{code}` là mã phòng 6 ký tự. REST bỏ khoảng trắng ở hai đầu và chuyển mã thành chữ hoa. Các route từ `state` đến `close` yêu cầu header `X-Player-Token` chứa token của người gọi và danh tính từ cookie phải khớp. Các lệnh `POST` cũng cần CSRF token.

| Method và đường dẫn | Ai được gọi | Body JSON | Phản hồi hoặc điều kiện |
|---|---|---|---|
| `GET /api/rooms/current` | Mọi người | Không có | `JoinReceipt` của phòng hiện tại, hoặc `204 No Content` nếu không có phòng. |
| `POST /api/rooms` | User hoặc Admin | `{"nickname":"Host","quizSetId":1}` | `JoinReceipt`. Bộ đề phải đã xuất bản và có câu hỏi. |
| `POST /api/rooms/{code}/join` | Mọi người | `{"nickname":"Player"}` | `JoinReceipt`. Thành viên đang hoạt động có cùng danh tính nhận lại thông tin cũ sau khi nickname đầu vào hợp lệ. Người mới chỉ vào được ở `LOBBY`, tối đa 12 người và nickname phải duy nhất theo cách so sánh không phân biệt hoa thường. |
| `GET /api/rooms/{code}/state` | Thành viên | Không có | Snapshot trạng thái chung kèm `self`. Xem [WebSocket Protocol](WEBSOCKET_PROTOCOL.md) để biết trường chung. |
| `POST /api/rooms/{code}/ready` | Thành viên | `{"ready":true}` | `{"status":"ok"}`. Chỉ ở `LOBBY`. Host luôn ở trạng thái sẵn sàng. |
| `POST /api/rooms/{code}/start` | Host | Không có | `{"status":"ok"}`. Chỉ ở `LOBBY`, cần ít nhất hai người và tất cả đã sẵn sàng. |
| `POST /api/rooms/{code}/kick` | Host | `{"playerId":"<id-người-chơi>"}` | `{"status":"ok"}`. Chỉ ở `LOBBY`, không loại Host. |
| `POST /api/rooms/{code}/leave` | Thành viên | Không có | `{"status":"ok"}`. Thu hồi quyền của người rời. Nếu người rời là Host thì đóng phòng. |
| `POST /api/rooms/{code}/close` | Host | Không có | `{"status":"ok"}`. Đóng phòng. |

`JoinReceipt` gồm các trường `roomCode`, `playerId`, `playerToken`, `nickname` kiểu chuỗi và `host` kiểu boolean. Trong snapshot REST, `self` gồm `playerId`, `nickname`, `host`, `answered` và có thêm `selectedOption` (số nguyên `0..3`) nếu đã có lượt nộp của người gọi trong vòng hiện tại. `answered` trong `self` phản ánh lượt nộp còn trong snapshot. `players[].answered` chỉ bật trong `QUESTION_OPEN`.

- Biệt danh sau khi bỏ khoảng trắng hai đầu dài từ 2 đến 20 ký tự, sai trả `400 INVALID_NICKNAME`. Thiếu dữ liệu tạo hoặc tham gia có thể trả `400 INVALID_ROOM` khi tới được kiểm tra trong controller. Các mã `400` khác gồm `QUIZ_UNAVAILABLE`, `QUIZ_EMPTY`, `INVALID_READY` và `INVALID_KICK` tùy thao tác.
- Guest tạo phòng, token sai hoặc không khớp danh tính, và người không phải Host gọi lệnh của Host đều trả `403 FORBIDDEN`. Phòng, bộ đề hoặc người bị kick không tồn tại trả `404 NOT_FOUND` tùy thao tác.
- Sai trạng thái phòng, đầy phòng, trùng biệt danh, chưa đủ người sẵn sàng hoặc đang tham gia phòng khác trả `409 CONFLICT` tùy thao tác. Tạo phòng giới hạn 12 yêu cầu mỗi phút, tham gia 30 yêu cầu mỗi phút theo nguồn. Vượt giới hạn trả `429 RATE_LIMITED`.

## Quản trị bộ đề và câu hỏi

Tất cả các route trong bảng cần vai trò Admin. Với route dưới `/quizzes/`, `{id}` là ID bộ đề. Với `/questions/`, `{id}` là ID câu hỏi.

| Method và đường dẫn | Body | Phản hồi |
|---|---|---|
| `GET /api/admin/quizzes` | Không có | Mảng `QuizSet`, gồm cả bộ chưa xuất bản, theo `id` giảm dần. |
| `POST /api/admin/quizzes` | `QuizSetInput` | `QuizSet` mới. |
| `PUT /api/admin/quizzes/{id}` | `QuizSetInput` | `QuizSet` sau cập nhật. |
| `DELETE /api/admin/quizzes/{id}` | Không có | `{"deleted":true}`. |
| `GET /api/admin/quizzes/{id}/questions` | Không có | Mảng `QuizQuestion`, theo `position` rồi `id` tăng dần. |
| `POST /api/admin/quizzes/{id}/questions` | `QuestionInput` | `QuizQuestion` mới. |
| `PUT /api/admin/questions/{id}` | `QuestionInput` | `QuizQuestion` sau cập nhật. |
| `DELETE /api/admin/questions/{id}` | Không có | `{"deleted":true}`. |

- `QuizSetInput` có `title` (1–140 ký tự), `category` (1–60), `description` (tối đa 600, mặc định chuỗi rỗng), `secondsPerQuestion` (số nguyên 5–90, mặc định 15) và `published` (boolean, mặc định `true`).
- `QuestionInput` có `prompt` (1–800 ký tự), `options` (mảng đúng 4 chuỗi, mỗi chuỗi 1–350 ký tự), `correctOption` (số nguyên `0..3`), `position` (số nguyên `1..10000`, mặc định 1), `difficulty` (`EASY`, `MEDIUM`, `HARD`, mặc định `MEDIUM`).
- `QuizQuestion` có các trường của `QuestionInput` sau chuẩn hóa, kèm `id` và `quizSetId` kiểu số.
- Dữ liệu không hợp lệ trả `400` với mã `INVALID_SET`, `INVALID_QUESTION` hoặc `INVALID_INPUT`. ID không tìm thấy trả `404 NOT_FOUND`.

## Lịch sử và thống kê

Các route này dùng `GET`, không có body. `/api/me/**` dành cho User và Admin, lọc theo tài khoản hiện tại. `/api/admin/**` yêu cầu Admin. ID ở route chi tiết là ID trận đấu đã lưu.

| Đường dẫn | Phản hồi |
|---|---|
| `/api/me/stats` | Object có `matches`, `totalScore`, `correctAnswers`, `answers`, `bestScore`. |
| `/api/me/stats/daily` | Mảng `{day, matches, score}` theo ngày tăng dần trong dữ liệu 14 ngày gần đây. |
| `/api/me/history` | Tối đa 30 trận gần nhất. Mỗi dòng có `id`, `roomCode`, `quizTitle`, `questionCount`, `finishedAt`, `nickname`, `score`, `correctCount`, `answeredCount`. |
| `/api/me/history/{id}` | `MatchDetail` nếu tài khoản đã tham gia trận. Trận ngoài lịch sử cá nhân trả `404 NOT_FOUND`. |
| `/api/admin/stats` | Object có `matches`, `participants`, `answers`, `correctAnswers`. |
| `/api/admin/stats/daily` | Mảng `{day, matches, players}` theo ngày tăng dần trong dữ liệu 14 ngày gần đây. |
| `/api/admin/history` | Tối đa 25 trận gần nhất. Mỗi dòng có `id`, `roomCode`, `quizTitle`, `questionCount`, `startedAt`, `finishedAt`, `playerCount`, `topScore`. |
| `/api/admin/history/{id}` | `MatchDetail`. Trận không tồn tại trả `404 NOT_FOUND`. |

`MatchDetail` gồm ba trường:

- `match` chứa `id`, `roomCode`, `quizTitle`, `questionCount`, `startedAt`, `finishedAt`.
- `players` là mảng các bản ghi `playerId`, `nickname`, `score`, `correctCount`, `answeredCount`.
- `answers` là mảng các bản ghi `playerId`, `questionNumber`, `selectedOption`, `correctOption`, `correct`, `points`, `responseMs` của cả trận. Route cá nhân kiểm tra quyền theo trận trước khi trả cấu trúc này.

`day` dùng định dạng `YYYY-MM-DD`. Các endpoint theo ngày chỉ trả những ngày có dữ liệu. Lịch sử Guest không được gắn lại vào tài khoản đăng ký sau đó.
