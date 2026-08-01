# The architecture

Four decisions carry most of the weight. Each solves a specific problem, and each
was arrived at by hitting the problem rather than by taste.

```
Phone / browser
   │  Google login ──► Cloudflare Access (email allowlist)
   ▼
Cloudflare Pages ─────────────► R2  (finished images, edge-served)
   │                               ▲
   │  /api/* through tunnel        │ upload
   ▼                               │
Cloudflare Tunnel ──► Mac Studio ──┘
                          │
                    Gateway (FastAPI)
                      • verify Access JWT
                      • profile ─► presets + export targets
                      • dish + cuisine ─► ComfyUI workflow
                      • SQLite job queue
                      • upscale ─► crop ─► overlay
                          │
                    ComfyUI (127.0.0.1 only) ──► MPS
```

## 1. The API is async because Cloudflare says so

Generation takes 60–150 seconds on Apple Silicon; the Foodpanda upscale adds more.
Cloudflare's free and pro plans terminate proxied origin responses at **100
seconds** with error 524.

A synchronous `POST /generate` that waits for the image would time out — not
occasionally, but as the normal case.

So: `POST /api/jobs` returns `202 {job_id}` immediately, and the client polls.
Which is the right shape for a slow job regardless of any timeout; the platform
constraint just removed the option of getting it wrong.

The state machine is explicit, and **every terminal state carries a
human-readable reason**:

```
queued ─► running ─► upscaling ─► exporting ─► complete
   └─────────┴───────────┴────────────┴──► failed | timeout | engine_unavailable
```

A job never disappears silently. If the Mac was rebooting, the user is told the
Mac was rebooting.

## 2. A gateway in front of the engine

ComfyUI is never exposed. Not behind auth, not on a private hostname — the tunnel
routes the gateway and nothing else.

The reason is specific: **ComfyUI ships with no authentication, and its node graph
can read and write arbitrary files.** Anyone who reaches port 8188 has the
machine. On a box that also holds the dish dictionary and Cloudflare credentials,
that's the whole game.

The gateway owns everything ComfyUI shouldn't: JWT verification, tenant scoping,
profile policy, the blocklist, the queue. It also means **all rejection happens
before GPU time is spent** — a request for a close-up composition from a
cloud-kitchen tenant returns 400 in milliseconds rather than after two minutes of
generating something Foodpanda will refuse.

## 3. R2, not the tunnel, serves the images

The obvious design serves finished images back through the tunnel from the Mac's
disk. It works, and it has a failure mode: when the Mac reboots, the entire
gallery goes dark. Every image the merchant has ever generated becomes
unavailable because a machine in someone's office is restarting.

Uploading to R2 decouples them. The Mac becomes necessary only for *making new*
images. Past work is served from Cloudflare's edge — faster, closer, and available
when the origin isn't. R2 charges no egress fees, so this costs approximately
nothing.

The user-visible version: when the Mac is down, the site loads, the gallery works,
and the generate button is disabled behind an honest banner. Degradation is
visible and partial rather than total and mysterious.

## 4. The engine seam

```python
class ImageEngine(Protocol):
    async def submit(self, spec: GenerationSpec) -> EngineJobId: ...
    async def poll(self, job: EngineJobId) -> EngineProgress: ...
    async def fetch(self, job: EngineJobId) -> bytes: ...
    async def health(self) -> EngineHealth: ...
```

Four methods. An engine receives a fully resolved spec — every prompt already
merged, every policy already applied — and returns bytes. It reads no catalog data
and makes no decisions.

That narrowness buys two things:

**Portability.** Moving generation from the Mac to a cloud GPU is a config change
rather than a rewrite. This is the difference between "enterprise ready" as a
claim and as a property.

**Testability.** `FakeEngine` implements the same protocol and returns generated
fixtures. The entire test suite — API, worker, export pipeline, job lifecycle —
runs on a Linux CI runner with **no GPU and no network**. If a test needs ComfyUI
running, it's in the wrong place.

`health()` has one unusual rule: it must never raise. Callers use it to render a
status banner, so an exception there converts a degraded state into a broken page.

## Data as data

Three directories — `cuisines/`, `profiles/`, `dishes/` — are pure JSON and depend
on nothing. Nothing depends on their internals either.

The test of whether this boundary is real: **adding a sixth cuisine should be a
data task.** If it requires a Python change, the boundary has been violated. That's
a falsifiable check rather than an aspiration, and it's written into the spec so
it can be held to later.

It also means the dish dictionary — the part that took the actual work — is a pile
of JSON that a person who knows food can extend without knowing Python.

## Two axes that compose

Business type (fine-dining, in-dining, cloud-kitchen) and cuisine (five of them)
are orthogonal. Cuisine supplies vocabulary — vessels, surfaces, garnishes, and the
negative anchors that stop Korean food rendering as Chinese. Business type supplies
presets and export targets. They merge at workflow-build time.

Three plus five is eight configurations to maintain. Enumerating the combinations
would have been fifteen, each drifting from the others.

## What isn't solved

The Mac is a single point of failure for *generating* new images. R2 covers the
gallery, but if the machine is off, nothing new gets made. For a pilot that's
acceptable and honestly surfaced. For a real product it means either a second
engine behind the same protocol, or accepting that the service has office hours.

The engine seam makes that a decision rather than a rewrite — which is the most
you can ask of an architecture before you know which way it'll go.
