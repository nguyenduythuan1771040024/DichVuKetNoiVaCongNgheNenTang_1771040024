# Phân tích yêu cầu — vai Provider (Core Business Service)

- Cặp đàm phán: Pair 02 (Core ↔ AI Vision), Pair 03 (Core ↔ Access Gate), Pair 10 (Access Gate ↔ Core Policy)
- Product: Product A / Product B
- Provider service: Core Business Service (`core-business`)
- Consumer service: AI Vision Service (`ai-vision`), Access Gate Service (`access-gate`), Analytics Service (`analytics`)
- Người viết: Nhóm Core Business
- Ngày: 2026-08-11

---

## 1. Resource chính

| Resource | Mô tả | Thuộc tính bắt buộc | Thuộc tính tùy chọn |
|---|---|---|---|
| `Alert` | Đối tượng cảnh báo bất thường trong hệ thống Smart Campus | `id`, `sourceService`, `alertType`, `severity`, `message`, `status`, `createdAt`, `resolvedAt` | `relatedEventId` |
| `CreateAlertRequest` | Payload yêu cầu tạo cảnh báo từ Consumer | `sourceService`, `alertType`, `severity`, `message` | `relatedEventId` |
| `CampusEvent` | Sự kiện nghiệp vụ đầu vào (Polymorphic) | `eventType`, `eventId`, `timestamp` | `metric`, `value`, `gateId`, `cardId`, `decision` |
| `Problem` | Cấu trúc phản hồi lỗi chuẩn RFC 7807 | `type`, `title`, `status`, `detail` | `instance`, `errors` |

---

## 2. Action/API dự kiến

| Method | Path | Mục đích | Consumer gọi khi nào? |
|---|---|---|---|
| GET | `/health` | Kiểm tra trạng thái hoạt động của Core Service | Mọi Consumer kiểm tra liveness/readiness |
| POST | `/alerts` | Tiếp nhận và tạo bản ghi cảnh báo mới | AI Vision / Access Gate gọi khi phát hiện bất thường nghiêm trọng |
| GET | `/alerts` | Lấy danh sách tất cả cảnh báo có phân trang cursor | Analytics Service gọi lấy dữ liệu lịch sử cảnh báo |
| GET | `/alerts/recent` | Lấy danh sách các cảnh báo gần nhất | Dashboard / Analytics gọi cập nhật realtime |
| GET | `/alerts/{alertId}` | Xem thông tin chi tiết một cảnh báo theo UUID | Admin UI / Analytics gọi xem chi tiết |
| POST | `/events` | Tiếp nhận sự kiện nghiệp vụ thô (Sensor/Access) | IoT Ingestion / Access Gate đẩy event trực tiếp |

---

## 3. Error case

| Status | Tình huống | Response body dự kiến |
|---:|---|---|
| 400 | Payload JSON không khớp schema (thiếu field bắt buộc, sai pattern) | `Problem` (`type: .../validation`) |
| 401 | Thiếu hoặc sai Bearer JWT token xác thực | `Problem` (`type: .../unauthorized`) |
| 403 | Token không có quyền thực hiện thao tác tạo/sửa policy | `Problem` (`type: .../forbidden`) |
| 404 | Không tìm thấy `alertId` yêu cầu | `Problem` (`type: .../not-found`) |
| 409 | Xung đột Idempotency key hoặc sự kiện trùng lặp (`eventId`) | `Problem` (`type: .../conflict`) |
| 422 | Dữ liệu đúng định dạng JSON nhưng vi phạm quy tắc nghiệp vụ | `Problem` (`type: .../unprocessable`) |
| 500 | Lỗi xử lý nội bộ server / mất kết nối Database | `Problem` (`type: .../internal-server-error`) |

---

## 4. Giả định bổ sung

- **Giả định 1:** Định dạng thời gian `createdAt`, `resolvedAt`, `timestamp` bắt buộc dùng định dạng chuẩn ISO8601 UTC (ví dụ: `2026-08-11T08:00:00Z`).
- **Giả định 2:** Định danh `alertId` và `eventId` bắt buộc sử dụng định dạng UUID v4/v7 hợp lệ.
- **Giả định 3:** Toàn bộ request gọi đến các API trừ `/health` bắt buộc phải kèm theo Header `Authorization: Bearer <token>`.

---

## 5. Câu hỏi cho Consumer

1. Tần suất Consumer (Analytics / Dashboard) gọi lấy danh sách `/alerts/recent` là bao nhiêu (polling rate hay dùng webhook)?
2. Phía Consumer có cần truyền thêm thông tin chi tiết thiết bị (`metadata` mở rộng) khi gửi yêu cầu `POST /alerts` không?
3. Khi xảy ra lỗi 409 (Idempotency conflict), Consumer sẽ tự động retry hay bỏ qua sự kiện?

---

## 6. Rủi ro tích hợp

| Rủi ro | Tác động | Đề xuất xử lý |
|---|---|---|
| Tên field không thống nhất (`alert_id` vs `alertId`) | Consumer/Provider parse lỗi `undefined` | Thống nhất chuẩn camelCase trong toàn bộ `openapi.yaml` |
| Consumer đẩy dữ liệu dồn dập (Spike Traffic) | Server Core quá tải / chậm phản hồi | Áp dụng Rate Limiting và trả về `429 Too Many Requests` |
| Sai lệch mốc thời gian giữa các hệ thống (Clock Drift) | Lệch thứ tự sự kiện trong audit log | Yêu cầu toàn bộ service đồng bộ NTP Server và truyền UTC ISO8601 |
