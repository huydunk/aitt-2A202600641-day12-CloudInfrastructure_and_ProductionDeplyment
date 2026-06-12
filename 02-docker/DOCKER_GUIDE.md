# Docker — Beginner's Walkthrough (Section 2)

> You have **zero Docker experience** — this guide assumes that. It explains every
> step of *this* lab: **what you do**, **what each command means**, and **why it
> matters in the bigger picture** of deploying an AI agent to the cloud.
>
> Read top-to-bottom once. Then keep it open while you run the commands.

---

## Part 0 — The mental model (read this first, 3 minutes)

### The problem Docker solves

Your agent runs fine on **your** laptop. But your laptop has:
- A specific Python version (3.11)
- Specific installed libraries (`fastapi`, `uvicorn`, …)
- Specific OS settings, environment variables, file paths

When you copy the code to a **cloud server**, that server has *none* of that.
Result: *"works on my machine"* → crashes in production. This is the **dev/prod gap**.

### The Docker idea: ship the whole box, not just the code

A **container** is a sealed box that holds your code **plus everything it needs to
run** — the right Python, the right libraries, the right OS files. You build the box
*once*, and it runs **identically** on your laptop, your teacher's laptop, and a
Google Cloud server. No more "works on my machine."

### Three words you must not confuse

| Word | What it is | Real-world analogy |
|------|-----------|--------------------|
| **Dockerfile** | A text **recipe** — a list of steps to build the box | The recipe written on paper |
| **Image** | The built box, frozen and ready to ship | The finished frozen meal in the box |
| **Container** | A **running** copy of an image | The meal heated up and being eaten |

Flow: **You write a `Dockerfile`** → `docker build` turns it into an **image** →
`docker run` turns the image into a live **container**.

You can run many containers from one image (that's how you "scale" — Section 5).

### One critical idea: the "build context"

When you build, Docker bundles up a folder and sends it to the Docker engine. That
folder is the **build context**. In this lab the context is the **project root**
(the top folder), *not* the `02-docker/develop` folder. That's why every command
below ends with a `.` (meaning "context = current folder") and why the Dockerfile
paths look like `COPY 02-docker/develop/requirements.txt`. **This trips up every
beginner — see Part 3.**

---

## Part 1 — What you are expected to DO in this section

From `02-docker/README.md`, your goals are:

1. ✅ Understand what a container is and why you need it (Part 0 above).
2. ✅ **Run the basic example** — build one image, run one container. (Part 2)
3. ✅ **Run the production example** — a multi-stage build + a 4-service stack with
   Docker Compose. (Part 4)
4. ✅ **Compare the two image sizes** and understand *why* production is ~5× smaller. (Part 5)
5. ✅ Be able to answer the 3 discussion questions at the end of the README. (Part 6)

You are **not** writing Docker from scratch here — the files already exist. Your job
is to **run them, observe what happens, and understand each line.** Section 6 is
where you'll combine everything yourself.

> **Before you start:** open Docker Desktop and wait until it says *"Engine running."*
> Every `docker` command talks to that engine. If it's not running, nothing works.

---

## Part 2 — The BASIC example, step by step

📁 Files: `02-docker/develop/` → `app.py`, `Dockerfile`, `requirements.txt`, `.dockerignore`

### 2a. First, read the Dockerfile as a recipe

#### The mental model: a Dockerfile is a to-do list for an assistant

Imagine you hire an **assistant** and hand them a **brand-new, empty Linux computer**.
Your job is to write step-by-step instructions so they set it up to run your app. The
Dockerfile *is that instruction list.* Each **CAPITAL word is one type of order** you
can give the assistant.

The assistant reads your list **top to bottom, one line at a time**, and does exactly
what each line says. When they're finished, they shrink-wrap the whole computer into a
box (the image).

There are only **6 order-words** in your basic Dockerfile:

| Word | The order you're giving | Everyday equivalent |
|------|------------------------|---------------------|
| `FROM` | "Don't start from nothing — **start with this pre-made computer**" | Buying a laptop that already has the OS + Python installed |
| `WORKDIR` | "**Go into this folder.** Do all future work here." | `cd /app` (and create it if missing) |
| `COPY` | "**Copy this file from my computer into the box**" | Dragging a file from your USB stick onto the new laptop |
| `RUN` | "**Run this command now, while setting up the box**" | Typing a command and pressing Enter during setup |
| `EXPOSE` | "Just a note: this app will use port 8000" | A sticky-note label (does nothing by itself) |
| `CMD` | "**When someone later turns this box on, run this**" | Setting what program auto-launches at startup |

**The distinction that clears up most confusion — `RUN` vs `CMD`:** the difference is
*WHEN* they happen.

| | When it happens | Think of it as |
|--|----------------|----------------|
| `RUN` | **While building** the box (`docker build`) | Setup work — installing things, making folders. Happens once, ahead of time. |
| `CMD` | **When you start** the box (`docker run`) | The "on switch" — what the app does every time it boots up. |

**The trailing `.` in `COPY <source> .`** means "into the current folder I'm standing
in" (whatever `WORKDIR` set — here `/app`). This is *not* the same `.` as in the
`docker build ... .` command. Same symbol, two different jobs:
- In a **Dockerfile**, `COPY x .` → "put it *here inside the box* (`/app`)."
- In the **terminal**, `docker build ... .` → "the ingredients are *here on my computer*."

#### Now your `02-docker/develop/Dockerfile`, line by line (read each as an order to the assistant):

```dockerfile
FROM python:3.11
```
🗣️ *"Assistant, start with a computer that already has Python 3.11 and Linux on it."*
**What:** start from a pre-made image that already has Python 3.11 installed on a
Linux OS. **Why:** you never build a box from bare metal — you start from someone
else's box and add your stuff. `python:3.11` is the *full* version (~1 GB, has every
tool). Production will use a *slim* version instead — that's the whole lesson of this
section.

```dockerfile
WORKDIR /app
```
🗣️ *"Make a folder called `/app` and go stand inside it. Everything from now on happens here."*
**What:** "from now on, work inside the `/app` folder inside the box." Creates it if
missing. **Why:** keeps your code in one tidy place instead of scattered at the root.
Every command after this runs relative to `/app`.

```dockerfile
COPY 02-docker/develop/requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
```
🗣️ *"Copy my shopping-list file `requirements.txt` into this folder — then run the install command and install every library on the list."*
**What:** copy *only* the dependency list into the box, then install those libraries.
The `.` means "into the current WORKDIR (`/app`)". `--no-cache-dir` tells pip not to
keep its download cache → smaller image.
**Why copy requirements BEFORE the code?** This is **discussion question #1** and the
single most important Docker optimization. Docker builds in **layers** and **caches**
each one. If your code changes but `requirements.txt` doesn't, Docker **reuses** the
cached "install libraries" layer instead of re-downloading everything — turning a
3-minute rebuild into 3 seconds. If you copied all the code first, *any* code edit
would invalidate the cache and force a full reinstall.

```dockerfile
COPY 02-docker/develop/app.py .
```
🗣️ *"Copy my actual program file, `app.py`, into this folder too."*
**What:** now copy the actual application code in. **Why after:** see above — code
changes often, deps change rarely, so code goes in a *later* (cheaper to rebuild) layer.

```dockerfile
RUN mkdir -p utils
COPY utils/mock_llm.py utils/
```
🗣️ *"Make a sub-folder called `utils`, then copy the helper file `mock_llm.py` into it."*
**What:** create a `utils/` folder and copy the shared **mock LLM** into it. **Why:**
`app.py` does `from utils.mock_llm import ask`. The mock means the whole lab runs
**offline with no API key**. Notice the source path is `utils/mock_llm.py` — relative
to the project root, *not* `02-docker/develop`. That's the build-context thing again.

```dockerfile
EXPOSE 8000
```
🗣️ *"Stick a label on the box saying 'this app uses port 8000.'"*
**What:** documentation that says "this app listens on port 8000." **Why:** purely a
label for humans/tools — it does **not** actually open the port. The real port mapping
happens at `docker run` time with `-p`. (Yes, it's slightly misleading; that's Docker.)

```dockerfile
CMD ["python", "app.py"]
```
🗣️ *"You're done setting up — shrink-wrap the box. And remember: whenever someone turns this box on later, the first thing it should do is run `python app.py`."*
**What:** the default command to run when the container **starts**. **Why:** an image
is inert; this line is what makes the container *do* something — here, launch your
FastAPI server. `app.py`'s last lines start `uvicorn` on `0.0.0.0:8000`.
(`0.0.0.0` = "accept connections from outside the container," not just localhost —
important, or `-p` mapping wouldn't reach it.)

### 2b. Now run it

Open a terminal **at the project root** (the folder above `02-docker`):

```powershell
# 1. BUILD the image from the recipe
docker build -f 02-docker/develop/Dockerfile -t agent-develop .
```
- `docker build` — turn a Dockerfile into an image.
- `-f 02-docker/develop/Dockerfile` — *which* recipe file to use (`-f` = file).
- `-t agent-develop` — *tag* (name) the resulting image `agent-develop` (`-t` = tag).
- `.` — **the build context is the current folder** (project root). This is the `.`
  that everything hinges on.

You'll see Docker run each `Step` from the Dockerfile. First build is slow (downloads
the ~1 GB Python image); later builds are fast (cache).

```powershell
# 2. See the image and its size
docker images agent-develop
```
Lists your image with its size — note it (~800 MB–1 GB). You'll compare this later.

```powershell
# 3. RUN a container from the image
docker run -p 8000:8000 -d agent-develop
```
- `docker run` — start a container from an image.
- `-p 8000:8000` — **port mapping**: connect port 8000 on *your machine* (left) to
  port 8000 *inside the container* (right). Without this, the app is sealed inside the
  box and you can't reach it.
- `-d` — **detached**: run in the background so your terminal stays free.
- `agent-develop` — which image to run.

It prints a long **container ID**. That's the handle for this running box.

```powershell
# 4. See what's running
docker ps
```
Lists running containers (`ps` = "process status," borrowed from Linux). You should
see `agent-develop` with status `Up` and the port mapping. (`docker ps -a` also shows
*stopped* ones.)

```powershell
# 5. Test it — does the agent respond?
curl http://localhost:8000/health
```
This hits the `/health` endpoint in `app.py`. Expect `{"status":"ok",...}`. 🎉
**This is the payoff:** your agent is running inside a sealed Linux container, and you
reached it from your Windows host through the port mapping.

```powershell
# 6. (Optional) Step INSIDE the running container to look around
docker exec -it <container-id> sh
```
- `docker exec` — run a command in an *already-running* container.
- `-it` — `-i` keeps input open, `-t` gives you a terminal → together you get an
  interactive shell.
- `sh` — the shell program to start.
Now your prompt is *inside the box*. Try `ls` (see `/app`), `ls utils`, then `exit` to
leave. **Why this matters:** when production breaks, this is how you debug what's
actually inside the container vs. what you *think* is there.

```powershell
# 7. Clean up when done
docker stop <container-id>     # stop the running container
docker rm <container-id>       # delete the stopped container
```
**Why:** stopped containers and old images pile up and eat disk. Get in the habit.
(`docker ps` to get the ID; you only need the first few characters.)

---

## Part 3 — The build-context gotcha (the #1 beginner mistake)

Notice you build from the **project root** with `.`, and the Dockerfile says
`COPY 02-docker/develop/requirements.txt`, **not** `COPY requirements.txt`.

**Why:** the `COPY` source path is always relative to the **build context** (the `.`),
which is the project root — *not* to where the Dockerfile lives. The agent and the
shared `utils/mock_llm.py` live in different folders, so the context must be the root
that contains both.

❌ If you `cd` into `02-docker/develop` and run `docker build .` there, the build
**fails** — Docker can't find `utils/mock_llm.py`, because that folder isn't in the
context. **Always build from the project root in this lab.**

---

## Part 4 — The PRODUCTION example, step by step

📁 Files: `02-docker/production/` → `Dockerfile` (multi-stage), `docker-compose.yml`,
`nginx/nginx.conf`, `main.py`

This adds three big production ideas. Let's take them one at a time.

### 4a. Idea #1 — Multi-stage build (small, safe images)

#### First, the problem multi-stage solves

Remember the basic Dockerfile from Part 2? It started `FROM python:3.11` — the **full**
Python (~1 GB). Why so big? Because the full image ships with a whole workshop of
**build tools**: compilers (`gcc`), C libraries, headers, etc. Some Python packages
(like `numpy` or `psycopg2`) can't just be downloaded ready-made — they have to be
**compiled from source code on the spot**, and that needs those heavy tools.

Here's the catch: **you need those tools to *install* the libraries, but you don't need
them to *run* your app.** It's like needing a full carpentry workshop to *build* a
chair — but once the chair is built, you just want the chair, not the saws and
sawdust. If you ship the whole workshop with every chair, your delivery truck is
needlessly huge.

A single-stage Dockerfile ships the workshop *and* the chair. **Multi-stage ships only
the chair.**

#### The analogy: two assistants, two computers

Picture **two assistants**, each with their own fresh computer:

- **Assistant #1 (the "builder")** has a fully-stocked workshop computer. Their *only*
  job is to install/compile all the Python libraries. Their computer ends up bloated
  and messy — full of compilers and leftover build junk. **We will throw their whole
  computer away.**
- **Assistant #2 (the "runtime")** has a clean, minimal computer. Their job is to run
  the finished app. They **walk over to Assistant #1, grab *only* the finished
  libraries**, bring them back, add the app code, and that's it. No tools, no mess.

The image you actually ship is **Assistant #2's clean computer.** Assistant #1's
bloated one is discarded the moment the build finishes. That's the entire trick of
multi-stage: *do the dirty work in a throwaway computer, keep only the result.*

> Each `FROM` line in the Dockerfile = "here's a new fresh computer for a new
> assistant." Two `FROM` lines = two assistants. The `AS builder` / `AS runtime` part
> just gives each assistant a **name** so we can refer back to them.

#### Stage 1 — `builder` (Assistant #1's messy workshop)

```dockerfile
FROM python:3.11-slim AS builder
```
🗣️ *"Assistant #1, take a small Linux+Python computer. I'll call you 'builder.'"*

```dockerfile
RUN apt-get update && apt-get install -y gcc libpq-dev && rm -rf /var/lib/apt/lists/*
```
🗣️ *"Install the heavy workshop tools — the `gcc` compiler and the `libpq` database
library — because some Python packages need to be compiled from source."*
(`apt-get` is the Linux app-installer, like an app store on the command line. The
`rm -rf ...` at the end just tidies up the download lists afterward.)

```dockerfile
COPY 02-docker/production/requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt
```
🗣️ *"Copy in my shopping list, then install every library on it — and put them all in
your personal folder (`--user` → `/root/.local`) so they're easy to hand off later."*

The key detail is **`--user`**: normally pip scatters libraries across many system
folders, which are a pain to copy out. `--user` makes pip dump *everything into one
tidy folder* (`/root/.local`). Think of it as Assistant #1 putting all the finished
chairs **on one pallet by the door**, ready for Assistant #2 to grab in a single trip.

✅ At this point Assistant #1's computer has the libraries — **but also all the
compilers and build junk.** That's why we don't ship it.

#### Stage 2 — `runtime` (Assistant #2's clean computer — *this* is what ships)

```dockerfile
FROM python:3.11-slim AS runtime
```
🗣️ *"Assistant #2, take a brand-new small computer. I'll call you 'runtime.' This is
the one we actually deliver."*
Notice it's a **fresh** `python:3.11-slim` — none of Assistant #1's mess exists here.

```dockerfile
RUN groupadd -r appuser && useradd -r -g appuser appuser
```
🗣️ *"Create a low-privilege user named `appuser` to run the app — don't run as the
all-powerful root account."* (Security; more below.)

```dockerfile
WORKDIR /app
```
🗣️ *"Make `/app` and stand inside it."* (Same as Part 2.)

```dockerfile
COPY --from=builder /root/.local /home/appuser/.local
```
🗣️ *"Walk over to Assistant #1 ('builder') and grab **only** the finished-libraries
folder. Bring it back to my computer. Leave all their tools and junk behind."*

**This single line is the heart of multi-stage.** `COPY --from=builder` means "copy
from *the other assistant's computer*, not from my own computer." It grabs the one
pallet of finished libraries (`/root/.local`) and nothing else — no `gcc`, no caches,
no source code. That's how the final image stays tiny.

```dockerfile
COPY 02-docker/production/main.py .
RUN mkdir -p /app/utils
COPY utils/mock_llm.py /app/utils/mock_llm.py
```
🗣️ *"Now copy in the actual app code and its helper file."* (Plain file copies from
*your* computer, exactly like Part 2.)

```dockerfile
RUN chown -R appuser:appuser /app
USER appuser
```
🗣️ *"Give `appuser` ownership of the app folder, then switch to being `appuser` — from
now on everything runs as that limited user, not root."*

```dockerfile
ENV PATH=/home/appuser/.local/bin:$PATH
ENV PYTHONPATH=/app
```
🗣️ *"Tell the system where to find the libraries you carried over, so Python can
actually import them."*
`ENV` sets an environment variable inside the box. `PATH` is the list of folders the
system searches for programs — we add the `.local/bin` folder so the copied tools are
found. Without this line, Python wouldn't see the libraries Assistant #2 carried over.

```dockerfile
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "2"]
```
🗣️ *"When this box is turned on, start the web server with 2 worker processes."*

#### Why this matters (the payoff)

The final shipped image is **only Assistant #2's clean computer**: a slim Python base
+ the libraries + your code. **No compilers, no build junk.** Result: **~160 MB instead
of ~800 MB.** Smaller image = faster cloud deploys, cheaper storage/bandwidth, and a
**smaller attack surface** (fewer tools inside = fewer things an attacker can abuse).

#### Two more production touches in this stage

- **`USER appuser` (non-root):** by default a container runs as `root` — the all-powerful
  admin. If an attacker finds a bug and breaks in, they inherit whatever user the app
  runs as. Run as `root` → they own the whole box. Run as a powerless `appuser` → they're
  stuck in a sandbox. Always drop to a non-root user in production.
- **`HEALTHCHECK ...`:** Docker periodically calls `/health` itself. If it fails
  repeatedly, the platform knows the container is sick and can restart/replace it. This
  is the foundation of Section 5 (reliability).
- **`--workers 2`:** runs 2 copies of the app *inside one container* so it can handle 2
  requests truly simultaneously instead of one at a time.

### 4b. Idea #2 — Docker Compose (run many services with one command)

#### First, the problem Compose solves

So far, one container = one `docker run`. But a real agent isn't *just* your code. It
needs a **cache**, a **database**, and a **gateway** out front — that's **4 separate
containers**. Doing that by hand means: run 4 commands, in the right *order* (the agent
must wait for the database to be ready), wire them onto the same network so they can
talk, set up shared storage, and remember every flag. Miss one detail and it breaks.

**Docker Compose** lets you write all of that down **once** in a single file
(`docker-compose.yml`) and bring the whole system up with **one command**:
`docker compose up`.

#### The analogy: Compose is the restaurant manager

Think of opening a restaurant. You don't personally start each worker one by one. You
hand the **manager** a single staffing plan that says: *"We need a chef, a stock-room,
a fridge, and a host at the door. Start the fridge and stock-room first; only let the
chef start once those are ready; the host is the only one allowed to talk to
customers."* The manager reads that plan and starts everyone correctly.

`docker-compose.yml` **is that staffing plan**, and `docker compose up` is you telling
the manager **"open the restaurant."** Each entry under `services:` is one worker (one
container).

Open `02-docker/production/docker-compose.yml`. It defines **4 services (workers)**:

| Service | Image | Their job (restaurant role) |
|---------|-------|------------------------------|
| **agent** | built from *your* Dockerfile | The **chef** — your FastAPI AI agent, does the real work |
| **redis** | `redis:7-alpine` | The **fridge** — fast cache for sessions & rate-limit counters (Section 4/5) |
| **qdrant** | `qdrant/qdrant` | The **stock-room** — vector database for RAG (storing embeddings) |
| **nginx** | `nginx:alpine` | The **host at the door** — reverse proxy & load balancer |

#### Reading the `agent` service line by line

```yaml
agent:
  build:
    context: ../..
    dockerfile: 02-docker/production/Dockerfile
    target: runtime
```
🗣️ *"Manager, for the 'agent' worker: don't pull a ready-made image — **build** one
from my Dockerfile. The ingredients are at the project root (`../..`), and only build
the `runtime` stage (skip the throwaway `builder`)."*
This is the multi-stage Dockerfile from 4a. `target: runtime` says "we only ship
Assistant #2's clean computer." `context: ../..` is the same build-context idea as the
`.` in Part 2 — just written as a path because the compose file lives two folders deep.

```yaml
  environment:
    - REDIS_URL=redis://redis:6379/0
    - QDRANT_URL=http://qdrant:6333
```
🗣️ *"Tell the chef where the fridge and stock-room are."*
**Look closely:** the address of Redis is literally `redis` — *the service name*. Inside
a Compose setup, every worker can reach another just by its name, like calling a
coworker across the kitchen ("Hey, fridge!"). You never deal with IP addresses — Compose
runs a tiny internal phone book (DNS) that maps `redis` → the right container. This is
**service-name DNS**, and it's why the agent code can just say `redis:6379`.

```yaml
  depends_on:
    redis:
      condition: service_healthy
    qdrant:
      condition: service_healthy
```
🗣️ *"Don't let the chef start until the fridge and stock-room report they're actually
ready."*
This controls **startup order**. Without it, the agent might boot, immediately try to
reach a database that's still starting, and crash. `condition: service_healthy` means
"wait not just until they've *started*, but until their **health check passes**." (Those
health checks are defined on the `redis` and `qdrant` services further down.)

```yaml
  networks:
    - internal
```
🗣️ *"Put the chef on the private staff-only network."*
A **network** is the walkie-talkie channel the workers share. `internal` is private — the
outside world can't reach it directly. Notice the agent has **no `ports:` line**, so it
is *not* reachable from your computer at all...

```yaml
  nginx:
    ports:
      - "80:80"
```
🗣️ *"...only the host at the door (`nginx`) gets a public entrance — port 80."*
**This is deliberate security.** The chef (agent) is sealed in the back; the only way a
customer reaches them is *through* the host (Nginx). One controlled front door instead
of four exposed ones. (`80:80` is the same `outside:inside` port mapping as the `-p` flag
from Part 2.)

#### Two more things to spot

- **`volumes:` (`redis_data`, `qdrant_data`)** — containers are **disposable**: delete
  one and everything inside it vanishes. A **volume** is a storage locker that lives
  *outside* the container and survives restarts, so your database data isn't lost. (This
  is **discussion question #3**'s territory.) Think: the fridge's *contents* are kept in a
  walk-in cooler that stays even if you replace the fridge unit.
- **`restart: unless-stopped`** — "if this worker crashes, automatically restart it
  (unless I deliberately stopped it)." Basic self-healing.

### 4c. Idea #3 — Nginx as the front door

#### The analogy: Nginx is the receptionist at the front desk

Your agent *could* face the internet directly — but that's like letting visitors wander
straight into your office. Instead you put a **receptionist at the front desk**. Every
visitor talks to the receptionist first, and the receptionist:

- **directs them to the right person** (and spreads visitors across multiple staff if
  you have several) → *load balancing*,
- **turns away anyone who shows up too many times per second** → *rate limiting*,
- **enforces building rules** (no tailgating, show ID) → *security headers*,
- **hands a polite "please come back later" slip** when the office is overwhelmed →
  the clean `429 Too Many Requests` response.

Your Python agent shouldn't waste effort doing any of that — let the specialist
(Nginx) handle the door. `nginx/nginx.conf` is the receptionist's rule book.

#### The key lines in `nginx/nginx.conf`

```nginx
upstream agent_backend {
    server agent:8000;
    # server agent_2:8000;   ← add more here to load-balance across copies
}
```
🗣️ *"The people I forward visitors to are reachable at `agent:8000`. (When we scale,
list more here and I'll spread visitors across them.)"*
Again notice `agent` — the service name from the compose file. The receptionist finds
the chef by name via the same internal phone book.

```nginx
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;
...
limit_req zone=api_limit burst=20 nodelay;
limit_req_status 429;
```
🗣️ *"Track each visitor by their address. Allow about 10 requests per second each; if
someone floods me, turn them away with a `429`."*
This protects the agent (and your LLM bill) from being hammered. `burst=20` allows short
spikes before it starts rejecting.

```nginx
proxy_pass http://agent_backend;
proxy_set_header X-Real-IP $remote_addr;
proxy_read_timeout 60s;
```
🗣️ *"Forward this visitor to a chef, and pass along who they really are. Wait up to 60s
for a reply (LLM calls can be slow)."*
`proxy_pass` is the actual hand-off. The `X-Real-IP` header tells the agent the
*visitor's* address (otherwise the agent would only see the receptionist's).

```nginx
add_header X-Frame-Options "DENY";
server_tokens off;
```
🗣️ *"Enforce safety rules, and don't reveal which Nginx version I'm running (less info
for attackers)."*

#### Where it all fits

The full request journey in this stack:

```
visitor → Nginx (front desk) → agent (chef) → Redis / Qdrant (fridge / stock-room)
```

You'll go much deeper on rate limiting and auth in Section 4, and on load-balancing
across many agent copies in Section 5 — here, the goal is just to *see where Nginx sits*
and why the agent hides behind it.

### 4d. Run the whole stack

> Compose here reads `env_file: .env.local`. If that file doesn't exist yet, create an
> empty one in `02-docker/production/` first (`New-Item .env.local`), or Compose may
> complain.

From the project root:

```powershell
# Start everything (add -d to run in background)
docker compose -f 02-docker/production/docker-compose.yml up
```
- `docker compose up` — read the YAML, build/pull every image, create the network and
  volumes, and start all services in dependency order. **One command, whole system.**
- `-f ...` — point at the compose file (since it's not in the current folder).
- Leave off `-d` the first time so you can *watch the logs* of all 4 services
  interleaved. Press `Ctrl+C` to stop.

```powershell
# In another terminal: see the services
docker compose -f 02-docker/production/docker-compose.yml ps

# Test the agent THROUGH Nginx — note: port 80, no :8000
curl http://localhost/health

# Tear it all down (stops & removes containers + network)
docker compose -f 02-docker/production/docker-compose.yml down
```
Notice the test hits `http://localhost/health` (port 80 = Nginx), **not**
`:8000`. You're going through the front door now — exactly like real production.

---

## Part 5 — Compare the sizes (the lesson lands here)

```powershell
docker images | Select-String agent
```
You'll see something like:
```
agent-develop     ~800 MB    ← python:3.11  (full), single-stage
agent-production  ~160 MB    ← python:3.11-slim + multi-stage
```
**Big picture:** same app, **5× smaller** image. In the cloud that means faster
deploys, cheaper storage/bandwidth, and a smaller attack surface. *This size
difference is the entire point of Section 2.*

---

## Part 6 — Answers to the README discussion questions

**Q1: Why `COPY requirements.txt` + `RUN pip install` BEFORE `COPY . .`?**
Docker caches each layer. Dependencies change rarely, code changes constantly. Putting
deps in an *earlier* layer means editing your code doesn't bust the (slow) install
layer's cache — rebuilds drop from minutes to seconds.

**Q2: What goes in `.dockerignore`, and why are `venv/` and `.env` important?**
It lists files to keep *out* of the build context/image. `venv/` is your local virtual
environment — huge, and built for *your* OS, so it's useless (and harmful) inside a
Linux container; excluding it shrinks the context and speeds builds. `.env` holds
**secrets** (keys, passwords) — baking those into an image that gets pushed to a
registry would leak them. Never put secrets in an image.

**Q3: If the agent needs to read files from disk, how do you mount a volume?**
Use a **volume** so data survives container restarts. CLI: `docker run -v mydata:/app/data ...`.
Compose: under the service, `volumes: ["mydata:/app/data"]` and declare `mydata:` in
the top-level `volumes:` block — exactly how `redis_data` and `qdrant_data` work in
this lab's compose file.

---

## Cheat sheet — the commands you'll actually type

| Command | Plain-English meaning |
|---------|----------------------|
| `docker build -f <file> -t <name> .` | Cook the recipe into an image named `<name>`, context = here |
| `docker images` | List my images (and their sizes) |
| `docker run -p 8000:8000 -d <img>` | Start a container, expose port 8000, run in background |
| `docker ps` / `docker ps -a` | List running / all containers |
| `docker logs <id>` | Show a container's output (great for debugging) |
| `docker exec -it <id> sh` | Open a shell *inside* a running container |
| `docker stop <id>` / `docker rm <id>` | Stop / delete a container |
| `docker compose up` / `down` | Start / stop a whole multi-service stack |
| `docker compose logs -f <service>` | Follow the logs of one service in the stack |

**Golden rules for this lab:**
1. Always build from the **project root**, always end with `.`
2. Make sure **Docker Desktop** is running first.
3. **Image** = frozen box · **Container** = running box · **Dockerfile** = recipe.
