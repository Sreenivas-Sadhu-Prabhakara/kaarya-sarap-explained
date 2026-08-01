# KĀRYA SARAP — explained

How a food-imagery tool for Philippine F&B MSMEs got built, and what image models
get wrong about Filipino food.

This is the companion to a private build. The code and the dish dictionary are not
published — the dictionary is the part that took the work. The **method** is here:
the decisions, the arithmetic behind them, and the things that turned out to be
wrong.

A [KĀRYA](https://kaarya-bench.pages.dev/) solution. *A working studio thinks out
loud.*

---

## The one-paragraph version

A restaurant owner picks a dish by name and gets marketing photographs sized for
Facebook, Instagram, TikTok, GrabFood and Foodpanda. Generation runs on a Mac
Studio behind a Cloudflare tunnel, so the marginal cost is electricity. The user
never writes a prompt — the prompt is the product's hidden asset, not its
interface.

## Contents

| | |
|---|---|
| [01 — Why not the obvious architecture](docs/01-why-not-cloud-run.md) | Serverless GPU costs, and why "pay per use" isn't "pay per image" |
| [02 — Why not Fooocus](docs/02-why-not-fooocus.md) | The tool this project was supposed to be built on, and why it wasn't |
| [03 — What models get wrong about Filipino food](docs/03-filipino-food-benchmark.md) | The benchmark, and the results |
| [04 — Designing for MSMEs, not for developers](docs/04-designing-for-msmes.md) | Why the prompt box had to go |
| [05 — Platform rules as design constraints](docs/05-platform-rules.md) | How Foodpanda's upload policy deleted two features |
| [06 — The architecture](docs/06-architecture.md) | Async jobs, the engine seam, and what R2 is actually for |

---

## Three findings worth the click

**Serverless GPU inverts the usual economics.** Normally serverless wins at low
volume. With a GPU it loses, because at low volume your bill is dominated by
loading model weights into VRAM rather than by generating anything. Cloud Run
GPU works out at roughly **$1.04/hour** fully loaded — and a scale-to-zero
configuration spends most of that on cold starts.
[Details and the arithmetic →](docs/01-why-not-cloud-run.md)

**The tool the project was named after was the wrong tool.** Fooocus is excellent,
and its entire value proposition is a polished UI hiding SDXL's complexity. This
project builds its own UI — so it would have paid Fooocus's costs (no REST API, a
bug-fix-only maintenance mode, permanently frozen on SDXL) for a benefit it was
throwing away. [Details →](docs/02-why-not-fooocus.md)

**Platform rules shaped the product more than taste did.** Foodpanda prohibits
watermarks and branding on menu photos, which deleted the price-overlay feature
for delivery exports. It rejects close-ups, which deleted a preset. And its
4000×2925 minimum forced an entire upscale stage into the pipeline that nothing
else needed. Reading merchant documentation before designing presets saved
building two features that would have been rejected on upload.
[Details →](docs/05-platform-rules.md)

---

## Status

Built through Phase 4. Phase 0 (the model benchmark) is the interesting part and
has its own writeup. Numbers in these documents were measured or verified against
a primary source on the date given, not recalled.

## Licence

Documentation under [CC BY 4.0](LICENSE). Code samples, where they appear, are
illustrative fragments rather than the running system.
