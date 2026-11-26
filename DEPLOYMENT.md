# Deployment Information

## Public URL
https://k5-l3b-day14-caophamtankhai-2a202603883.onrender.com

## Platform
Render (Web Service — Docker runtime, Root Directory: `06-lab-complete`, Region: Singapore + Render Key Value / Redis cùng Region)

## Test Commands

Đặt biến môi trường trước khi chạy (không ghi thẳng API key vào tài liệu hoặc lịch sử lệnh):

```bash
BASE="https://k5-l3b-day14-caophamtankhai-2a202603883.onrender.com"
AGENT_API_KEY="<YOUR_API_KEY>"
```

### 1. Health Check (`/health` → 200)
```bash
curl -sS -o /dev/null -w '%{http_code}\n' "$BASE/health"
# Expected: 200
```

### 2. Readiness Check (`/ready` → 200, kiểm tra kết nối Redis)
```bash
curl -sS -o /dev/null -w '%{http_code}\n' "$BASE/ready"
# Expected: 200
```

### 3. API Test — Không có API key (`/ask` → 401)
```bash
curl -sS -o /dev/null -w '%{http_code}\n' -X POST "$BASE/ask" \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello"}'
# Expected: 401
```

### 4. API Test — Có API key hợp lệ (`/ask` → 200)
```bash
curl -sS -o /dev/null -w '%{http_code}\n' -X POST "$BASE/ask" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -d '{"question": "Hello"}'
# Expected: 200
```

## Environment Variables Set (trên Render Dashboard, không commit vào Git)
- `PORT` (Render tự inject)
- `ENVIRONMENT` (`production`)
- `REDIS_URL` (Internal Key Value URL của Render Redis)
- `AGENT_API_KEY` (secret đặt trên Render Dashboard)
- `RATE_LIMIT_PER_MINUTE` (`10`)
- `DAILY_BUDGET_USD` (`1.0`)

## Screenshots
- [Deployment dashboard](screenshots/dashboard.png)
- [Service running](screenshots/running.png)
- [Test results](screenshots/test.png)
