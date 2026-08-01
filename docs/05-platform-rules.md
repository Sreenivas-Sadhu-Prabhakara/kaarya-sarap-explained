# Platform rules as design constraints

*Sourced from published merchant documentation, 2026-08-01.*

The plan was to design the presets, build the exporter, then look up what
dimensions the delivery apps wanted. Doing it in the other order changed the
product.

## What the documentation says

| Platform | Requirement |
|---|---|
| **GrabFood** | 800 × 800, JPEG or PNG, max 6 MB — [Sapaad KB, GrabFood Connect](https://kb.sapaad.com/help/what-are-the-recommended-specs-for-menu-item-images-in-grabfood-connect) |
| **Foodpanda** | Minimum 4000 × 2925 at 72 dpi, under 20 MB, max 16 MP, **top or front view, no watermarks, no branding, no close-ups, no people** — [menu requirements](https://www.scribd.com/document/495668199/foodpanda-s-menu-requirements) |

Three consequences, none of which were in the design an hour earlier.

## 1. Foodpanda's minimum forces an upscale stage

SDXL generates around 1024×1024 *total pixels*. At Foodpanda's ~1.37:1 aspect
ratio that's roughly 1216×888.

Getting to 4000×2925 needs a **~3.3× upscale pass** — an ESRGAN-class model as a
ComfyUI node, adding seconds per image on Apple Silicon.

That's a whole pipeline stage, a model download, a job state, and a measurable
latency cost that existed nowhere in the design until someone read a PDF. The
export module went from *"crop with Pillow"* to *"depends on the generation
engine"*, which is a considerably larger thing.

## 2. "No watermarks or branding" deletes a feature

Price overlay — `₱149 only` composited onto the image — was a headline feature.
Foodpanda prohibits it on menu photos. The platform renders price and dish name
itself, in its own UI, and merchant-applied text on the image reads as branding.

So overlay is now **unconditionally suppressed** on all delivery-app export
targets, regardless of what the business profile permits. It's enforced in the
export target definition rather than left to the caller, because the failure mode
otherwise is a merchant uploading forty images and having them rejected one at a
time.

For cloud kitchens — whose *only* surfaces are GrabFood and Foodpanda — that means
the feature does not exist at all.

## 3. "No close-ups" deletes a preset

`hero-closeup` — shallow depth of field, visible steam, tight on the texture — is
the most immediately appetising preset in the set.

Foodpanda rejects close-ups. Delivery-only merchants can't use it. The
cloud-kitchen profile offers exactly one preset, `catalog-clean`, and it's tuned
for something almost opposite: even light, neutral ground, dish centred, optimised
for **legibility at 150 pixels** because that's the thumbnail size on a phone.

## The 150-pixel point

This one reframes what "quality" means.

A delivery-app menu image is displayed at roughly 150 pixels. A photograph that is
gorgeous at 1024px and unidentifiable at 150px has failed at its only job. So the
cloud-kitchen preset optimises for silhouette clarity and subject-to-frame ratio,
not atmosphere.

It produces images that look *worse* in isolation and *work better* where they're
actually seen. That's a hard thing to accept when you're looking at a full-size
preview, and it's why it's encoded as a preset constraint rather than left to
taste.

## An unresolved wrinkle

The global Foodpanda requirements document specifies 4000×2925. Foodpanda Malaysia
[documents 1000×1000 square](https://foodphoto.ai/delivery-photo-specs/foodpanda/malaysia/).

Regional partner portals differ, and the Philippine specification wasn't
confirmable from public documentation. Rather than guess, the exporter produces
**both** — the cost of an extra crop is negligible, the cost of a merchant's whole
menu being rejected is not. Once a real PH partner portal is visible during pilot
onboarding, the unused target gets dropped.

## The lesson

**Read platform documentation before designing, not before shipping.**

Two features were designed, argued about, and specified in detail before the
documentation revealed that neither could exist on the surfaces that matter most
to a third of the target market. The reading took twenty minutes.

There's a second-order point too: these rules aren't arbitrary. "No watermarks, no
close-ups, top or front view" is Foodpanda enforcing catalogue consistency across
thousands of merchants, because a coherent menu grid converts better than a
beautiful inconsistent one. That's the same conclusion the cloud-kitchen locked-style
design reached independently. The platform had already solved it.
