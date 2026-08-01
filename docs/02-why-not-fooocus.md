# Why not Fooocus

*Verified against the Fooocus repository, 2026-08-01.*

This project was going to be built on [Fooocus](https://github.com/lllyasviel/Fooocus).
It's a genuinely excellent piece of software. It isn't the right tool here, and
the reason is worth writing down because it generalises.

## What the repo actually says

Four facts, read from the source rather than remembered:

| Fact | Consequence |
|---|---|
| **Limited LTS — bug fixes only**, no migration to newer model architectures | Permanently frozen on SDXL. No FLUX, no successor models, ever. |
| **Gradio UI only — no REST API** | Driving it from your own frontend requires an unmaintained third-party fork. |
| Requires Python 3.10 | A separate environment. Annoying, not disqualifying. |
| Apple Silicon runs **~9× slower than an RTX 3000-series card** | 60–150s per image. Real, and it shaped the whole async design. |

The `--share` flag and `auth.json` exist, so you *can* expose Fooocus directly with
basic auth. But that means shipping Fooocus's interface — every SDXL knob, in
English, on a desktop-oriented layout — to a restaurant owner on a phone.

## The argument

Fooocus's value proposition is **hiding SDXL's complexity behind a polished,
well-tuned UI**. Its defaults are genuinely good; its prompt expansion is
thoughtful; you can hand it to someone who has never heard of a sampler and they
get a nice picture.

This project builds its own UI, in Taglish, around a dish picker rather than a
prompt box.

So the deal on the table was: **pay Fooocus's costs — no API, frozen models, a
dependency on a third-party fork — in exchange for a benefit being thrown away.**

Put that way it isn't a close call.

## What replaced it

[ComfyUI](https://docs.comfy.org/development/comfyui-server/comms_overview), which
is headless-first in a way that matters:

> HTTP routes for submitting workflows, uploading files and querying status;
> server-to-client updates over WebSocket.

HTTP in, WebSocket out. That asymmetry is exactly the shape of a slow job — submit
over a short request, stream progress over a long-lived socket — rather than
something bolted on afterwards. There's a real queue. It runs SDXL, FLUX and
newer architectures, so the base model becomes a swappable decision instead of a
permanent one.

Same Mac, same tunnel, same $0.

## The generalisable bit

**When you replace a tool's primary interface, re-examine whether you still want
the tool.**

Fooocus is a UI with a diffusion engine attached. Used as a headless engine, you
inherit all the constraints that pay for a UI you deleted — and the constraints
are permanent while the UI was optional.

The tell was in the maintenance status. "Bug fixes only, no new architectures" is
fine for a finished product someone uses as-is. For a foundation you intend to
build on for years, it's an expiry date. Worth reading before writing code, not
after.

## What Fooocus is still better at

If you want a good image from a text prompt on your own machine today, with no
integration work and no workflow graph to assemble, Fooocus is the faster path and
its defaults beat what most people configure by hand. Its prompt expansion is
better than what this project's dictionary achieves for anything outside the
curated dish list.

The choice here says something about *this* project — building a custom interface
over a long horizon — not about the tool.
