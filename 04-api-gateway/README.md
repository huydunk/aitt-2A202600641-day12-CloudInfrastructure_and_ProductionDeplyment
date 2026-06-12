# Section 4 — API Gateway & Security

## Mục tiêu học
- Hiểu tại sao cần lớp bảo vệ trước agent
- Implement API Key authentication
- Implement JWT authentication (nâng cao)
- Rate limiting và cost protection

---

## Ví dụ Basic — API Key Authentication

```
develop/
├── app.py              # Agent với API Key auth
└── requirements.txt
```

### Chạy thử
```bash
cd develop
pip install -r requirements.txt
AGENT_API_KEY=my-secret-key python app.py

# Test với key hợp lệ
curl -H "X-API-Key: my-secret-key" http://localhost:8000/ask \
     -X POST -H "Content-Type: application/json" \
     -d '{"question": "hello"}'

# Test không có key → 401
curl http://localhost:8000/ask -X POST \
     -H "Content-Type: application/json" \
     -d '{"question": "hello"}'
```

---

## Ví dụ Advanced — JWT + Rate Limiting + Cost Guard

```
production/
├── app.py              # Full security stack
├── auth.py             # JWT token logic
├── rate_limiter.py     # In-memory rate limiter
├── cost_guard.py       # Token budget và spending alerts
├── test_advanced.py    # Test suite
└── requirements.txt
```

### Chạy thử
```bash
cd production
pip install -r requirements.txt
python app.py

# Lấy JWT token
curl -X POST http://localhost:8000/auth/token \
     -H "Content-Type: application/json" \
     -d '{"username": "student", "password": "demo123"}'

# Dùng token
curl -H "Authorization: Bearer <token>" \
     http://localhost:8000/ask \
     -X POST -H "Content-Type: application/json" \
     -d '{"question": "what is docker?"}'

# Test rate limit: spam 20 requests liên tiếp
# Lấy token rồi gửi 20 requests — request 11+ sẽ bị chặn (429)
TOKEN=$(curl -s -X POST http://localhost:8000/auth/token \
     -H "Content-Type: application/json" \
     -d '{"username": "student", "password": "demo123"}' \
     | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

for i in $(seq 1 20); do
  printf "Request %2d: " $i
  curl -s -H "Authorization: Bearer $TOKEN" \
       -X POST http://localhost:8000/ask \
       -H "Content-Type: application/json" \
       -d '{"question": "what is docker?"}' \
       | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('answer','')[:60] if 'answer' in d else f'❌ {d.get(\"detail\",d)}')"
done
```

---

## Luồng bảo vệ

```
Request
  → Auth Check (401 nếu fail)
  → Rate Limit (429 nếu vượt quota)
  → Input Validation (422 nếu invalid)
  → Cost Check (402 nếu hết budget)
  → Agent (200 nếu mọi thứ OK)
```

---

## Câu hỏi thảo luận

1. Khi nào nên dùng API Key vs JWT vs OAuth2?
2. Rate limit nên đặt bao nhiêu request/phút cho một AI agent?
3. Nếu API key bị lộ, bạn phát hiện và xử lý như thế nào?

---

## Trả lời câu hỏi thảo luận (Answers)

**Q1 — When to use API Key vs JWT vs OAuth2?**

They solve the same question ("are you allowed in?") at increasing levels of
sophistication:

| Method | What it is | Best for | Trade-off |
|--------|-----------|----------|-----------|
| **API Key** | One shared secret string in a header (`X-API-Key`) | Internal tools, B2B, server-to-server, MVPs | No per-user identity, no expiry, no roles — if leaked, anyone can use it until you rotate it |
| **JWT** | A signed, self-contained, **time-limited** token carrying user id + role | Apps with users/logins, microservices, role-based access | You manage login + token refresh; revoking *before* expiry is harder (it's stateless) |
| **OAuth2** | A full delegation protocol — "Log in with Google/GitHub", third-party access | Letting users sign in with another provider, or granting *scoped* access to third-party apps | Most complex to set up; usually you use a provider/library, not hand-roll it |

*Rule of thumb:* start with an **API key** for a simple/internal agent (this lab's
`develop/`). Move to **JWT** once you have real users, roles, and need expiry (this
lab's `production/auth.py` — note the `role: user/admin` and 60-min expiry). Reach for
**OAuth2** only when you need "sign in with X" or to delegate access on a user's behalf.

**Q2 — How many requests/min should you allow for an AI agent?**

There's no universal number — set it from **cost, capacity, and abuse-prevention**, and
**tier it by user type**. Considerations:

- **Each request costs real money** (an LLM call) and takes real time/CPU — unlike a
  normal API, you can't allow thousands/sec.
- A **human** chatting realistically sends maybe a few requests/minute; anything much
  higher is automation or abuse.
- **Tier it:** free/anonymous users get a low limit, paying/admin users get more — exactly
  what this lab does: `rate_limiter_user = 10/min`, `rate_limiter_admin = 100/min`.

*Practical starting points:* ~**10–20 req/min** per free user, ~**60–100/min** per
paid/admin, and a separate **global** cap to protect total spend. Then watch real usage
and adjust. Always pair the rate limit with a **cost guard** (`cost_guard.py`) — rate
limiting caps *frequency*, the budget caps *total spend*; you need both.

**Q3 — If an API key is leaked, how do you detect and handle it?**

**Detect:**
- **Monitor for anomalies** — sudden spikes in requests, traffic from unexpected
  IPs/regions/times, or one key blowing through its rate limit / budget (the 429s and
  402s in this lab are your early-warning signals).
- **Secret scanning** — tools like GitHub secret scanning / gitleaks alert you when a key
  appears in a public repo.
- **Log per-key usage** so you can spot a key behaving abnormally.

**Handle (in order):**
1. **Revoke/rotate immediately** — invalidate the leaked key and issue a new one. This is
   the #1 action; everything else is secondary.
2. **Deploy the new key** to legitimate clients via env vars / secret manager (never
   hardcoded — see Section 1).
3. **Purge it from history** — remove from git history *and* rotate, because the old
   commit/caches keep the leaked value (deleting the commit alone is not enough).
4. **Investigate the blast radius** — check logs for what the leaked key did, and the
   cost guard for any budget damage.
5. **Prevent recurrence** — short-lived tokens (JWT expiry), per-key scopes/limits,
   secret scanning in CI, and keys in a vault not in code.

*Why JWT helps here:* a leaked JWT auto-expires (60 min in `auth.py`), so the damage
window is naturally bounded — unlike a static API key that works forever until you
notice and rotate it.
