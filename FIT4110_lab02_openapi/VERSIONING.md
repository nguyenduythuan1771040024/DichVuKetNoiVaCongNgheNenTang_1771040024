# Versioning & Changelog — Lab 02 OpenAPI & AsyncAPI Contract

## Phiên bản: v1.0.0
- **Ngày phát hành:** 2026-08-11
- **Mô tả:** Bản thiết kế hợp đồng API hoàn chỉnh (REST Sync & Queue Async) cho Core Business Service.

### Thay đổi chính:
1. **REST Sync Specification (`openapi.yaml`):**
   - Khai báo OpenAPI `3.1.0`.
   - Định nghĩa 9 REST endpoints: `/health`, `/alerts`, `/alerts/recent`, `/alerts/{alertId}`, `/access/check`, `/vision/face-match`, `/access/logs/recent`, `/events` và Webhook `alertCreated`.
   - Chuẩn hóa đối tượng báo lỗi `Problem` theo RFC 7807 cho tất cả mã lỗi HTTP 4xx/5xx.
   - Kiểm tra linter Spectral (`campus-spectral.yaml`) đạt clean 0 error/0 warning.

2. **Queue Async Specification (`asyncapi.yaml`):**
   - Khai báo AsyncAPI `2.6.0` cho RabbitMQ Message Broker (`amqp://localhost:5672`).
   - Định nghĩa Channels `campus.alerts` (`core.alert.created`) và `campus.telemetry` (`iot.sensor.measured`).
   - Đầy đủ Message Payload Schemas chuẩn UUID Idempotency và UTC ISO8601 timestamps.

3. **Biên bản đàm phán & Phân tích (`docs/` & `negotiation-log.md`):**
   - Hoàn thiện 6 Issue đàm phán kỹ thuật chi tiết.
   - Đầy đủ 3 file Event Contract sơ bộ cho các cặp Async: Pair 04 (`Core ➔ Notification`), Pair 05 (`IoT ➔ Core`), Pair 08 (`Core ➔ Analytics`).
