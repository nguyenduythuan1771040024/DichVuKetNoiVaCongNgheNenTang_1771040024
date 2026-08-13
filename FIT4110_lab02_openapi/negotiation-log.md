# Biên bản đàm phán hợp đồng API (Negotiation Log)

- Cặp đàm phán: Pair 02 (Core ↔ AI Vision), Pair 03 (Core ↔ Access Gate), Pair 10 (Access Gate ↔ Core Policy)
- Product: Product A / Product B
- Provider: Core Business Service (`core-business`)
- Consumer: Access Gate Service (`access-gate`), AI Vision Service (`ai-vision`), Analytics Service (`analytics`)
- Phiên: v1.0
- Ngày: 2026-08-11

---

## Issue #1: Quy chuẩn đặt tên trường (Naming Convention) cho ID và DTO

- **Raised by:** Consumer (`access-gate`)
- **Endpoint:** Tất cả các endpoint (`/alerts`, `/events`)
- **Concern:** Phía Access Gate đang sử dụng `snake_case` (`alert_id`, `source_service`), trong khi Core Business thiết kế mẫu theo `camelCase` (`alertId`, `sourceService`). Nếu không thống nhất sẽ gây lỗi `undefined` khi deserialize JSON.
- **Proposal:** Thống nhất sử dụng duy nhất một chuẩn đặt tên trường trong toàn bộ yêu cầu và phản hồi API.
- **Resolution:** Accepted.
- **Rationale:** Chuẩn RESTful OpenAPI khuyến nghị dùng `camelCase` nhất quán cho toàn bộ JSON Payload để tương thích với các thư viện client (JavaScript/TypeScript/Python Pydantic).
- **Impact:** Phía Consumer điều chỉnh DTO serialization chuyển hoàn toàn sang `camelCase`.

---

## Issue #2: Chuẩn hóa định dạng mốc thời gian (Timestamp Format)

- **Raised by:** Provider (`core-business`)
- **Endpoint:** `POST /alerts`, `GET /alerts`, `POST /events`
- **Concern:** Một số client gửi định dạng Unix Timestamp (millisecond integer), một số khác gửi string dạng `YYYY-MM-DD HH:mm:ss` không có múi giờ, gây sai lệch khi sắp xếp thứ tự audit log.
- **Proposal:** Bắt buộc sử dụng chuẩn **ISO 8601 (RFC 3339)** với múi giờ UTC (`YYYY-MM-DDTHH:mm:ssZ`).
- **Resolution:** Accepted.
- **Rationale:** Chuẩn ISO 8601 giúp tránh lệch múi giờ giữa các dịch vụ phân tán (Clock Drift) và tuân thủ định dạng `format: date-time` trong OpenAPI 3.1.
- **Impact:** Thêm ràng buộc regex / format validation vào `openapi.yaml`.

---

## Issue #3: Cấu trúc báo lỗi thống nhất theo chuẩn RFC 7807 (Problem Details)

- **Raised by:** Consumer (`ai-vision`)
- **Endpoint:** Tất cả các HTTP error responses (`400`, `401`, `403`, `404`, `409`, `422`, `500`)
- **Concern:** Provider trả về các chuỗi text đơn thuần khi gặp lỗi làm Consumer không parse được cấu trúc chi tiết trường bị sai để hiển thị cho UI.
- **Proposal:** Sử dụng đối tượng `Problem` chuẩn RFC 7807 (`type`, `title`, `status`, `detail`, `instance`, `errors`).
- **Resolution:** Accepted.
- **Rationale:** Giúp Consumer phân loại được lỗi hệ thống vs lỗi validation từng field thông qua mảng `errors: [{ field, code, message }]`.
- **Impact:** Khai báo `$ref: '#/components/schemas/Problem'` cho toàn bộ HTTP status code 4xx/5xx trong `openapi.yaml`.

---

## Issue #4: Cơ chế phân trang cho danh sách cảnh báo (Pagination Strategy)

- **Raised by:** Consumer (`analytics`)
- **Endpoint:** `GET /alerts`
- **Concern:** Sử dụng phân trang Offset (`page=1&limit=20`) dễ bị bỏ sót hoặc trùng lặp bản ghi khi có cảnh báo mới liên tục chèn vào Database (Data Drift).
- **Proposal:** Chuyển sang cơ chế phân trang dựa trên con trỏ (**Cursor-based Pagination**) dùng `cursor` và `limit`.
- **Resolution:** Accepted.
- **Rationale:** Cursor-based pagination đảm bảo tính nhất quán dữ liệu cao khi truy vấn chuỗi thời gian realtime và tối ưu hiệu năng truy vấn database.
- **Impact:** Cập nhật query parameters `Cursor` và `Limit` trong `components/parameters`, trả về `AlertPage` chứa `nextCursor` và `hasMore`.

---

## Issue #5: Định danh sự kiện trùng lặp và xử lý Idempotency

- **Raised by:** Provider (`core-business`)
- **Endpoint:** `POST /events`, `POST /alerts`
- **Concern:** Do mạng chập chờn, Consumer có thể gửi lặp lại cùng một sự kiện quẹt thẻ/cảnh báo nhiều lần khiến Core tạo nhiều bản ghi trùng.
- **Proposal:** Yêu cầu Consumer luôn truyền `eventId` dạng UUID v4/v7 trong payload. Core sẽ kiểm tra trùng lặp và trả về `409 Conflict` nếu `eventId` đã tồn tại.
- **Resolution:** Accepted.
- **Rationale:** Đảm bảo tính Idempotent cho các API ghi nhận sự kiện, tránh ghi nhận sai lệch dữ liệu cảnh báo an ninh.
- **Impact:** Định nghĩa `relatedEventId` và `eventId` bắt buộc `format: uuid` trong schema.

---

## Issue #6: Xác thực bảo mật API (Authentication & Authorization)

- **Raised by:** Provider (`core-business`)
- **Endpoint:** Tất cả các API trừ `GET /health`
- **Concern:** Các API không có cơ chế xác thực sẽ dẫn đến nguy cơ bị giả mạo sự kiện cảnh báo từ nguồn ngoài.
- **Proposal:** Bắt buộc sử dụng Bearer JWT Token ở HTTP Header `Authorization: Bearer <token>`.
- **Resolution:** Accepted.
- **Rationale:** Đảm bảo tính bảo mật inter-service giao tiếp giữa các dịch vụ trong Smart Campus.
- **Impact:** Cấu hình `securitySchemes.bearerAuth` và gán global `security: - bearerAuth: []` trong `openapi.yaml`.

---

# Chốt hợp đồng v1.0

- **Provider sign-off:** Core Business Service Team (`core-business`)
- **Consumer sign-off:** Access Gate / AI Vision / Analytics Team
- **Witness (GV/TA):** FIT4110 Instruction Team
- **Date:** 2026-08-11

---

## Ghi chú warning nếu Spectral còn cảnh báo

| Warning | Lý do chấp nhận tạm thời | Kế hoạch sửa |
|---|---|---|
| Không có warning lỗi | Đã kiểm tra qua Spectral linter đạt 0 error/0 warning | Hoàn thành |
