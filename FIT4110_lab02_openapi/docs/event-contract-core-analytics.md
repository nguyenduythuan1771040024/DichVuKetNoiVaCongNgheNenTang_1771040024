# Event Contract sơ bộ — Core Business ➔ Analytics (Pair 08)

> File này ghi nhận thỏa thuận sơ bộ về giao tiếp bất đồng bộ cho Pair 08 giữa Core Business Service và Analytics Service trong Lab 02. Phần đặc tả chi tiết AsyncAPI sẽ thực hiện ở Lab 03.

## 1. Thông tin dependency

- **Dependency số:** Pair 08 (Core Business ➔ Analytics)
- **Producer:** Core Business Service (`core-business`)
- **Consumer:** Analytics Service (`analytics`)
- **Cơ chế:** Queue async (RabbitMQ / Message Broker)
- **Event/topic dự kiến:** `core.alert.created`, `policy.decision.created`
- **Người ghi:** Nhóm Core Business
- **Ngày:** 2026-08-11

## 2. Mục đích nghiệp vụ

Core Business phát sự kiện khi có cảnh báo mới được khởi tạo (`core.alert.created`) hoặc khi có quyết định đánh giá chính sách (`policy.decision.created`). Analytics Service lắng nghe các sự kiện này để tổng hợp chỉ số KPI vận hành, đếm số lượt cảnh báo theo mức độ nghiêm trọng và tạo biểu đồ theo chuỗi thời gian.

## 3. Event name / topic

| Mục | Giá trị |
|---|---|
| Event name | `core.alert.created` |
| Topic / Exchange | `campus.events` |
| Routing Key | `core.alert.created` |
| Queue Name | `analytics_alert_queue` |
| Producer | `core-business` |
| Consumer | `analytics` |

## 4. Payload tối thiểu

```json
{
  "eventId": "0196fb3d-4ad7-7d1e-9f49-5d5148d2babc",
  "eventType": "core.alert.created",
  "occurredAt": "2026-08-11T10:00:00Z",
  "correlationId": "0196fb3d-4ad7-7d1e-9f49-5d5148d2babd",
  "source": "core-business",
  "data": {
    "alertId": "ALT-2026-001",
    "alertType": "UNAUTHORIZED_ACCESS",
    "severity": "HIGH",
    "message": "Phát hiện truy cập trái phép tại cổng chính",
    "relatedEventId": "0196fb3d-4ad7-7d1e-9f49-5d5148d2babd",
    "status": "OPEN"
  }
}
```

## 5. Ràng buộc cần thống nhất

| Vấn đề | Quyết định tạm thời |
|---|---|
| Event id có bắt buộc không? | Có (UUID format bắt buộc) |
| Có cần correlationId không? | Có (Để truy vết luồng từ Access Gate / AI Vision sang Analytics) |
| Có cho phép gửi trùng event không? | Có thể do retry mạng, Consumer (Analytics) phải xử lý Idempotency bằng `eventId` |
| Retry khi lỗi | Retry tối đa 3 lần với Exponential Backoff |
| Dead-letter queue | Đẩy vào `analytics_dlq` sau khi retry thất bại |

## 6. Issue chuyển sang Lab 03

1. Định nghĩa chi tiết schema AsyncAPI 3.0 cho RabbitMQ Exchange `campus.events`.
2. Cấu hình cơ chế Dead Letter Queue (DLQ) và Time-To-Live (TTL) cho tin nhắn bị lỗi.
