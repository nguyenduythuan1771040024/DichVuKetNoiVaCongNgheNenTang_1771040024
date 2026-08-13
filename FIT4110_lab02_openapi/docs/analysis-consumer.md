# Phân tích yêu cầu — vai Consumer (Dành cho Consumer tích hợp với Core Business)

- Cặp đàm phán: Pair 02 (Core ↔ AI Vision), Pair 03 (Core ↔ Access Gate), Pair 10 (Access Gate ↔ Core Policy)
- Product: Product A / Product B
- Consumer service: Access Gate Service (`access-gate`), AI Vision Service (`ai-vision`), Analytics Service (`analytics`)
- Provider service: Core Business Service (`core-business`)
- Người viết: Nhóm tích hợp Consumer
- Ngày: 2026-08-11

---

## 1. Resource Consumer cần nhận/gửi

| Resource | Consumer dùng để làm gì? | Field bắt buộc với Consumer | Field có thể tùy chọn |
|---|---|---|---|
| `CreateAlertRequest` | Gửi yêu cầu tạo cảnh báo từ AI Vision / Access Gate sang Core | `sourceService`, `alertType`, `severity`, `message` | `relatedEventId` |
| `Alert` | Nhận kết quả bản ghi cảnh báo đã được tạo từ Core | `id`, `sourceService`, `alertType`, `severity`, `status` | `resolvedAt`, `relatedEventId` |
| `AlertPage` | Consumer Analytics lấy danh sách thống kê các cảnh báo | `items`, `nextCursor`, `hasMore` | `nextCursor` (khi hết trang) |

---

## 2. API Consumer cần gọi

| Method | Path | Lúc nào gọi? | Kỳ vọng response |
|---|---|---|---|
| GET | `/health` | Khi khởi động hoặc health check định kỳ | `200 OK` với `{ status: "ok" }` |
| POST | `/alerts` | Khi phát hiện truy cập trái phép hoặc AI phát hiện nguy cơ | `201 Created` kèm Header `Location` chứa URI của alert |
| GET | `/alerts/recent` | Khi Analytics / Dashboard cần lấy danh sách cảnh báo tức thời | `200 OK` chứa mảng `items` các Alert gần nhất |
| GET | `/alerts/{alertId}` | Khi người dùng click xem thông tin chi tiết một cảnh báo | `200 OK` thông tin chi tiết đối tượng Alert |

---

## 3. Error case Consumer cần xử lý

| Status | Consumer hiểu là gì? | Consumer sẽ xử lý thế nào? |
|---:|---|---|
| 400 | Request payload sai schema hoặc sai tham số query | Ghi log lỗi, dừng gửi payload hỏng và báo cho developer |
| 401 | Bearer Token chưa được truyền hoặc đã hết hạn | Thực hiện cấp lại/refresh token và thử lại request |
| 403 | Service account của Consumer không có quyền truy cập endpoint | Log cảnh báo security và dừng gọi API |
| 404 | `alertId` không tồn tại trong hệ thống Core | Hiển thị thông báo "Không tìm thấy cảnh báo" trên giao diện |
| 409 | Request bị trùng lặp `eventId` | Bỏ qua việc gửi lại (dữ liệu đã được ghi nhận trước đó) |
| 422 | Vi phạm quy tắc nghiệp vụ phía Core | Parse đối tượng `Problem` để lấy chi tiết trường bị lỗi |
| 500 | Core Business Service gặp sự cố nội bộ | Thực hiện Retry cơ chế Exponential Backoff (tối đa 3 lần) |

---

## 4. Giả định bổ sung

- **Giả định 1:** Consumer gửi request phải truyền đính kèm Header `Authorization: Bearer <valid_token>`.
- **Giả định 2:** Core Business hỗ trợ phân trang Cursor-based cho API `GET /alerts`, Consumer sẽ dùng `nextCursor` để lấy trang kế tiếp.
- **Giả định 3:** Trong trường hợp mất kết nối mạng tạm thời, Consumer sẽ lưu trữ tạm sự kiện vào local buffer và retry sau.

---

## 5. Câu hỏi cho Provider

1. Core Business Service có hỗ trợ nhận Webhook đăng ký nhận sự kiện cảnh báo realtime không?
2. Giới hạn số lượng cảnh báo tối đa trả về trong API `GET /alerts` một lần là bao nhiêu (limit mặc định)?
3. Thời gian sống (TTL) của Bearer JWT token dành cho Consumer là bao nhiêu lâu?

---

## 6. Rủi ro tích hợp

| Rủi ro | Tác động | Đề xuất xử lý |
|---|---|---|
| Provider thay đổi cấu trúc schema đột ngột | Consumer bị crash do parse JSON thất bại | Thống nhất chuẩn RFC 7807 `Problem` và dùng OpenAPI Spec |
| Mạng chập chờn gây mất gói tin | Cảnh báo không được ghi nhận | Thêm header Idempotency-Key và thực hiện retry |
