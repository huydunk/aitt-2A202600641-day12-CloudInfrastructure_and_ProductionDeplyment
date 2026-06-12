# Section 1 — Từ Localhost Đến Production

## Mục tiêu học
- Hiểu tại sao "it works on my machine" là vấn đề
- Nhận ra sự khác biệt giữa dev và production environment
- Áp dụng 4 nguyên tắc 12-factor cơ bản

---

## Ví dụ Basic — Agent "Kiểu Localhost"

```
develop/
├── app.py          # ❌ Anti-patterns: hardcode secrets, no config, no health check
└── requirements.txt
```

### Chạy thử
```bash
cd develop
pip install -r requirements.txt
python app.py
# Truy cập: http://localhost:8000
```

### Những vấn đề trong code này:
1. API key hardcode trong code
2. Không có health check endpoint
3. Debug mode bật cứng
4. Không xử lý SIGTERM gracefully
5. Config không đến từ environment

---

## Ví dụ Advanced — 12-Factor Compliant Agent

```
production/
├── app.py          # ✅ Clean: config from env, health check, graceful shutdown
├── config.py       # ✅ Centralized config management
├── .env.example    # ✅ Template — không commit .env thật
└── requirements.txt
```

### Chạy thử
```bash
cd production
pip install -r requirements.txt
cp .env.example .env
# Sửa .env nếu cần
python app.py
```

### So sánh với Basic:

| | Basic (❌) | Advanced (✅) |
|--|-----------|--------------|
| Config | Hardcode trong code | Đọc từ env vars |
| Secrets | `api_key = "sk-abc123"` | `os.getenv("OPENAI_API_KEY")` |
| Port | Cố định `8000` | Từ `PORT` env var |
| Health check | Không có | `GET /health` |
| Shutdown | Tắt đột ngột | Graceful — hoàn thành request hiện tại |
| Logging | `print()` | Structured JSON logging |

---

## Câu hỏi thảo luận

1. Điều gì xảy ra nếu bạn push code với API key hardcode lên GitHub public?
2. Tại sao stateless quan trọng khi scale?
3. 12-factor nói "dev/prod parity" — nghĩa là gì trong thực tế?

---

## Trả lời câu hỏi thảo luận (Answers)

**Q1 — What happens if you push code with a hardcoded API key to public GitHub?**
The key is leaked the moment you push — automated bots constantly scan public GitHub
for secrets and find them within **minutes**. An attacker can then use your key to run
up huge bills (LLM/cloud APIs), steal data, or access connected systems. Deleting the
commit does **not** help: it stays in git history and in GitHub's caches/forks. The only
real fix is to **revoke (rotate) the key immediately**. This is exactly why
`develop/app.py` hardcoding `OPENAI_API_KEY = "sk-..."` is an anti-pattern, and why
`production/config.py` reads it from `os.getenv("OPENAI_API_KEY")` instead — the secret
lives in the environment, never in code.

**Q2 — Why is statelessness important when scaling?**
"Scaling" means running **many copies** of the agent to handle more traffic. Requests
get spread across those copies, so request #1 from a user might hit copy A and request
#2 might hit copy B. If a copy stores session data **in its own memory** (state), copy B
won't know what copy A remembered — the user gets inconsistent behavior, and if a copy
crashes its data is lost. A **stateless** app keeps no per-user data in local memory;
shared state lives **outside** the app in something all copies share (e.g. Redis). Then
any copy can serve any request identically, and you can add/remove copies freely. (You
build this shared-state design in Section 5.)

**Q3 — 12-factor says "dev/prod parity" — what does it mean in practice?**
Keep your **development** environment as close as possible to **production** so code that
works in dev also works in prod. In practice: same language version (Python 3.11 in
both), same dependencies (pinned in `requirements.txt`), same OS, same backing services,
and the same config mechanism (env vars). The smaller the gap, the fewer
"worked-on-my-machine-but-broke-in-prod" surprises. **Docker (Section 2) is the main
tool for achieving parity** — it packages the exact same environment everywhere.
