# QuizPulse

**Đề tài đã đăng ký:** Quiz Multiplayer thời gian thực sử dụng WebSocket.

QuizPulse là dự án bài tập lớn môn Lập trình mạng. Mục tiêu của dự án là cho phép nhiều người cùng chơi trắc nghiệm theo phòng, gửi đáp án theo thời gian thực và xem bảng xếp hạng. Máy chủ quản lý trạng thái phòng, thời hạn trả lời và điểm số.

## Thông tin nhóm

**Tên nhóm:** [Điền tên nhóm]

| Thành viên | Họ và tên | MSSV | Email | Phần đảm nhiệm |
|---|---|---|---|---|
| Người A | [Họ tên] | [MSSV] | [Email] | Khởi tạo repository và soạn hợp đồng giao tiếp. Phát triển backend và kiểm thử backend. |
| Người B | [Họ tên] | [MSSV] | [Email] | Review Phase 01. Phát triển frontend và kiểm thử tích hợp. |

## Kiến trúc dự kiến

Trình duyệt gọi REST API qua HTTP và trao đổi sự kiện thời gian thực qua STOMP trên WebSocket. Nginx phục vụ giao diện và chuyển tiếp yêu cầu tới Spring Boot. MySQL lưu tài khoản, bộ câu hỏi và kết quả trận đấu.

Xem [sơ đồ kiến trúc QuizPulse](statics/diagram.svg).

## Cấu trúc repository

- `source/client/` chứa phần giao diện trình duyệt.
- `source/server/` chứa phần backend.
- `statics/` chứa sơ đồ và tài nguyên minh họa.
- `INSTRUCTION.md` chứa hướng dẫn của giảng viên.

## Quy trình Git

1. Cập nhật `main` trước khi tạo nhánh cho một công việc mới.
2. Commit các thay đổi có cùng mục đích và kiểm tra nội dung trước khi push.
3. Mở Pull Request từ nhánh công việc vào `main`. Thành viên còn lại review trước khi merge.
4. Sau khi PR được merge, cập nhật `main` trên máy và dọn nhánh đã hoàn thành.

## Trạng thái dự án

Dự án đang ở Phase 01, gồm khởi tạo repository và thống nhất hợp đồng REST/WebSocket. Hướng dẫn build và chạy sẽ được bổ sung sau khi có bản chạy thử.
