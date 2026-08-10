# Deployment Information

## Public URL

<https://day12-cohorts34-cloud-deployment-production-7ff4.up.railway.app/>

## Platform

Railway, deployed from `03-cloud-deployment/railway` using Nixpacks.

## Test Commands

### Health Check

```bash
curl -i https://day12-cohorts34-cloud-deployment-production-7ff4.up.railway.app/health
```

Expected status: `HTTP 200`; the JSON body contains `"status":"ok"` and `"platform":"Railway"`.

### API Test

```bash
curl -i -X POST \
  https://day12-cohorts34-cloud-deployment-production-7ff4.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"What is cloud deployment?"}'
```

Expected status: `HTTP 200`; the response includes the submitted question, a mock-agent answer, and `"platform":"Railway"`.

## Environment and Runtime Configuration

- `PORT`: injected automatically by Railway and consumed by the Uvicorn start command.
- No LLM secret is required because this deployment uses `utils/mock_llm.py`.
- Health check path: `/health` with a 30-second timeout.
- Restart policy: restart on failure, at most three retries.

Secrets must be set in Railway Variables and must never be committed to the repository if the mock LLM is replaced by a real provider.

## Verification

The public root, health, and ask endpoints were tested on 2026-08-10. All returned HTTP 200. At verification time, `/health` reported 2,986.5 seconds of uptime.

## Screenshots

Before final submission, add a Railway deployment dashboard screenshot as `screenshots/railway-dashboard.png` and a public endpoint test screenshot as `screenshots/railway-test.png`.
