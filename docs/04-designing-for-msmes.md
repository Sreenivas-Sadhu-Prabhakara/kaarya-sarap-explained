# Designing for MSMEs, not for developers

The first design had a prompt box. It was wrong, and the reason it was wrong took
a while to see.

## The mistake

A text field where you describe the image you want is the obvious interface for an
image generator. It's what every tool in the category ships.

It also assumes the user can describe an image *in the register the model
responds to*. `"85mm, f/1.4, shallow depth of field, volumetric side lighting,
shot on Portra 400"` is a real skill. It is not a skill a restaurant owner in
Quezon City has, or should need to acquire to photograph their own adobo.

They know one thing with total precision: **the name of the dish.**

So the input is a dish picker. Free text survives as an advanced escape hatch,
used by almost nobody, which is the correct amount of use for it.

## What this moves

Once the user picks a dish rather than writing a prompt, the prompt has to come
from somewhere. That somewhere is a curated dictionary: dish name → an engineered
prompt describing what the dish actually looks like, plus a negative prompt naming
the dishes it gets confused with.

This relocates the entire difficulty of the project.

- **Before:** the hard part is infrastructure. Get a GPU, get an API, ship a UI.
- **After:** the hard part is a corpus of culinary prompt engineering, and the
  infrastructure is a weekend.

Which is also where the defensibility moved. Anyone can tunnel ComfyUI. The
dictionary is the thing that took the work — and it's why the code repository is
private while this one is public.

## Four other things that follow

**The output isn't an image.** Nobody needs a 1024×1024 PNG. They need a Facebook
post, a story, a GrabFood thumbnail, a Foodpanda menu image. Generate once, export
the full set. Handing over a bare PNG and letting a merchant crop it in their
phone gallery is handing over homework.

**Text must not go through the model.** They want `₱149` on the image. Diffusion
models render text unreliably. A slightly-wrong price isn't a cosmetic glitch —
it's a merchant advertising the wrong number. So text is composited afterwards as
a raster layer, always, with no exceptions.

**Bulk beats interactive.** A restaurant has forty dishes, not one. One-at-a-time
is a toy. Upload the menu, queue everything, collect it in the morning.

And this is where the platform's weakest property stops mattering: 150 seconds per
image is painful when someone is watching a spinner, and completely irrelevant
overnight. The constraint that looked fatal for interactive use is a non-issue for
the workflow that actually delivers value.

**Business type changes the product, not just a setting.** A fine-dining plate
optimises for atmosphere — dark ground, shallow focus, negative space. A cloud
kitchen's delivery thumbnail optimises for **legibility at 150 pixels**, because
that's the size it will be seen at. Those are opposing objectives, not two points
on one slider.

The cloud-kitchen case has a further twist: forty dishes shot in forty styles looks
amateur, and forty in one consistent style looks like a brand. So consistency
across the catalogue outranks the quality of any individual image — the opposite of
what you'd optimise for anywhere else. Each delivery-only tenant gets a locked
style: fixed preset, background, lighting and seed offset.

## The reframe

The product isn't an image generator. It's a **thing that produces the marketing
assets a small food business needs**, which happens to use an image generator
internally.

Once framed that way, the prompt box looks like what it is — an implementation
detail that escaped into the interface.
