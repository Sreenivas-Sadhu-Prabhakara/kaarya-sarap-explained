# Why not Cloud Run

*Verified against the Google Cloud Billing API, `us-central1`, 2026-08-01.*

The project started with a clear plan: run [Fooocus](https://github.com/lllyasviel/Fooocus)
on Google Cloud Run, scale to zero, pay only for what gets used. Cloud Run
supports GPUs now. It seemed obvious.

The question that unravelled it was simple: *if I only generate thirty images a
day, why isn't this almost free?*

## What an L4 actually costs

Cloud Run's GPU configuration has a floor — one GPU requires at least 4 vCPU and
16 GiB of memory. Pulling the real SKU prices rather than trusting a blog post:

| Component | Rate |
|---|---|
| NVIDIA L4, no zonal redundancy | $0.0001867 /s |
| CPU, 4 vCPU minimum | $0.000072 /s |
| Memory, 16 GiB minimum | $0.0000309 /s |
| **Fully loaded** | **$0.00029 /s ≈ $1.04/hour** |

That number alone isn't damning. What's damning is *what you spend it on*.

## Three reasons "pay per use" isn't "pay per image"

**GPU forces CPU-always-allocated.** You cannot use the request-throttled billing
mode that makes CPU-only Cloud Run nearly free. Billing follows **instance
lifetime**, not request duration.

**Cold starts bill at GPU rates.** Loading a 7 GB SDXL checkpoint into VRAM is
billed L4 time during which no image exists. Three minutes of cold start is about
$0.05 — the floor for a session, and the user watches a spinner for all of it.

**The idle tail.** An instance isn't killed the moment a response returns; it's
kept warm in case another request arrives, and you don't precisely control that
window. A "ten-second image" can bill several minutes of wall clock.

There's also a trap worth naming, because it's the natural reaction to the cold
start problem: setting `min-instances=1` costs $1.04 × 730 hours ≈ **$760/month**.

## Running the numbers

Two twenty-minute sessions a day, fifteen images each:

```
2 × 1200s × $0.00029 = $0.70/day ≈ $21/month
```

Then compare Vertex AI Imagen at roughly $0.02–0.04 per image: 900 images a month
lands at **$18–36**.

They cost the same. But Imagen has no cold start, no ops, no model weights, and no
fight between Gradio's session state and an autoscaler. Self-hosting only wins
economically at thousands of images a month — and at that volume a persistent GPU
VM beats Cloud Run anyway.

**Cloud Run GPU is built for steady traffic. This workload is sparse.** That's the
whole finding.

## Where "almost free" actually lived

An M2 Max with 64 GB of unified memory and 30 GPU cores, already configured as an
always-on server:

```
$ pmset -g custom
  sleep                0     # never sleeps
  womp                 1     # wake on network
  autorestart          1     # recovers from power loss
```

Slower than an L4 — Fooocus's own documentation warns Apple Silicon runs about
**9× slower than an RTX 3000-series card**, putting a single image somewhere
around 60–150 seconds. But the marginal cost is electricity.

Final shape:

- **Cloudflare Pages** — the UI. Static, free.
- **Cloudflare Access** — Google login, free for a small allowlist.
- **Cloudflare Tunnel** — outbound only; no ports opened, no public IP.
- **Mac Studio** — the GPU work.
- **R2** — finished images, served from the edge, no egress fees.

Total: about **$0/month**.

## The part that's easy to miss

Cost and latency are the same variable here, not two. Every second you spend
making it cheaper — scaling to zero, avoiding warm instances — is a second the
user spends waiting. Optimising one degrades the other.

The Mac Studio sidesteps the trade rather than solving it: the model is already
resident in unified memory, so there's no cold start to amortise and nothing to
scale down. It's slower per image and free per image, which for a low-volume
internal-ish tool is the correct corner of the design space.

## What this doesn't prove

The Mac path depends on a machine that's already always-on and already paid for.
Buying a Mac Studio to serve this would be a different calculation entirely.

It also isn't a general verdict on Cloud Run GPU. For steady traffic — where
instances stay warm and cold starts amortise across many requests — the economics
invert again and it looks good. The finding is narrower than "don't use serverless
GPU": **it's that sparse traffic and GPU cold starts combine badly**, and that's
worth checking before you build.
