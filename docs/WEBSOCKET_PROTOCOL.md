# Giao thức WebSocket và STOMP của QuizPulse

Trình duyệt dùng STOMP trên WebSocket cùng origin với REST. Tài liệu này mô tả giao thức của mã nguồn tham chiếu QuizPulse. Các thao tác tạo phòng, join, ready, start, kick, leave và close dùng [REST API](API_CONTRACT.md). STOMP phát trạng thái phòng và chuyển đáp án.

## Thiết lập kết nối

1. Gọi `GET /api/auth/session` qua REST để nhận danh tính phiên. Tạo hoặc tham gia phòng, hoặc gọi `GET /api/rooms/current` để lấy `JoinReceipt` của phòng hiện tại.
2. Mở WebSocket tại `/ws` bằng `ws://` cho HTTP và `wss://` cho HTTPS. Handshake dùng danh tính có sẵn từ cookie phiên hoặc cookie Guest. Không có danh tính hợp lệ thì handshake bị từ chối.
3. Gửi STOMP `CONNECT` với hai native header `roomCode` và `playerToken` từ `JoinReceipt`. Server kiểm tra token, thành viên đang hoạt động và danh tính từ handshake. Các frame `SEND` và `SUBSCRIBE` tiếp tục được kiểm tra theo phòng gắn với kết nối.
4. Sau khi kết nối, subscribe cả hai kênh bên dưới rồi gọi REST `GET /api/rooms/{code}/state` với `X-Player-Token` để đồng bộ trạng thái ban đầu hoặc bù sự kiện bị lỡ.

| Frame và destination | Dữ liệu |
|---|---|
| `SUBSCRIBE /user/queue/room` | Trạng thái phòng `ROOM_STATE` gửi riêng tới từng thành viên đang hoạt động. |
| `SUBSCRIBE /user/queue/acks` | `ANSWER_ACK`, `REMOVED`, `ROOM_CLOSED`. |
| `SEND /app/rooms/{code}/answer` | JSON đáp án. Mã trong destination phải khớp mã phòng chữ hoa đã xác thực. |

Server và frontend cấu hình heartbeat 10 giây theo cả hai chiều. Frontend tham chiếu có thời gian chờ kết nối 10 giây và thử kết nối lại sau 1,5 giây. Lỗi xác thực hoặc sai destination xảy ra ở tầng STOMP trước khi controller xử lý đáp án. Frontend xử lý sự kiện STOMP error và WebSocket close ở luồng kết nối.

## Trạng thái phòng

`ROOM_STATE` có dạng `{"type":"ROOM_STATE","state":{...}}`, trong đó `state` là snapshot chung của phòng. So với snapshot REST, sự kiện STOMP không có `self`.

| Trường trong `state` | Kiểu và ý nghĩa |
|---|---|
| `code`, `quizTitle`, `hostId` | Chuỗi mã phòng, tên bộ đề và ID Host. Mã này mang tên `roomCode` trong `JoinReceipt` và header `CONNECT`. |
| `phase` | `LOBBY`, `COUNTDOWN`, `QUESTION_OPEN`, `ANSWER_REVEAL`, `LEADERBOARD` hoặc `FINISHED` trong snapshot phòng đang tồn tại. Đóng phòng dùng sự kiện `ROOM_CLOSED`. |
| `secondsPerQuestion`, `questionCount`, `currentQuestionNumber`, `maxPlayers` | Số nguyên. `currentQuestionNumber` bằng 0 trước câu đầu và đếm từ 1 khi có câu. `maxPlayers` là 12. |
| `stateVersion` | Số tăng mỗi lần server broadcast trong phòng. Client bỏ snapshot có phiên bản thấp hơn snapshot đang giữ của phòng đó. |
| `serverNow`, `deadlineAt` | Thời gian Unix tính bằng mili giây. `serverNow` là thời điểm server tạo snapshot. `deadlineAt` là mốc của pha có thời hạn, bằng 0 khi chưa có hạn hoặc trận đã kết thúc. |
| `roundId` | Chuỗi ID của vòng, được tạo khi mở câu hỏi. Có thể là `null` trước câu đầu. |
| `historyId` | ID số của trận đã lưu, hoặc `null` nếu chưa có bản ghi. |
| `players` | Mảng người chơi, sắp theo `score` giảm dần rồi `nickname` tăng dần. |
| `question`, `answeredCount` | Xuất hiện khi có câu hỏi hiện tại và pha khác `COUNTDOWN`. `answeredCount` là số lượt nộp đang giữ cho vòng. |

Mỗi `players[]` có `id`, `nickname` kiểu chuỗi, `host`, `ready`, `online`, `active`, `answered` kiểu boolean và `score`, `correctCount`, `answeredCount` kiểu số. `answered` chỉ đúng trong `QUESTION_OPEN` với người đã nộp.

`question` có `id`, `index` (số thứ tự từ 1), `prompt`, `options` (4 chuỗi) và `difficulty`. `correctOption` chỉ xuất hiện trong `ANSWER_REVEAL`, `LEADERBOARD` và `FINISHED`.

Luồng trận là `LOBBY → COUNTDOWN → QUESTION_OPEN → ANSWER_REVEAL → LEADERBOARD`. Sau bảng điểm, server mở câu tiếp theo hoặc chuyển `FINISHED`. Server chạy đếm ngược 3 giây, công bố đáp án 3 giây và hiển thị bảng điểm 4 giây. Thời hạn mở câu hỏi dựa vào `secondsPerQuestion`. Frontend dùng `deadlineAt` và thời gian server để hiển thị đếm ngược. Server quyết định khi nào đóng câu hỏi và tính điểm.

## Gửi đáp án và đọc ACK

Gửi JSON tới `/app/rooms/{code}/answer` với `content-type:application/json`:

```json
{
  "roundId": "<roundId-từ-snapshot>",
  "requestId": "<id-do-client-tạo>",
  "option": 2
}
```

`option` là số nguyên từ 0 đến 3. `roundId` lấy từ snapshot và `requestId` là chuỗi có nội dung, dài tối đa 100 ký tự, dùng để ghép phản hồi với yêu cầu của client. Server dùng thời điểm nhận để so với `deadlineAt`. Lượt nộp hợp lệ đầu tiên của mỗi người trong mỗi vòng được giữ nguyên.

```json
{
  "type": "ANSWER_ACK",
  "requestId": "<id-do-client-tạo>",
  "accepted": true,
  "code": "ACCEPTED",
  "message": "Đã ghi nhận đáp án.",
  "option": 2
}
```

| `code` | `accepted` | Ý nghĩa |
|---|---|---|
| `ACCEPTED` | `true` | Server đã ghi nhận lựa chọn. |
| `ALREADY_SUBMITTED` | `true` | Đã nộp trong cùng vòng. `option` là lựa chọn ban đầu và giữ nguyên dù request mới mang lựa chọn khác. |
| `NOT_IN_ROOM` | `false` | Người chơi không còn hoạt động hoặc không còn phiên socket trong phòng khi service kiểm tra. |
| `BAD_ANSWER` | `false` | Thiếu hoặc sai `roundId`, `requestId` hoặc `option` theo kiểm tra dữ liệu của service. |
| `ROUND_CLOSED` | `false` | Sai `roundId` hiện tại hoặc câu hỏi không mở. |
| `DEADLINE_PASSED` | `false` | Server nhận sau `deadlineAt` khi trạng thái vẫn là `QUESTION_OPEN`. |

Với ACK từ chối thông thường, `option` là `null` và `requestId` có thể là `null` nếu không có trong command. Nếu controller bắt `ApiException`, ACK chỉ có `type`, `accepted`, `code` và `message`. `accepted: true` xác nhận đã ghi nhận đáp án. Điểm được cập nhật ở pha `ANSWER_REVEAL` và gửi trong snapshot.

## Rời phòng và kết nối lại

- `REMOVED` và `ROOM_CLOSED` có hai trường `type` và `message`. `REMOVED` gửi tới người bị kick hoặc người chơi thường chủ động rời. Khi đóng phòng, server gửi `ROOM_CLOSED` tới người chơi trong phòng.
- Mất kết nối kích hoạt luồng reconnect. Frontend subscribe lại hai kênh rồi lấy snapshot REST để khôi phục trạng thái, điểm, lựa chọn đã nộp trong vòng và thời gian còn lại. Khi tải lại trang, frontend dùng `/api/rooms/current` để tìm phòng theo danh tính. `204` nghĩa là danh tính hiện tại không có phòng để khôi phục.
- Guest dùng chung `QP_GUEST` giữa các tab cùng trình duyệt. Mặc định 15 giây khôi phục bắt đầu khi server phát hiện socket cuối cùng của Guest đã ngắt. Nếu vẫn còn tab kết nối, đồng hồ này chưa bắt đầu. Khi hết hạn, danh tính Guest cũ không còn quyền khôi phục phòng.
- Host mất kết nối quá 15 giây có thể được thay bằng một thành viên đang hoạt động, online và đã đăng nhập. Nếu không có người thay thế ở `LOBBY`, server đóng phòng. Thứ tự Host sau khi khôi phục được đọc từ `hostId` trong snapshot.
