# Automated Design Service — Implementation Spec

> Full automation blueprint for the per-customer design service powered by the
> **Magnific** MCP connector. Carry this into the real business repo/session and
> build against it. Companion to `WORKFLOW.md` (which holds the manual process
> and the جِوار brand decisions). This file = how to make that process repeatable
> and hands-off.
>
> **Status:** the manual pipeline is validated end-to-end (logo → 4 reference
> photos → 4 on-brand Nano Banana Pro scenes → assembled in a Space). This spec
> turns that proven flow into a product.

---

## 0. TL;DR

We proved we can produce a branded deliverable set by hand. To automate it, we
separate **fixed logic** (prompts, palette rules, aspect ratios, "no Arabic in
image", the scene list) from **per-customer inputs** (logo SVG, brand colors,
reference photos, services bought). The fixed logic gets frozen into **template
Spaces**; the inputs get injected per customer from their **Library + Brand Kit**;
an **orchestration script** runs it all on a trigger.

Three levels, build in order:

| Level | What | Effort | Payoff |
|-------|------|--------|--------|
| **1. Template Spaces** | Reusable node graphs, swap inputs, one-click run | Low | 80% of the value |
| **2. Library + Brand Kit** | Per-customer asset/brand store = consistency engine | Medium | "Select customer → Run" |
| **3. Orchestration script** | Intake → run → export → deliver, no human in loop | Higher | True hands-off |

Target outcome: **one-click + a short QA pass**, not zero-touch. Arabic text and
final creative sign-off stay human by design.

---

## 1. Architecture overview

```
                    ┌─────────────────────────────────────────────┐
   Customer intake  │              ORCHESTRATION LAYER             │
   (form / webhook) │   (script or agent driving the Magnific API) │
        │           └─────────────────────────────────────────────┘
        ▼                         │            │            │
  ┌───────────┐                   ▼            ▼            ▼
  │  INTAKE   │            ┌────────────┐ ┌──────────┐ ┌──────────┐
  │  logo SVG │   ──────►  │  LIBRARY   │ │ BRAND KIT│ │ TEMPLATE │
  │  colors[] │            │ per-cust.  │ │ palette+ │ │  SPACES  │
  │  photos[] │            │ refs       │ │ logo     │ │ (per     │
  │  services │            └────────────┘ └──────────┘ │ service) │
  └───────────┘                   │            │       └──────────┘
                                  └─────┬──────┘            │
                                        ▼                   ▼
                                  ┌──────────────────────────────┐
                                  │  RUN (spaces_run / flows_run) │
                                  │  → poll → export PNG4K/SVG    │
                                  └──────────────────────────────┘
                                        │
                                        ▼
                              ┌────────────────────┐
                              │ QA + Arabic layers  │  ← human, in Designer
                              │ (Designer)          │
                              └────────────────────┘
                                        │
                                        ▼
                                    DELIVER (webUrls / files)
```

---

## 2. The Magnific primitives you build on

These are the actual MCP tools the pipeline uses. Group them by role:

**Account / guardrails**
- `account_balance` — check credits before a run.
- `simulate_cost`, `simulate_spaces`, `simulate_flows` — estimate credit cost of a
  generation / Space / flow **before** running. Use as a pre-flight gate.

**Assets in / Library (per-customer consistency engine)**
- `creations_request_upload` → `creations_finalize_upload` (or `creations_upload_image`)
  — push a customer's logo/photos into their account.
- `library_create` — register 1–6 images as a reusable asset
  (`type`: character | style | element/product | locations). Returns a numeric `id`.
- `library_list` / `library_show` — enumerate / pick assets. Each entry's numeric
  `id` is what you pass into generation and Space nodes.

**Generation**
- `images_generate` — the workhorse. Key params:
  - `mode`: model slug. **`imagen-nano-banana-2`** = Nano Banana Pro (brand fidelity,
    final assets). `imagen-nano-banana-2-flash` = faster/cheaper. `gpt-2` = text-heavy
    design (menus). `recraft-v4` family = fast drafts/illustration.
  - `aspectRatio`: use explicit ratios (`1:1`, `4:5`, `9:16`, …), **never `auto`** for
    multi-asset boards.
  - `resolution`: `1k` | `2k` | `4k`.
  - `references`: array of `{type, identifier}`. `type` ∈ `style|character|product|image`.
    For a raw creation/upload use `{type:"image", identifier:"<creationId>"}`; for a
    Library asset use `{type:"<asset type>", identifier:<numeric id>}`.
  - `brandKitId`: apply a saved palette/logo/typography automatically (Level 2).
  - `count`: images per call.
- `images_generate_svg` / `images_to_svg` — vector output (logos, signage).
- `design_auto_layers` — open a generation as editable layers (for Arabic text later).
- `design_auto_resize` — auto-resize one hero into every platform format (social pack).
- Background removal, relight, upscale, crop, variations — finishing ops.

**Spaces (the template engine)**
- `spaces_create` — new node canvas.
- `spaces_add_creations` — drop creations in as nodes (we used this).
- `spaces_get_nodes` / `spaces_state` — read the graph.
- `spaces_edit` / connections — wire nodes, set node params.
- `spaces_run` → `spaces_run_status` — execute the whole graph, poll to completion.

**Flows (pre-built pipelines)**
- `flows_list` / `flows_show` — discover ready-made multi-step flows
  (e.g. "Audience-driven ads", "Product ad spot", "Mockup realizer").
- `flows_run` → `flows_wait` — run one and wait for the result.

**Premium / add-ons**
- `video_generate`, `video_speak`, `video_concatenate`, `video_upscale` — promo reels.
- `audio_tts`, `audio_music_generate` — voiceover + music.
- `models3d_generate` — 3D.

**Status helpers**
- `creations_wait` — block until one or more creations reach a terminal state and
  return final asset URLs (use when chaining to export/video).
- `creations_get` / `creations_show` / `creations_search` — fetch metadata / display /
  search history.

---

## 3. Per-customer data model (the intake)

Everything per-customer reduces to one record. Define it once and the whole
pipeline keys off it.

```jsonc
{
  "customerId": "jiwar",
  "name": "جِوار",
  "meaning": "neighbourhood",                 // for copy / logo write-up
  "market": "Saudi, Arabic-first",
  "brand": {
    "logoSvgCreationId": "<svg creation id>", // editable vector logo
    "logoRasterCreationId": "<png id>",       // fallback for AI reference
    "palette": {
      "green":  "#0A7D4B",
      "orange": "#EC5F33",
      "cream":  "#F<cream>",
      "dark":   "#<neutral dark>"
    },
    "brandKitId": null,                        // filled after Brand Kit creation
    "rules": [
      "Never render Arabic inside generated images — add as font layers later.",
      "Keep the logo unaltered; prefer SVG overlay over baked raster.",
      "One consistent warm palette across every asset.",
      "People need not look Saudi."
    ]
  },
  "references": {                              // uploaded photo creation ids
    "orderHandoff": "<id>",
    "carBarcodeScan": "<id>",
    "deliveryHandoff": "<id>",
    "coffeeCup": "<id>"
  },
  "servicesBought": ["brand-scene-pack", "menu", "social-pack"],
  "delivery": { "formats": ["png-4k", "svg"], "channel": "email" }
}
```

The **fixed** side (never per-customer) lives in the template, not the record:
prompt text, aspect ratios, model choice, "no text" rule, scene order.

---

## 4. Level 1 — Template Spaces

A Space is a graph you run with one command. Build **one template per service**.
The pattern is always: *input nodes (per-customer) → generator nodes (fixed) →
output nodes*.

### 4.1 "Brand scene pack" template (the one we validated)

Nodes:
```
INPUTS (swap per customer)         GENERATORS (locked)              OUTPUTS
─────────────────────────          ──────────────────────          ───────
[ logo SVG ]            ─┬────────► [ NB-Pro: order handoff ]  ───► [ out 1 ]
[ ref: order handoff ]  ─┘
[ ref: car scan ]       ─┬────────► [ NB-Pro: car barcode ]    ───► [ out 2 ]
[ logo SVG ]            ─┘
[ ref: delivery ]       ─┬────────► [ NB-Pro: delivery ]       ───► [ out 3 ]
[ logo SVG ]            ─┘
[ ref: coffee cup ]     ─┬────────► [ NB-Pro: coffee cup ]     ───► [ out 4 ]
[ logo SVG ]            ─┘
```

Each generator node has its prompt, palette hex codes, `aspectRatio: 1:1`,
`resolution: 2k`, `mode: imagen-nano-banana-2` **baked in**. Per customer you only
re-point the input nodes. Hit `spaces_run`, poll `spaces_run_status`, collect the
four output creation ids.

The four locked prompts are exactly the ones already proven (see `WORKFLOW.md`
§ kickoff and the validated run). Keep them in the node config so they never drift.

### 4.2 Other service templates (same pattern)

- **"Menu builder"** — `gpt-2` generator producing a sectioned menu with prices on a
  clean cream/green/orange layout → `design_auto_layers` so Arabic + per-location
  price edits happen in the Designer.
- **"Social pack"** — one hero post generator → `design_auto_resize` fan-out to
  Story / feed / landscape / portrait.
- **"Full brand kit"** — combines logo usage board + scene pack + social templates.

Keep them **separate** so you can sell à la carte and run only what a customer
bought.

### 4.3 Build-once checklist

- [ ] Create the Space (`spaces_create`).
- [ ] Add placeholder input nodes + generator nodes (`spaces_add_creations` / edit).
- [ ] Lock prompt + palette + ratio + model on each generator.
- [ ] Wire connections (input → generator → output).
- [ ] Test-run once; confirm outputs.
- [ ] Tag it as the canonical template; clone per customer (never edit the master).

---

## 5. Level 2 — Library + Brand Kit (consistency engine)

Onboard each customer **once** so later runs are "select customer → Run".

**Onboarding steps**
1. Upload logo (SVG + PNG) and reference photos → `creations_request_upload` /
   `creations_upload_image`.
2. Register reusable assets → `library_create`:
   - logo → `element`/`style`
   - product photos → `product`
   - recurring people/mascot → `character`
   - store/location shots → `locations`
3. Build a **Brand Kit** (palette + logo + typography). Store its id as
   `brand.brandKitId` in the intake record.
4. From then on, every `images_generate` / Space generator passes `brandKitId`, so
   palette + logo come out on-brand automatically — no per-prompt color wrangling.

This is the **moat**: the longer a customer is onboarded, the more their Library
encodes their exact brand, and the cheaper/faster every future deliverable gets.

---

## 6. Level 3 — Orchestration script (hands-off)

A thin program (or agent) that drives the Magnific MCP/HTTP API from an intake
record. Pseudocode:

```python
def run_customer(intake):
    # 0. Guardrail
    est = simulate_cost(plan_for(intake.servicesBought))
    assert account_balance().credits.available >= est, "top up credits"

    # 1. Ensure assets onboarded (idempotent)
    if not intake.brand.brandKitId:
        onboard(intake)            # upload → library_create → brand kit

    results = {}
    for service in intake.servicesBought:
        space = clone_template(TEMPLATES[service])      # never edit master
        bind_inputs(space, intake)                      # logo + refs + palette
        run = spaces_run(space.id)
        outputs = poll(spaces_run_status, run.id)        # until terminal
        results[service] = export(outputs, intake.delivery.formats)  # PNG4K/SVG

    handoff_for_qa(results)        # human: Arabic layers + sign-off in Designer
    return results
```

**Key implementation notes**
- **Idempotent onboarding** — skip upload if `brandKitId` already set.
- **Clone, don't mutate** — each run clones the template Space so concurrent
  customers never collide.
- **Poll, don't block forever** — `spaces_run_status` / `creations_wait` with a
  timeout + retry/backoff.
- **Pre-flight cost gate** — `simulate_cost` then compare to `account_balance`;
  refuse to run if it would overspend.
- **Persist the mapping** — store every output creation id + webUrl against the
  customer for re-delivery and audit.

**Triggers** (pick one to start):
- Customer intake form (Tally / Typeform) → webhook → `run_customer`.
- A GitHub Action `workflow_dispatch` taking the intake JSON.
- A queue row / admin button.

---

## 7. What stays manual (by design)

- **Arabic text** — AI mangles Arabic script. Always generate text-free (or
  placeholder), then add real Arabic as font layers in the Designer
  (`design_auto_layers`). This gives perfect RTL, per-location price edits, and
  full editability — a selling point, not a limitation.
- **Logo fidelity QA** — AI placement can distort a mark. Prefer overlaying the
  **SVG logo node** over a baked raster; glance at every asset before delivery.
- **Creative sign-off** — one human approval before it goes to the customer.

So the honest target is **one-click generation + a short QA pass**.

---

## 8. Cost model

- **Still images** are cheap (~15–75 credits each) → hundreds/month on the plan.
- **Video** is the premium tier (~3,000–8,000 credits/clip) → price as a paid add-on,
  not bundled.
- Always `simulate_cost` / `simulate_spaces` before a run and gate on
  `account_balance`. Build a per-service credit→price sheet so margins are explicit
  before you quote a customer.

| Service | Rough credit cost | Notes |
|---------|-------------------|-------|
| Brand scene pack (4× NB-Pro 2k) | ~300 | the validated run |
| Menu (gpt-2 + layers) | ~30–75 | text-heavy, cheap |
| Social pack (hero + auto-resize) | ~75–150 | one gen + resizes |
| Video promo | 3,000–8,000 | premium add-on |

(Confirm exact numbers with `simulate_cost` per template before quoting.)

---

## 9. Implementation roadmap

**Phase 1 — Templatize (do first)**
- [ ] Convert the validated جِوار scene Space into a clean **"Brand scene pack"
      template** (inputs separated from locked generators).
- [ ] Build "Menu builder" and "Social pack" templates.
- [ ] Set explicit aspect ratios on every generator.

**Phase 2 — Onboarding**
- [ ] Define the intake record (§3) in the business repo.
- [ ] Script idempotent onboarding (upload → `library_create` → Brand Kit).
- [ ] Onboard جِوار as customer #1.

**Phase 3 — Orchestration**
- [ ] Implement `run_customer` (§6) against the MCP/HTTP API.
- [ ] Add `simulate_cost` pre-flight gate + credit check.
- [ ] Wire one trigger (form webhook or GitHub Action).

**Phase 4 — Delivery & ops**
- [ ] Auto-export PNG 4K / SVG; PDF via Designer where needed.
- [ ] Persist customer → output mapping.
- [ ] Build the per-service credit→price sheet.
- [ ] Document the human QA + Arabic-layer step as a checklist.

---

## 10. Kickoff prompt for the business session

> Start a **new session in the real business repo** with the **Magnific connector
> Connected at session start** (auth done mid-session does NOT apply). Paste:

```
Read AUTOMATION.md and WORKFLOW.md. We have validated the manual design pipeline
(logo → reference photos → 4 on-brand Nano Banana Pro scenes → assembled in a
Space). Now implement Phase 1 + Phase 2 from AUTOMATION.md:

1. Take my existing "Brand scene pack" Space and turn it into a clean reusable
   TEMPLATE: separate the per-customer input nodes (logo SVG + the 4 reference
   photos) from the generator nodes, and lock each generator's prompt, palette
   (#0A7D4B green, #EC5F33 orange, cream + dark neutral), aspectRatio 1:1,
   resolution 2k, mode imagen-nano-banana-2. Do not edit the master after — we
   clone it per customer.

2. Build two more templates the same way: "Menu builder" (gpt-2 + design_auto_layers
   for Arabic) and "Social pack" (one hero + design_auto_resize).

3. Onboard جِوار as customer #1: upload the logo (SVG + PNG) and reference photos,
   register them with library_create, and build a Brand Kit; store the brandKitId.

Rules: never render Arabic inside generated images (font layers in the Designer
after); keep the logo unaltered, prefer SVG overlay; simulate_cost before any run
and check account_balance. Report the template Space ids and the jiwar brandKitId
when done.
```

---

*Move this file into the business (Curbside) repo once that session is scoped to it.
It is saved here only because this repo was the scratch location.*
