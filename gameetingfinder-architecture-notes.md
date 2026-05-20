# GAMeetingFinder — current architecture notes

Internal notes. Not client language.

## Bottom-line read

This is a plain Python/FastAPI + Jinja + Postgres app with a lot of product,
admin, analytics, and AI behavior packed into a few large files.

That is not automatically bad.

For resilience work, the important read is:

- the web tier is **mostly stateless enough** to move around
- the system is **not yet operationally hardened**
- the main problems are **durability, recovery, and blast radius**, not
  framework choice

We should be careful not to overprescribe dual-host failover.

The repo does **not** currently make the case that they need active failover.
It makes the case that they need a better recovery story.

---

## Current architecture

### App shape

- Python FastAPI app served with Uvicorn
- server-rendered HTML via Jinja templates
- Postgres as source of truth
- plain JS/CSS, no frontend build chain
- Dockerized deploy
- Railway-hosted today
- Cloudflare in front for CDN/security/analytics

### Repo shape

Main app logic is concentrated in a few large files:

- `web/main.py` — public site, auth flows, settings, admin surfaces, misc ops
- `web/auth.py` — JWT auth and user data helpers
- `web/ai_helpers.py` — AI helpers, summaries, agent-like features
- `web/static/app.js` — main client-side behavior
- `web/static/style.css` — main styling

### Operational shape

- public site
- `/dailypaper/*` admin/ops/reporting surface
- cron-driven analytics / summaries / internal automations
- direct DB access from app and scripts

### What is working in their favor

1. **Cookie/JWT auth instead of server-memory sessions**
   - This helps. Requests do not obviously need sticky sessions.

2. **Postgres-backed state**
   - Users, meetings, submissions, newsletter, admin data are in DB.
   - Good shape for recovery and redeploy.

3. **Simple runtime**
   - One main app process, one DB, static assets, a few scripts.
   - Simpler than a split SPA/API stack.

4. **Cache-friendly public behavior**
   - Static assets are cacheable.
   - Some public data is already cached in-process.
   - This helps CDN-style resilience.

---

## What this architecture is good at

- shipping fast without much tooling overhead
- simple mental model for request -> DB -> template response
- easy cold-start into a secondary host if env + DB access exist
- public pages that can benefit from edge caching

---

## What this architecture is bad at

- separating public uptime concerns from internal/admin load
- making infra changes surgically, because concerns are densely packed
- durable file handling across hosts
- clearly bounded blast radius between app, admin, analytics, and AI surfaces

---

## What we think they probably do **not** need yet

Probably **not** needed yet:

- active-active multi-region failover
- complex load balancing strategy
- frontend rewrite
- framework migration
- dual-host architecture as the first move

Why:

- outages at the host level are probably rarer than recovery mistakes,
  deployment mistakes, or data durability gaps
- more moving parts would likely outpace their actual risk
- the repo suggests the more likely failure modes are operational, not
  architectural purity failures

---

## What they likely need before failover is even worth discussing

### 1. Better backup story

Current read:

- there is backup logic, but it reads more like a script than a hardened backup
  system
- off-platform storage is still an explicit future concern

What matters:

- off-provider backups
- retention clarity
- tested restore path
- confidence that a host/vendor issue does not become a data-loss event

### 2. Better recovery story

This is the real gap.

Questions we should care about:

- If Railway went down for a few hours, how quickly could they stand this back
  up elsewhere?
- If the DB was corrupted, what is the restore process?
- If env vars disappeared, how hard is rebuild?
- If a deployment broke prod, how easy is rollback?

### 3. Better durability story for runtime files

Current read:

- some files are still local-disk assumptions
- that is the most obvious thing that breaks once multiple hosts enter the
  picture

This matters whether or not they ever do failover.

### 4. Better public/admin separation

The repo itself documents origin stress and 504 pressure from admin polling /
query behavior.

That means:

- some reliability issues are likely internal load-shape issues
- a second host would not automatically solve them
- public uptime should be protected from internal tools where possible

### 5. Better health visibility

If the app has no clean health-check surface, every future uptime improvement
gets fuzzier.

---

## Reliability risk stack, as we currently see it

Ordered roughly from most likely / most useful to address first.

### Tier 1 — recovery and durability risks

- backups not sufficiently off-platform or tested
- local-disk runtime files
- unclear redeploy / restore procedure
- vendor outage with no quick recovery path

### Tier 2 — origin stability risks

- admin/analytics load causing origin pressure
- DB query pileups or polling bursts
- app/process health ambiguity

### Tier 3 — architecture density risks

- giant files make operational changes harder to reason about
- public/admin/AI concerns live too close together

### Tier 4 — true high-availability risks

- single-host dependency
- no warm standby
- no traffic cutover mechanism

Important: Tier 4 is real, but it is probably **not** the first thing to sell.

---

## Opinionated recommendation

Reframe away from “minimum failover plan.”

Lead with:

1. **Current state**
2. **Resilience and recovery**
3. **Backup and downtime risk**
4. **Practical uptime plan**
5. **Business continuity plan**

That is a better shape because it matches what they are more likely to need and
more likely to understand.

It also avoids quietly pushing them into a dual-instance architecture they may
not need.

---

## Proposed client framing

### Current state

- simple stack
- mostly stateless web tier
- database-backed business state
- some local-disk assumptions
- admin and public traffic share too much runtime surface

### Resilience and recovery

- can the app be rebuilt elsewhere quickly?
- can the DB be restored cleanly?
- are critical files durable outside the host?
- is there a clear health signal?

### Backup and downtime risk

- how much data could be lost?
- how long could they be down?
- what depends on one provider today?

### Practical uptime plan

- reduce origin overload
- improve health visibility
- strengthen caches where helpful
- make rebuild/redeploy/recovery boring

### Business continuity plan

- define acceptable downtime
- define restore owner/process
- define vendor outage response
- define data durability expectations

---

## Short version we should remember

The right move is probably:

**make recovery boring before making failover clever**

That feels truer to the repo, truer to the likely business need, and less prone
to overengineering.
