# Deploying MARG on Azure for Students (launch → live)

A step-by-step runbook to host the full MARG stack (API, workers, producer,
Postgres+PostGIS, Redis, MinIO, frontend) on a single Azure VM using the free
Azure for Students credit. No credit card needed — just student verification.

It uses the production compose file (`docker-compose.prod.yml`): the frontend is
a real Vite build served by nginx on **port 80**, the data services are
internal-only, and every service has log rotation so the disk can't fill.

---

## 0. What you need first

- A student email or enrollment proof (for Azure verification).
- The repo pushed to GitHub (it is: `AadityaKhurana/sih-internal-round`).
- ~30 minutes.

---

## 1. Get the free credit

1. Go to <https://azure.microsoft.com/free/students/>.
2. Sign in with a Microsoft account and verify you're a student (student email
   or an academic document). **No card is requested.**
3. You get **$100 credit for 12 months** + some always-free services.

---

## 2. Create the VM

In the Azure Portal → **Create a resource → Virtual machine**:

| Setting | Value |
|---|---|
| Image | **Ubuntu Server 24.04 LTS** |
| Size | **B2als_v2** (2 vCPU, 4 GB, AMD, x64) |
| Region | **Central India** → fall back to **South India** → **Southeast Asia** |
| Authentication | **SSH public key** (generate one if you don't have it) |
| Public inbound ports | **SSH (22)** only for now |
| Disk | 30 GB standard SSD |

**Size + budget (running 24/7 to end of December, ~95 days):**
- **B2als_v2 (4 GB)** ≈ ~$27/mo → ~$85 compute + ~$8 disk ≈ **~$93 on $100**. Fits,
  with a thin buffer — set the budget alert in step 9.
- Do **not** pick B2as_v2 (8 GB, ~$50/mo) — always-on it would overrun the $100.
- If B-series sizes show "Unavailable" in a region, switch region (Southeast
  Asia / Singapore has the best India latency after India itself) and retry.
- Use a **dynamic** public IP (not static): since the VM never stops, the IP
  stays put while running, saving the ~$3/mo static-IP charge.

Click **Create**, keep the SSH key, and note the VM's **public IP**.

---

## 3. Open the ports you actually need

VM → **Networking → Add inbound port rule**. Keep this tight:

- **22** (SSH) — already open.
- **80** — the dashboard (nginx serves the app here).
- **443** — only if you add HTTPS via Caddy (step 8b).

The prod compose does **not** publish Postgres (5432), Redis (6379), or MinIO
(9000/9001) — they're reachable only inside the Docker network, so there's
nothing unsafe to expose. Just 22 + 80 (+ 443 for HTTPS).

---

## 4. Connect and prep the box

```bash
ssh -i /path/to/key.pem azureuser@<PUBLIC_IP>

# Docker + compose plugin (official script)
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
newgrp docker   # or log out/in so the group applies

# 2 GB swap — safety net for the frontend build (npm ci + vite build) on 4 GB
sudo fallocate -l 2G /swapfile && sudo chmod 600 /swapfile
sudo mkswap /swapfile && sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

---

## 5. Get the code

If the repo is **public**:

```bash
git clone https://github.com/AadityaKhurana/sih-internal-round.git
cd sih-internal-round
```

If it's **private**, either run `gh auth login` (GitHub CLI) first, or create a
fine-grained Personal Access Token and clone with it:

```bash
git clone https://<TOKEN>@github.com/AadityaKhurana/sih-internal-round.git
```

---

## 6. Set your secrets

The compose defaults (`anpr/anpr`, `minioadmin/minioadmin`) are fine locally but
must not go to a public box. Create `.env` in the repo root:

```bash
cat > .env <<'EOF'
POSTGRES_USER=anpr
POSTGRES_PASSWORD=<pick-a-strong-password>
POSTGRES_DB=anpr
DATABASE_URL=postgresql://anpr:<same-password>@postgres:5432/anpr
REDIS_URL=redis://redis:6379/0
S3_ENDPOINT=http://minio:9000
S3_ACCESS_KEY=<pick-a-key>
S3_SECRET_KEY=<pick-a-secret>
S3_BUCKET=anpr-media
EOF
```

> Match the exact variable names your backend/workers read — check the existing
> `.env.example` in the repo and copy any keys it lists that aren't above.

---

## 7. Launch the stack (production compose)

```bash
docker compose -f docker-compose.prod.yml up -d --build
docker compose -f docker-compose.prod.yml ps    # all services running/healthy
```

The first build compiles the frontend bundle (a few minutes on 4 GB — the swap
from step 4 covers the peak). The database schema auto-applies on first startup.

---

## 8. Load the demo data (Dwarka network)

Without this the map is empty. Apply the committed seed **with the workers
stopped** — the seed truncates tables and needs an exclusive lock the running
workers would otherwise hold:

```bash
docker compose -f docker-compose.prod.yml stop producer persistence alerts analytics

docker compose -f docker-compose.prod.yml exec -T postgres psql -U anpr -d anpr < db/seed_dwarka.sql
docker compose -f docker-compose.prod.yml exec -T postgres psql -U anpr -d anpr < db/seed_metrics.sql

docker compose -f docker-compose.prod.yml start producer persistence alerts analytics
```

(Use the committed `db/seed_dwarka.sql` as-is — no need to regenerate it; that
requires network access to OSRM/Overpass.)

---

## 8a. Access it (HTTP)

Open `http://<PUBLIC_IP>/` — the dashboard loads with the Dwarka map. nginx
serves the built app on port 80 and proxies `/api` and `/ws/live` to the backend
internally, so this is all you need for a working shared link.

## 8b. Optional — HTTPS via Caddy + a free domain

For `https://` (nicer for a printed/shared link), use the free `.me` domain from
the GitHub Student Pack (point its A record at the VM's public IP). Caddy needs
ports 80/443, so first move the frontend off host-80 — edit the `frontend`
service `ports:` in `docker-compose.prod.yml` to bind locally only:

```yaml
    ports:
      - "127.0.0.1:8080:80"
```

Re-up the frontend (`docker compose -f docker-compose.prod.yml up -d frontend`),
open **443** in the NSG, then run Caddy on the host:

```bash
# /etc/caddy/Caddyfile
your-domain.me {
    reverse_proxy localhost:8080
}
```

`sudo apt install caddy` and drop the file in `/etc/caddy/Caddyfile`. Caddy
fetches a Let's Encrypt certificate automatically; `https://your-domain.me` now
serves the dashboard and only 80/443 are public.

---

## 9. Make the $100 last to end of December

Since evaluation timing is unknown, the VM runs **24/7** — so the levers are
size (already chosen: B2als_v2) and watching the spend, not stop/start:

- **Budget alert (do this now):** Cost Management → Budgets → alert at **$85**.
  That leaves ~2 weeks of runway to react if it's trending over before Dec 31.
- **Log rotation is built into the prod compose** (3 × 10 MB per service), so
  30 GB won't fill from logs over the run.
- **Data persists** on the Docker volumes across reboots, so the seeded Dwarka
  network survives a restart.
- **If the alert ever fires with no evaluation yet:** fail over to a home machine
  + a free Tailscale Funnel for the tail of the window (no credit clock).
- **Optional credit saver:** if plans change and you *can* predict idle stretches,
  Portal → VM → **Stop (deallocate)** bills only ~$1.50/mo for the disk. Set a
  static IP first if you'll rely on the address staying constant.

---

## 10. Quick troubleshooting

All commands take `-f docker-compose.prod.yml`.

| Symptom | Check |
|---|---|
| Dashboard won't load | `docker compose -f docker-compose.prod.yml ps` — is `frontend` running? `... logs frontend` |
| Map is empty | Did the seed apply? Re-run step 8 (workers stopped) |
| API 502 / no data | `... logs api`; confirm `.env` `DATABASE_URL` matches your password |
| Frontend build OOM | Confirm the 2 GB swap from step 4 is on (`free -h`) |
| Can't reach it from browser | NSG inbound rule for **80** (or 443 with Caddy) added? |

---

### Summary

Azure for Students → **B2als_v2 (4 GB)** Ubuntu VM → Docker →
`docker compose -f docker-compose.prod.yml up -d --build` → seed → open port 80.
Production build behind nginx, data services internal-only, log rotation on.
Runs 24/7 to end of December inside the $100 — set the $85 budget alert.
