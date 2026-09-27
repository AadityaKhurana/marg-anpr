# MARG — 5-Minute Pitch & Demo Script

**MARG — Multi-camera ANPR & Route Graph** · SIH 2026 (PS SIH26127) · Dwarka, New Delhi prototype

Target: **~5:00** spoken at ~150 wpm (~760 words). Time cues are guides, not hard cuts.
`[DO]` lines are stage directions — do them silently while you talk.
The demo is ordered to hit the **most awe-inducing feature first** (city-wide trajectory), then a
**live alert firing in real time**, then breadth (anomalies + analytics + reports).

---

## ⏱ 20-second setup (before you present — screen already on the Live map)

```bash
docker compose up -d --build
docker compose stop producer persistence alerts analytics
docker compose exec -T postgres psql -U anpr -d anpr < db/seed_dwarka.sql
docker compose exec -T postgres psql -U anpr -d anpr < db/seed_metrics.sql
docker compose start producer persistence alerts analytics
```

Open **http://localhost:5173** on the **Live map**. Confirm the green **LIVE** badge and that the
**Live sightings** panel is streaming plates. Have `DL3CAB1234` and `DL8CAF5678` ready to type.

---

## [0:00 – 0:35]  Intro + the problem

**[Intro — ~12s]** "Good morning — we're **Team Coding_Uncles**. This is **MARG**, a city-wide ANPR
platform that turns a city's existing cameras into one system for **tracking vehicles, live
enforcement, and traffic analytics**. Here's the problem it solves."

`[If the panel already has your name/PS on the slide, drop the 'we're Team Coding_Uncles' half and open on 'This is MARG…'.]`

**[The problem]** "Every big city already runs thousands of CCTV and ANPR cameras — but each feed
lives in its **own silo**. A camera reads a plate, logs it, and forgets it. Nothing links what one
camera saw to what the camera two kilometres away saw a minute later.

So today, if the police need to **follow one suspect vehicle across the city**, someone scrubs
footage camera by camera, by hand, for hours. And all the movement data the city already collects is
just **thrown away**."

`[DO]` Live map is on screen — let the junctions and the streaming sightings panel speak for themselves.

## [0:35 – 1:10]  Our solution

"**MARG** sits on top of that existing camera network and turns it into **one intelligent system**.
Four things, all live:

- It reads plates with a **YOLO + PaddleOCR** engine built for Indian plates in bad light and motion blur.
- It **reconstructs any vehicle's full route** across the city, on a map, in seconds.
- It runs **real-time enforcement** — blacklisted vehicles and impossible-route anomalies flagged the instant they happen.
- And it turns the same feed into **city-scale traffic analytics** — live congestion, hotspots, and origin–destination flows.

No new hardware. It's software that makes the cameras a city already owns finally talk to each other."

`[DO]` Sweep the cursor down the left nav — Live map, Plate trajectory, Alerts, Traffic analytics, Reports.

## [1:10 – 1:45]  Architecture (brief)

"The architecture is deliberately simple. Every camera — real OCR or our demo feed — reports the
**same event: a `PlateSighting`** — a plate, a location, and a timestamp. That one event is the only
contract in the whole system. Those events stream through **Redis** to a handful of workers that each
own one job: one **stores every sighting in PostgreSQL with PostGIS** and rolls the per-approach
cameras up to junctions, one runs the **blacklist and route-anomaly checks**, and one **aggregates
traffic into five-minute windows**. **FastAPI** serves it, and the **React + Leaflet** dashboard
updates live over a **WebSocket** — no polling.

Because every camera meets the system at that one event, the **real OCR model and our synthetic feed
are interchangeable**, and we scale out just by adding more workers."

`[DO]` One glance at `assets/architecture.png` (or the diagram tab), tracing top-to-bottom, then back to the app.

## [1:45 – 4:30]  Live demo  ← the core; keep moving

### ① The "wow": track one vehicle across the whole city  *(1:45 – 2:35)*
"Let's say this plate is our vehicle of interest. I just type it in —"

`[DO]` **Plate trajectory** → type **`DL3CAB1234`** → select it.

"— and MARG instantly reconstructs its **entire trajectory across Dwarka**: every junction it passed,
in order, with **timestamps, direction of travel, and the road it took between each junction**. What
used to be hours of manual footage-scrubbing is now **one search**."

`[DO]` Point along the plotted path on the map, then the hop-by-hop timeline beside it.

### ② Live enforcement: a blacklist hit firing in real time  *(2:35 – 3:20)*
"Meanwhile enforcement never stops. Watch the live feed — this vehicle is on the **blacklist**, and
the moment a camera sees it, the alert fires **on its own**, in real time."

`[DO]` **Alerts** → open the **blacklist** alert for **`DL8CAF5678`** → show plate, camera/junction, time → **Acknowledge** it (show the audit action).

"That alert wasn't something I refreshed for — it was **pushed live** the instant a camera saw the car."

### ③ Intelligence, not just matching: route anomalies  *(3:20 – 3:45)*
"It also catches vehicles that shouldn't be possible. If the **same plate** appears at two junctions
**faster than any car could legally travel between them**, or moving **against** a one-way
carriageway, MARG flags a **route anomaly** — a classic signature of a **cloned or spoofed plate**."

`[DO]` In Alerts, filter to **Route anomalies** → open one → show the impossible-travel / wrong-direction reason.

### ④ City-scale analytics from the same feed  *(3:45 – 4:30)*
"And the exact same data, **aggregated**, becomes **city-scale analytics**."

`[DO]` **Live map** → toggle **Congestion** view, then **Node load** view (watch junctions resize/recolour by volume).
`[DO]` **Traffic analytics** → show, quickly: the **node heat / load**, the **worst corridor** congestion, **origin–destination** (journeys by first & last camera), and the **daily volume + congestion trend**.
`[DO]` **Reports** → flash the **weekly report** (vehicles recorded, mean corridor speed, total delay).

"Density hotspots, the worst corridor right now, where trips start and end, and the weekly trend —
all from cameras the city **already** has."

## [4:30 – 5:00]  Close

"So that's **MARG**: one platform that turns a city's existing cameras into **trajectory tracking,
live enforcement, and traffic analytics** — all off a single, standard event.

It's production-shaped — **containerized and config-driven**, ready to swap in managed cloud services
and scale to a real city. Today it runs on a **genuine Dwarka road network** with a synthetic feed;
drop the live OCR model onto the same event and **nothing downstream changes**. Thank you — happy to
take questions."

---

## ✅ Features covered (for your own check)
Live GIS map · streaming live sightings (WebSocket) · **plate trajectory reconstruction** ·
**live blacklist alert** · alert acknowledge/audit · **route-anomaly detection** · map Congestion &
Node-load views · analytics heatmap / worst-corridor / **origin–destination** / trend · **weekly
report**. (Blacklist management page is available if a judge asks how entries are added.)

## 🎯 Awe-priority order (if you run short on time, keep the top of this list)
1. Trajectory reconstruction (①) — the headline.
2. Live blacklist alert firing on its own (②).
3. Route anomaly = cloned-plate catch (③).
4. Analytics breadth (④) — cut to just the heatmap + one trend if time is tight.

## 🛟 Fallback (if the live feed stalls mid-demo)
The seed data renders the full network and completed journeys, so **trajectory, alerts, and analytics
all still demo from seeded data** — just say "here's data the pipeline already captured" and carry on.
Don't wait on a live frame.

## 🗣 Honesty notes (for Q&A, not the script)
- **>90% OCR accuracy** is the engine's design target; real YOLO + PaddleOCR needs a GPU, model
  weights, and video, so today's demo drives the identical `PlateSighting` contract with a synthetic
  producer. That's a feature of the architecture, not a shortcut — the integration boundary is real.
- The Dwarka network (junctions, links) is **genuine** — real OSM junctions, OSRM-routed roads.
