# Automated Design Workflow — Planning Notes

> Working notes for building an automated, per-customer design service powered by
> the **Magnific** MCP connector. Captured from a Claude Code session.
> _Note: saved here in `Testing-claude-legal-` as a scratch location — move to the real
> Curbside business repo when one is in scope._

## Goal

Offer customers a full design service à la carte — menus, social media packs, brand
identity, product/ad visuals — produced through an automated, repeatable pipeline.
Different customers want different things (some only menus, some only social), so the
system should be modular.

---

## 1. Magnific connector

- Added as a **claude.ai connector** (`https://mcp.magnific.com`, HTTP transport).
- Persists in account connector settings; may need re-authorization if it drops.
- Auth must be done in claude.ai connector settings (not from a remote/non-interactive session).
- Account: Premium plan, 20,000 credits/month.

## 2. Service catalog (sellable deliverables)

**Menus & print**
- Branded menus (dine-in, takeout, seasonal) — generate with GPT-2, then editable layers
- Flyers, posters, table tents, loyalty cards

**Social media**
- Hero posts auto-resized to every platform format (`design_auto_resize`)
- Audience-targeted ad variants ("Audience-driven ads" flow)
- Short video promos / reels ("Product ad spot" flow, `video_generate`)
- Voiceover + music (`audio_tts`, `audio_music_generate`)

**Product & brand**
- Product photography / mockups ("Mockup realizer", "Marketplace listing visuals")
- Branded merch from a logo ("Brand merch shots", "Portfolio builder: Logo")
- Color variants, background removal, relighting, upscaling to 4K

**Premium add-ons**
- Product video ads, UGC-style spokesperson videos, 3D models
- Image upscaling/enhancement (Magnific flagship) for low-quality customer photos

## 3. Recommended models

| Model | Best for |
|-------|----------|
| **GPT-2** | Menus, text layout, infographics, typography, UI/diagrams (text-heavy design) |
| **Nano Banana Pro** | Brand fidelity, product shots, reference-guided edits, final assets |
| **Recraft V4.1** | Fast first drafts, illustration, pure text-to-image |

## 4. Arabic text — solved approach

AI image models mangle Arabic script, so **never let the model render Arabic**. Instead:

1. Generate the design/background with no text (or placeholder).
2. Open as editable layers (`design_auto_layers` / `design_auto_resize`) in the Designer.
3. Type real Arabic in the Designer using actual fonts — perfect RTL, full control.

Benefits: bilingual (AR/EN) menus, per-location price edits, single-item updates without
regenerating, full editability as a selling point.

## 5. Export / print formats

| Format | Supported | How |
|--------|-----------|-----|
| PNG | ✅ Native | Up to 4K, plus AI upscaling for larger print |
| SVG (vector) | ✅ | `images_generate_svg`, `images_to_svg` — scalable, ideal for logos/signage |
| JPG | ✅ | Standard raster |
| PSD (layered) | ✅ | Layered handoff |
| PDF | ⚠️ Partial | Exported from the Designer canvas, not the generation API |

PNG (4K) + SVG cover ~90% of real print/digital needs.

## 6. Spaces — the build layer

Spaces is a **node-based visual canvas** (flowchart / ComfyUI-style): each node is a step
(input, generation, edit, conversion); connections feed one node's output into the next.
Build the graph once, run the whole chain with one command.

**This is the engine for the business:** a Space is a reusable template. Per customer,
swap in their logo/assets → run the Space → out comes the deliverable set. Keep separate
Spaces per service ("Menu builder", "Social pack", "Full brand kit") and pick per customer.

Existing space already started: logo → Image-to-SVG → Image Generator (full brand identity
system for a local Saudi neighbourhood food-ordering service, Arabic-first).

Notes:
- Use specific aspect ratios (or multiple generator nodes) instead of `auto` for multi-asset boards.
- Keep Arabic out of generated images; add as text layers in the Designer after the run.

## 7. Per-customer flow (repeatable)

1. **Onboard brand** → upload logo, colors, product photos into the **Library** as reusable
   `style`/`product`/`character` references (consistency engine — the moat).
2. **Run the relevant Space(s)** for the services that customer bought.
3. **Add Arabic/finishing text** in the Designer.
4. **Export** PNG (4K) / SVG; PDF via Designer if needed.
5. **Deliver** webUrls or finalized files.

## 8. Cost / margin notes

- Still images are cheap (~15–75 credits each → hundreds/month on the plan).
- Video is the premium tier (~3,000–8,000 credits per clip) — price as a paid add-on.
- Use `simulate_cost` / `simulate_spaces` to estimate before running.

---

## Sample demo (fictional brand "Olive & Ember")

Generated to validate quality:
- Restaurant menu (GPT-2) — full sectioned menu with prices
- Instagram promo post (Nano Banana Pro)
- Social pack — same post auto-resized into Story / landscape / portrait formats

(Assets live in the Magnific account.)

---

## جِوار brand identity — kickoff prompt (paste into a fresh, Magnific-authorized session)

> The brief and decisions below were approved. A running session can't pick up
> connector auth done after it started — so start a NEW session with Magnific
> Connected, then paste this:

```
Build the جِوار brand identity into my existing Magnific Space (the one with my
uploaded logo). جِوار is a local neighbourhood food-ordering service: customers
order from nearby restaurants and get food via the restaurant's own local delivery
or curbside pickup to their car. Warm, honest, community-rooted; Arabic-first.
The name جِوار means "neighbourhood".

First pull my assets (creations_list / library_list) and find my logo plus my
reference images for: (1) someone handing over an order with the logo on the bag,
(2) a customer in a car in front of a store scanning a barcode to order from the car,
(3) a delivery person handing over an order, (4) a coffee cup with the logo.

Then generate 4 SQUARE (1:1) images with Nano Banana Pro, each using the logo +
its matching reference image:
  1. Order handoff (logo on the bag)
  2. Curbside / in-car barcode scan ordering
  3. Local delivery handoff
  4. Coffee cup with the logo

Rules for all images: warm and clean; one consistent palette (the logo's green +
orange + warm cream neutrals); same lighting/theme so they read as one family;
NO text in the images (I'll add Arabic myself). People do NOT need to look Saudi.
Build these as nodes in the SAME existing Space.

Finally, after actually viewing my logo, write a full logo design explanation
(meaning of جِوار / neighbourhood, color rationale, usage guidance).
```

### Approved decisions
- Square (1:1) images.
- No text in any generated image (added manually in Arabic later).
- People need NOT look Saudi.
- Model: Nano Banana Pro (brand/logo fidelity).
- Palette: logo green + orange + warm cream neutrals; warm & clean; consistent across all.
- Build into the existing Space (not a new one).
- Note: connector auth must be live at session start — re-auth done mid-session does not apply.

## Open items / next steps

- [ ] Run the جِوار kickoff prompt above in a Magnific-authorized session.
- [ ] Decide the real home repo for this workflow (Curbside) and re-scope a session to it.
- [ ] Build per-service Spaces (Menu builder, Social pack, Full brand kit).
- [ ] Set specific aspect ratios in the brand-kit Space.
- [ ] Build a per-service credit-cost + pricing sheet.
- [ ] Establish the Library onboarding step per customer.
