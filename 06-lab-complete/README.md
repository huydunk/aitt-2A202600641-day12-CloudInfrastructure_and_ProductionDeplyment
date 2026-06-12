# Lab 12 — Complete Production Agent

Kết hợp TẤT CẢ những gì đã học trong 1 project hoàn chỉnh.

---

## 🚀 Live Deployment

**Live URL:** https://lab12-06-production-586c.up.railway.app

| Endpoint | Live check |
|----------|-----------|
| `GET /health` | https://lab12-06-production-586c.up.railway.app/health → `{"status":"ok",...}` |
| `GET /` | https://lab12-06-production-586c.up.railway.app/ → API info |
| `POST /ask` | Protected — requires `X-API-Key` header |

Deployed on **Railway**, built from the multi-stage **Dockerfile** in this folder.
Runs in `development` mode with the mock LLM (no API key required).

### How it was deployed (Railway CLI, from a monorepo subfolder)

```bash
railway login
railway init                       # project: lab12
railway add --service lab12-06     # new service inside the project
# In the Railway dashboard: set the service Root Directory = "06-lab-complete"
railway up --service lab12-06      # builds 06-lab-complete/Dockerfile
railway domain --service lab12-06  # → public URL
```

**Two gotchas worth noting** (both fixed in this folder's config):
1. **Monorepo root directory** — `railway up` uploads from the *git repo root*, so the
   service's **Root Directory** must be set to `06-lab-complete`, or Railway can't find
   the Dockerfile.
2. **`$PORT` expansion** — Railway runs the start command without a shell, so
   `--port $PORT` is passed literally. The `railway.toml` `startCommand` is wrapped in
   `sh -c '... --port ${PORT:-8000} ...'` so the platform's port is substituted.

---

## Checklist Deliverable

- [x] Dockerfile (multi-stage, < 500 MB)
- [x] docker-compose.yml (agent + redis)
- [x] .dockerignore
- [x] Health check endpoint (`GET /health`)
- [x] Readiness endpoint (`GET /ready`)
- [x] API Key authentication
- [x] Rate limiting
- [x] Cost guard
- [x] Config từ environment variables
- [x] Structured logging
- [x] Graceful shutdown
- [x] Public URL ready (Railway / Render config)
- [x] **Deployed live** → https://lab12-06-production-586c.up.railway.app

---

## Cấu Trúc

```
06-lab-complete/
├── app/
│   ├── main.py         # Entry point — kết hợp tất cả
│   ├── config.py       # 12-factor config
│   ├── auth.py         # API Key + JWT
│   ├── rate_limiter.py # Rate limiting
│   └── cost_guard.py   # Budget protection
├── Dockerfile          # Multi-stage, production-ready
├── docker-compose.yml  # Full stack
├── railway.toml        # Deploy Railway
├── render.yaml         # Deploy Render
├── .env.example        # Template
├── .dockerignore
└── requirements.txt
```

---

## Chạy Local

```bash
# 1. Setup
cp .env.example .env

# 2. Chạy với Docker Compose
docker compose up

# 3. Test
curl http://localhost/health

# 4. Lấy API key từ .env, test endpoint
API_KEY=$(grep AGENT_API_KEY .env | cut -d= -f2)
curl -H "X-API-Key: $API_KEY" \
     -X POST http://localhost/ask \
     -H "Content-Type: application/json" \
     -d '{"question": "What is deployment?"}'
```

---

## Deploy Railway (< 5 phút)

```bash
# Cài Railway CLI
npm i -g @railway/cli

# Login và deploy
railway login
railway init
railway variables set OPENAI_API_KEY=sk-...
railway variables set AGENT_API_KEY=your-secret-key
railway up

# Nhận public URL!
railway domain
```

---

## Deploy Render

1. Push repo lên GitHub
2. Render Dashboard → New → Blueprint
3. Connect repo → Render đọc `render.yaml`
4. Set secrets: `OPENAI_API_KEY`, `AGENT_API_KEY`
5. Deploy → Nhận URL!

---

## Kiểm Tra Production Readiness

```bash
python check_production_ready.py
```

Script này kiểm tra tất cả items trong checklist và báo cáo những gì còn thiếu.
