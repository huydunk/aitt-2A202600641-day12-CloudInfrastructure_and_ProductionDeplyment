# Section 3 — Cloud Deployment Options

## 3 Tier: Chọn Platform Theo Nhu Cầu

| Tier | Platform | Khi nào dùng | Thời gian deploy |
|------|----------|-------------|-----------------|
| 1 | Railway, Render | MVP, demo, học | < 10 phút |
| 2 | AWS ECS, Cloud Run | Production | 15–30 phút |
| 3 | Kubernetes | Enterprise, large-scale | Vài giờ setup |

---

## railway/ — Deploy < 5 Phút

Không cần server config. Kết nối GitHub → Auto deploy.

```
railway/
├── railway.toml        # Railway config
├── Procfile            # Define start command
├── app.py              # Agent (Railway-ready)
└── requirements.txt
```

### Các bước deploy Railway:
1. `railway login` (hoặc qua browser)
2. `railway init`
3. `railway up`
4. Nhận URL dạng `https://your-app.up.railway.app`

---

## render/ — render.yaml (Infrastructure as Code)

Định nghĩa toàn bộ infrastructure trong 1 YAML file.

```
render/
├── render.yaml         # Khai báo service, env vars, disk
└── app.py
```

---

## production-cloud-run/ — GCP Cloud Run + CI/CD

Production-grade. Tự động build và deploy khi push code.

```
production-cloud-run/
├── cloudbuild.yaml     # CI/CD pipeline
├── service.yaml        # Cloud Run service definition
└── README.md           # Hướng dẫn chi tiết
```

---

## Câu hỏi thảo luận

1. Tại sao serverless (Lambda) không phải lúc nào cũng tốt cho AI agent?
2. "Cold start" là gì? Ảnh hưởng thế nào đến UX?
3. Khi nào nên upgrade từ Railway lên Cloud Run?

---

## Trả lời câu hỏi thảo luận (Answers)

**Q1 — Why isn't serverless (e.g. AWS Lambda) always good for an AI agent?**

Serverless means the platform runs your code **only when a request arrives**, then
shuts it down. Great for short, bursty tasks — but AI agents often clash with its
limits:

- **Slow cold starts.** Each new invocation may have to load a big dependency stack (or
  even a model) before answering → seconds of delay (see Q2).
- **Execution time caps.** Lambda kills a function after **15 minutes**. An agent doing
  a long multi-step reasoning loop, a big RAG retrieval, or a slow LLM call can exceed
  that and get cut off.
- **No warm state between calls.** Anything you loaded (model, cache, DB connection
  pool) is thrown away after each request, so you re-pay that cost constantly.
- **Cost flips for steady traffic.** Serverless is cheap when idle, but for an agent
  getting *constant* traffic, an always-on container is often cheaper and faster.

*Example:* a chatbot that gets a request every few seconds. On Lambda, many requests hit
cold instances and wait 3–5s each, and a long "research" query risks the 15-min cutoff.
On **Cloud Run with `min-instances=1`** (this section's `service.yaml`), one container
stays warm → instant responses, no time cap. That's why container platforms usually win
for agents.

**Q2 — What is a "cold start" and how does it affect UX?**

A **cold start** is the delay when the platform has **no warm instance ready** and must
build one from scratch before it can answer: pull the image → boot the container → start
your app → load libraries/model. Only then does request #1 get served.

- **Warm start:** an instance is already running → reply in ~50ms.
- **Cold start:** no instance running → user waits **2–10+ seconds** on that first
  request while everything spins up.

*Example:* your Railway agent has been up ~3.9 hours (`uptime_seconds` in `/health`), so
it's **warm** — `/ask` replies instantly. But if the platform scaled it to **zero**
during idle time, the *next* visitor would trigger a cold start and stare at a spinner
for several seconds. For a chat UI that feels broken or "laggy."

**The fix used in this section:** keep one instance always warm —
`minScale: "1"` in `service.yaml` and `--min-instances=1` in `cloudbuild.yaml` (the
comment literally reads `# Tránh cold start`). Trade-off: you pay for 1 always-on
instance, but users never hit a cold start.

**Q3 — When should you upgrade from Railway to Cloud Run?**

Railway (Tier 1) is for shipping fast; Cloud Run (Tier 2) is for running a real product.
Upgrade when you outgrow Tier 1 and need one or more of these:

| You need… | Railway | Cloud Run |
|-----------|---------|-----------|
| **Fine resource control** (exact CPU/memory, request concurrency) | limited | ✅ `cpu`, `memory`, `containerConcurrency: 80` |
| **Real CI/CD** (auto test → build → deploy on every push) | basic | ✅ `cloudbuild.yaml` pipeline |
| **Proper secret management** | dashboard field | ✅ Google Secret Manager (`secretKeyRef`) |
| **Predictable scaling** (min/max instances, cold-start control) | basic | ✅ `minScale`/`maxScale` |
| **Region / compliance control, GCP ecosystem** | limited | ✅ pick regions, IAM, VPC |

*Example signals it's time:* your demo became a product with paying users; you're getting
enough traffic that you need to guarantee no cold starts and cap costs with
`max-instances`; security review demands secrets in a vault instead of a dashboard; or
you want every `git push` to main to automatically run tests, build the multi-stage
image (Section 2), and deploy — with an automatic rollback path. If you just need a
live URL for a class demo (what you did above), **Railway is the right tool** — don't
over-engineer.
