# Deployment Information

## Public URL
https://day-12-cloud-deployment-production.up.railway.app

## Platform
Railway

## Test Commands

### Health Check
```bash
curl https://day-12-cloud-deployment-production.up.railway.app/health
# Expected: {"status": "ok"}
```

**Sample output (tested):**
```json
{"status":"ok","version":"1.0.0","environment":"development","uptime_seconds":3252.4,"total_requests":24,"checks":{"redis":true,"llm":"mock"},"timestamp":"2026-04-17T16:05:04.752641+00:00"}
```

### API Test (with authentication)
```bash
KEY="[hãy ghi API key vào đây]"
curl -X POST https://day-12-cloud-deployment-production.up.railway.app/ask \
  -H "X-API-Key: $KEY" \
  -H "Content-Type: application/json" \
  -d '{"user_id": "test", "question": "Hello"}'
```

**Sample output (tested, key redacted):**
```json
{"user_id":"test_deploy_md","question":"Hello","answer":"Hello! How can I assist you today?","model":"gpt40-mini","timestamp":"2026-04-17T16:05:06.007687+00:00","history_count":0}
```

## Environment Variables Set
- PORT=""
- REDIS_URL=""
- AGENT_API_KEY=""

## Screenshots
- Screenshot: ![Deployment dashboard](https://res.cloudinary.com/dczdnu2ba/image/upload/v1776440665/railway_deploy_ple4o4.jpg)
- service running: ![service running](https://res.cloudinary.com/dczdnu2ba/image/upload/v1776442377/service_running_ndhxdc.jpg)
- test result: ![test result](https://res.cloudinary.com/dczdnu2ba/image/upload/v1776442376/test_result_f3o3fi.jpg)

