# Thông Tin Deploy — Checkpoint 5

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Tô Minh Đức |
| Mã học viên | 2A202601043 |
| Repo | DAY12-2A202601043-ToMinhDuc |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://day12-agent.onrender.com |
| Platform | Render (Blueprint Docker) |
| Ngày deploy | 2026-08-10 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `AGENT_API_KEY` | ✅ | đặt trong dashboard, không nằm trong repo |
| `REDIS_URL` | ✅ | Render Redis add-on (day12-redis) |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i https://day12-agent.onrender.com/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i https://day12-agent.onrender.com/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST https://day12-agent.onrender.com/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST https://day12-agent.onrender.com/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST https://day12-agent.onrender.com/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-test" \
    -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật Trên Cloud Render

Output kết quả gọi API công khai:

```json
1. GET /health -> 200 OK {"status":"ok","version":"1.0.0","environment":"production"}
2. GET /ready -> 200 OK {"ready":true}
3. POST /ask (No Key) -> 401 Unauthorized {"detail":"Invalid or missing API key. Include header: X-API-Key: <key>"}
4. POST /ask (With Key) -> 200 OK {"answer":"[Mock LLM Response]","user_id":"sv-test","history_length":0,"cost_usd":0.0001,"tokens":{"in":5,"out":12}}
5. POST /ask (Rate limited) -> 429 Too Many Requests {"detail":"rate limit exceeded"}
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên Render Dashboard (Blueprints `✓ Deployed`)
- `screenshots/health.png` — kết quả gọi `/health` trên Render public domain
