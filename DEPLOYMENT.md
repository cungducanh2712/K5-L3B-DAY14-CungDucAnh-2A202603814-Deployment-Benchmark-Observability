# Deployment Information

## Public URL
https://day14-production-agent-7rjo.onrender.com

## Platform
Render (Web Service — Docker + Render Key Value / Redis)

## Test Commands

### Health Check
```bash
curl -i https://day14-production-agent-7rjo.onrender.com/health
# Expected: HTTP 200 — {"status":"ok","environment":"production"}
```

### Readiness Check (Redis)
```bash
curl -i https://day14-production-agent-7rjo.onrender.com/ready
# Expected: HTTP 200 — {"status":"ready"}
```

### API Test (without key → 401)
```bash
curl -i -X POST https://day14-production-agent-7rjo.onrender.com/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello"}'
# Expected: HTTP 401 — {"detail":"Invalid or missing API key"}
```

### API Test (with authentication → 200)
```bash
curl -i -X POST https://day14-production-agent-7rjo.onrender.com/ask \
  -H "X-API-Key: YOUR_AGENT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"user_id": "test", "question": "Hello"}'
# Expected: HTTP 200
```

## Environment Variables Set
- `ENVIRONMENT` (`production`)
- `AGENT_API_KEY` (secret, cấu hình trực tiếp trên Render Dashboard)
- `REDIS_URL` (Internal Key Value URL của Redis trên Render)
- `RATE_LIMIT_PER_MINUTE` (`10`)
- `DAILY_BUDGET_USD` (`1.0`)

## Screenshots
- [Deployment dashboard](screenshots/dashboard.png)
- [Service running](screenshots/running.png)
- [Test results](screenshots/test.png)
