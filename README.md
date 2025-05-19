# Kafka Microservices Lab

## 🧱 Tổng quan

Hệ thống này mô phỏng kiến trúc event-driven microservices sử dụng:

* Kafka + Kafka UI
* Flask Services: `order-service`, `payment-service`, `inventory-service`, `notifier-service`, `search-service`
* Redis để correlate event
* Elasticsearch + Kibana để lưu và tìm kiếm logs
* WebSocket Frontend UI để hiển thị noti realtime

---

## 🚀 Khởi chạy hệ thống

```bash
docker-compose up -d --build
```

Sau khi thành công:

* Kafka UI: [http://localhost:8080](http://localhost:8080)
* Kibana: [http://localhost:5601](http://localhost:5601)
* Order API: [http://localhost:5000](http://localhost:5000)
* Search API: [http://localhost:5100](http://localhost:5100)
* Frontend UI: [http://localhost:8000](http://localhost:8000)

---

## 🔁 Flow hệ thống

1. Gửi order (POST `/create_order`)
2. Order được gửi vào topic Kafka `order.created`
3. `payment-service` và `inventory-service` lắng nghe, xử lý xong thì gửi lại Kafka:

   * `payment.success`
   * `inventory.reserved`
4. `notifier-service` dùng Redis để correlate order\_id
5. Khi đủ 2 event → gửi noti qua WebSocket → frontend hiển thị

---

## 📦 API

### 🧾 1. `order-service`

#### `POST /create_order`

Tạo đơn hàng mới

**Request**

```json
{
  "order_id": 1,
  "item": "mouse",
  "quantity": 2000
}
```

**Response**

```json
{ "message": "Order created" }
```

Cũng gửi log vào Elasticsearch: `order-logs/_doc`

---

### 🔍 2. `search-service`

#### `GET /search?item=mouse`

Tìm kiếm đơn hàng theo item name từ Elasticsearch

**Response**

```json
[
  {
    "order_id": 1,
    "item": "mouse",
    "quantity": 2000,
    "timestamp": "..."
  },
  ...
]
```

---

## 📡 WebSocket UI

* Nhận thông báo từ `notifier-service`
* Dựa trên event Kafka khi đủ điều kiện: `payment.success` + `inventory.reserved`

---

## 📊 Kibana & Elasticsearch

### Tạo Data View (Index Pattern)

* Vào [http://localhost:5601](http://localhost:5601)
* Discover → Create data view
* Name: `order-logs*`
* Chọn field `timestamp` (hoặc tương ứng)

### Tìm kiếm (ví dụ):

```json
{
  "query": {
    "match": {
      "item": "mouse"
    }
  }
}
```

---

## 🧪 Test nhanh

### Tạo order

```bash
curl -X POST http://localhost:5000/create_order \
  -H "Content-Type: application/json" \
  -d '{"order_id": 1, "item": "mouse", "quantity": 2000}'
```

### Search

```bash
curl "http://localhost:5100/search?item=mouse"
```

---

## 📌 Kafka Topics

| Topic                | Description                |
| -------------------- | -------------------------- |
| `order.created`      | Gửi từ `order-service`     |
| `payment.success`    | Gửi từ `payment-service`   |
| `inventory.reserved` | Gửi từ `inventory-service` |

---

## ✅ Ghi chú thêm

* `bulk-order-generator` có thể được thêm vào để sinh nhiều dữ liệu test vào Elasticsearch
* `notifier-service` kết hợp Redis để correlate event theo `order_id`
* Tất cả service đều có log stdout xem qua `docker logs <container>`

---

Muốn nâng cấp lên FastAPI + Swagger UI? → hãy refactor `order-service` đầu tiên để tích hợp OpenAPI dễ dàng.
