# Event Contract sơ bộ — Core Business ➔ Notification (Pair 04)

> File này ghi nhận thỏa thuận sơ bộ về giao tiếp bất đồng bộ cho Pair 04 giữa Core Business Service và Notification Service trong Lab 02. Phần đặc tả chi tiết AsyncAPI sẽ thực hiện ở Lab 03.

## 1. Thông tin dependency

- **Dependency số:** Pair 04 (Core Business ➔ Notification)
- **Producer:** Core Business Service (`core-business`)
- **Consumer:** Notification Service (`notification`)
- **Cơ chế:** Queue async (RabbitMQ / Message Broker)
- **Event/topic dự kiến:** `core.alert.created`
- **Người ghi:** Nhóm Core Business
- **Ngày:** 2026-08-11

## 2. Mục đích nghiệp vụ

Khi Core Business phát hiện vi phạm nghiệp vụ và tạo một cảnh báo mới, nó phát sự kiện `core.alert.created` vào Queue. Notification Service đăng ký nhận sự kiện này để gửi tin nhắn thông báo đẩy (Telegram Bot, Email SMTP, Discord Webhook) tới đội ngũ bảo vệ/quản trị viên.

## 3. Event name / topic

| Mục | Giá trị |
|---|---|
| Event name | `core.alert.created` |
| Topic / Exchange | `campus.alerts` |
| Routing Key | `alert.high.#` / `alert.critical.#` |
| Queue Name | `notification_dispatch_queue` |
| Producer | `core-business` |
| Consumer | `notification` |

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
    "targetGroup": "security_team",
    "channels": ["telegram", "email"]
  }
}
```

## 5. Ràng buộc cần thống nhất

| Vấn đề | Quyết định tạm thời |
|---|---|
| Event id có bắt buộc không? | Có (UUID format bắt buộc) |
| Có cần correlationId không? | Có (Để truy vết nguyên nhân tạo alert) |
| Có cho phép gửi trùng event không? | Notification Service phải khử trùng lặp (Deduplication) để tránh gửi tin rác tới người dùng |
| Retry khi lỗi | Notification tự động retry gửi tin tối đa 3 lần nếu API Telegram/SMTP gặp lỗi tạm thời |
| Dead-letter queue | Đẩy vào `notification_dlq` nếu tất cả các kênh thông báo thất bại |

## 6. Issue chuyển sang Lab 03

1. Định nghĩa đặc tả AsyncAPI cho các kênh thông báo `telegram`, `email`, `webhook`.
2. Cấu hình cơ chế chống gửi trùng tin nhắn (Rate Limit & Deduplication Window).
