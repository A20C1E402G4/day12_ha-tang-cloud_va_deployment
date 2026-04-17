# Deployment

**Student:** dduyanhhoang@gmail.com  
**Date:** 17/4/2026  
**Platform:** Railway  
**Public URL:** https://delightful-ambition-production-37e2.up.railway.app

---

## Verification

```bash
# Root
curl https://delightful-ambition-production-37e2.up.railway.app/
# {"message":"AI Agent running on Railway!","docs":"/docs","health":"/health"}

# Health check
curl https://delightful-ambition-production-37e2.up.railway.app/health
# {"status":"ok","uptime_seconds":804.8,"platform":"Railway","timestamp":"2026-04-17T08:16:33.562930+00:00"}

# No auth — expect 401
curl -X POST https://delightful-ambition-production-37e2.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"hello"}'
# {"detail":"..."}  HTTP 401

# With auth — expect 200
curl -X POST https://delightful-ambition-production-37e2.up.railway.app/ask \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"question":"what is docker?"}'
# HTTP 200
```

---

## Deploy Steps

```bash
cd 03-cloud-deployment/railway
railway login
railway init
railway up
```

Set env vars in Railway dashboard: `AGENT_API_KEY`, `PORT`.
