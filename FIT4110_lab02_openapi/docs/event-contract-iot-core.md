# Event Contract sơ bộ — IoT Ingestion ➔ Core Business (Pair 05)

> File này ghi nhận thỏa thuận sơ bộ về giao tiếp bất đồng bộ cho Pair 05 giữa IoT Ingestion Service và Core Business Service.

## 1. Thông tin dependency

- **Dependency số:** Pair 05 (IoT Ingestion ➔ Core Business)
- **Producer:** IoT Ingestion Service (`iot-ingestion`)
- **Consumer:** Core Business Service (`core-business`)
- **Cơ chế:** Queue async (RabbitMQ / Mosquitto MQTT)
- **Event/topic dự kiến:** `iot.sensor.measured`
- **Người ghi:** Nhóm Core Business
- **Ngày:** 2026-08-11

## 2. Mục đích nghiệp vụ

IoT Ingestion tiếp nhận dữ liệu chuỗi thời gian từ các cảm biến phần cứng (nhiệt độ, độ ẩm, chuyển động) và phát sự kiện `iot.sensor.measured` vào Queue. Core Business lắng nghe để kiểm tra nếu nhiệt độ vượt 35°C hoặc phát hiện chuyển động bất thường thì ra quyết định khởi tạo Cảnh báo (`Alert`).

## 3. Event name / topic

| Mục | Giá trị |
|---|---|
| Event name | `iot.sensor.measured` |
| Topic / Exchange | `campus.telemetry` |
| Routing Key | `sensor.data.#` |
| Queue Name | `core_sensor_processing_queue` |
| Producer | `iot-ingestion` |
| Consumer | `core-business` |

## 4. Payload tối thiểu

```json
{
  "eventId": "0196fb3d-4ad7-7d1e-9f49-5d5148d2babc",
  "eventType": "iot.sensor.measured",
  "occurredAt": "2026-08-11T10:00:00Z",
  "source": "iot-ingestion",
  "data": {
    "deviceId": "SENSOR-001",
    "metric": "temperature",
    "value": 38.5,
    "unit": "celsius"
  }
}
```

## 5. Ràng buộc cần thống nhất

| Vấn đề | Quyết định tạm thời |
|---|---|
| Event id có bắt buộc không? | Có (UUID format bắt buộc) |
| Tần suất gửi dữ liệu | Tối đa 1 packet/giây cho mỗi thiết bị cảm biến |
| Xử lý trễ dữ liệu | Lọc các bản ghi có timestamp quá 5 phút so với mốc thời gian hệ thống |
| Retry khi lỗi | Đẩy vào `core_sensor_dlq` nếu Core Service xử lý thất bại nhiều lần |

## 6. Issue chuyển sang Lab 03

1. Thống nhất cơ chế nén gói tin (Payload Compression) nếu số lượng thiết bị cảm biến tăng cao.
2. Thiết lập chính sách TTL (Time-To-Live) cho các message cảm biến cũ trong Queue.
