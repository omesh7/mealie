# Annam — a personal recipe brain on Cloudflare (plan v1)

Working name **Annam** (அன்னம், "food"). Rename freely; it is one string.

Status: **plan only, no code.** Draft v1 by the planner, before review.

---

## 0. Where you left off

- Branch `ccr-a0cc12c7-3urwd5` is identical to `mealie-next` = upstream
  Mealie **v3.28.0** at `9ca9cec` (2026-09-30). **Zero commits of yours.**
  Nothing is half-done; nothing is lost.
- So this fork was only ever "look at Mealie". The plan below explains why
  the app should not *be* Mealie, and what to take from it.

## 1. Can Mealie itself run on Cloudflare?

| Option | Verdict |
| :-- | :-- |
| Workers (incl. Python Workers) | **No.** FastAPI + SQLAlchemy + Alembic + yt-dlp + ffmpeg + a local image folder. Not portable to the Workers runtime. |
| Cloudflare Containers (GA Apr 2026) | **Technically yes, practically bad.** Container disk is ephemeral, so SQLite and `/app/data` images vanish on restart; you would bolt on external Postgres (Neon via Hyperdrive) and still lose images. A ~500 MB Python app cold-starts after sleeping. Mealie is multi-tenant (groups → households → users), has no MCP, and you would carry an upstream fork forever. |
| Mealie on a VPS + MCP proxy on Cloudflare | Works, but it is two systems, a server to patch, and Mealie's data model still does not know about pantry, Crohn's, variety or cost. |

**Decision: build a small Cloudflare-native app ("lifeD for food").**
Take from Mealie only what is worth stealing (section 9). Leave this fork as a
reference; the new app gets its own repo (`omesh7/annam`).

## 2. What the app is for (the five jobs)

1. **Capture in 5 seconds.** Scrolling Instagram / YouTube / a blog / WhatsApp
   → Share → Annam → done. No typing. Within ~1 minute it is a structured
   recipe with its source video linked, and a push says what it understood.
2. **"What should I eat today?"** Ask Claude (Android app, connector). It gets
   3–5 options that fit *you*: what's in the kitchen, what you haven't had
   lately, your goals, Crohn's phase, budget, time. Each with the source video.
3. **Adapt.** "Make it low-residue / more protein / no onion-garlic / for one
   person / under ₹60" → a saved **variant** that links to the original.
4. **Mix and match.** Recipes are also tagged as *components* (base, protein,
   veg, sauce/gravy, side, topping). "Build me a bowl from what I have" =
   combine components, not just pick a whole recipe.
5. **Remember what happened.** "Had the pesarattu, felt fine" → meal log →
   variety is real, favourites emerge, and (with lifeD's symptom notes)
   personal triggers become visible over time.

Non-goals (v1): multi-user, social, public sharing, rehosting videos,
calorie-perfect tracking, medical advice.

## 3. "Good for me" — personalisation that matters for you

From lifeD (#357): **Crohn's** (on Upadacitinib), **low B12, vitamin D, iron**,
food budget **₹5k/month** from Nov, a new **Tiruppur 1BHK** kitchen being set up.
A generic "healthy recipes" app would get this wrong. So:

- **Profile** (one record, editable by chat): conditions, *current phase*
  (`remission | flare | recovering`), hard avoids, learned triggers, targets
  (protein g/day, kcal), deficiencies to favour (B12, D, iron), budget,
  equipment, cuisine leanings, spice tolerance, veg/egg/non-veg, servings.
- **Every recipe gets a gut profile**, computed at import and editable:
  `residue: low|med|high`, insoluble fibre, spice heat, fat/fried, lactose,
  seeds/skins/nuts, raw veg — plus `flare_ok: yes|caution|no` with reasons.
- **Flare mode** ("I'm flaring") filters to `flare_ok = yes` (soft, low-residue,
  low-spice: kanji, curd rice if dairy tolerated, idli, steamed fish, etc.) until
  you say you're better. Personal tolerance (from logs) beats generic rules.
- **Nutrition** per serving (kcal, protein, carbs, fat, fibre, sodium, and
  "good source of" B12 / iron / vitamin D) is a model **estimate** with a
  confidence, shown as "≈". v3 swaps in IFCT 2017 (Indian Food Composition
  Tables) per ingredient for real numbers.
- **Cost ≈ ₹/serving** from a small price table you can correct by chat
  ("paneer is ₹90 per 200g now").
- **Guardrail:** the app flags, it never prescribes. No supplement doses, no
  "cures". Your doctor's instructions override everything; that sentence
  lives in the MCP server instructions.

## 4. Architecture

One Worker, one repo. Everything on Workers Paid.

```
 Android share sheet ──► PWA share_target (POST multipart) ─┐
 Claude / ChatGPT  ──► /mcp  (OAuth, owner passphrase)  ────┤
 PWA UI (/app)     ──► /api (session cookie)            ────┤
                                                            ▼
                         Hono Worker  ── D1 (Drizzle) ── R2 (thumbs, uploads, backups)
                              │
                              ├─► Workflow "ingest"  (durable, retried steps)
                              │      ├─ web:      fetch → JSON-LD schema.org Recipe (no AI)
                              │      ├─ youtube:  Data API description → Gemini (YouTube URL) if needed
                              │      ├─ instagram:Browser Run render of /embed/captioned → caption
                              │      │            → if no recipe in caption: video → Gemini
                              │      ├─ image:    Gemma 4 vision (screenshots)
                              │      └─ text:     Gemma 4 JSON schema
                              │      → enrich (gut profile, nutrition ≈, cost ≈, components)
                              │      → dedupe → save (draft|ready) → ntfy push
                              │
                              ├─► AI Gateway → Workers AI (Gemma 4) / Gemini
                              └─► Cron: nightly R2 backup, weekly "variety" digest
```

### 4.1 Two brains, one rule

- **Conversation brain = Claude** (your subscription) via MCP. It reads facts
  and writes facts. Adapting, combining, and explaining are its job, at zero
  cost to the app. (Autogram's rule: *the user's own model does the thinking*.)
- **Ingest brain = Workers AI (Gemma 4 26B A4B)**, only for the share-sheet
  path where no chat is open. Cheap (~$0.10 / $0.30 per M tokens), JSON
  schema output, vision for screenshots, thinking off.
- **Video brain = Gemini** (Flash tier), only when caption/description has no
  recipe. It accepts a public YouTube URL directly and an uploaded MP4.
  `autogram/apps/api/src/lib/video-analysis.ts` already does upload → poll.
- All model calls go through **AI Gateway** (logs, retries, caching, a spend cap).

### 4.2 Source adapters (the hard part)

| Source | First try | Fallback | Last resort |
| :-- | :-- | :-- | :-- |
| Recipe blog | `fetch` + `HTMLRewriter` → JSON-LD `Recipe` (most blogs have it; zero AI) | Browser Run `/markdown` → Gemma | `needs_input` |
| YouTube / Shorts | Data API v3 `videos.list` (title, description, thumb) → Gemma if description has a recipe | Gemini with the YouTube URL (sees + hears the video) | `needs_input` |
| Instagram reel/post | Browser Run renders `instagram.com/reel/<code>/embed/captioned/` → caption + thumbnail (plain `fetch` and oEmbed no longer return captions, per 2025–26 reports) | video URL from the rendered embed → R2 temp → Gemini | `needs_input`: push "Couldn't read this reel — share a screenshot or paste the caption" |
| Screenshot(s) | Gemma 4 vision → recipe JSON | — | `needs_input` |
| Plain text / WhatsApp | Gemma 4 JSON schema | — | — |

- **A share is never lost.** `/share` writes a `captures` row *before* any
  fetch, then starts the Workflow. Failures land in an Inbox with the reason.
- `needs_input` uses Workflows `step.waitForEvent`: the run pauses until you
  add a screenshot/text to that capture in the app, then resumes.
- Measured, not hoped: the Inbox shows success rate per platform. If Instagram
  falls below ~70%, Phase 3 adds a **Containers + yt-dlp** adapter. Not before.

### 4.3 Stack

| Layer | Choice | Why |
| :-- | :-- | :-- |
| HTTP | Hono 4 | Same as autogram/lifed |
| DB | D1 + Drizzle ORM + drizzle-kit migrations | Same as autogram; `db.batch()` makes recipe+ingredients+sources one atomic write |
| Validation | Zod (v4) | Tool inputs, model JSON output, API bodies — one schema each |
| Search | D1 FTS5 on title/ingredients/tags/notes | < 2k recipes; Vectorize only if FTS proves weak |
| Jobs | Cloudflare Workflows | Durable multi-step ingest with retries + `waitForEvent` |
| AI | Workers AI (Gemma 4) + Gemini via AI Gateway | See 4.1 |
| Pages | Browser Run (Browser Rendering) binding | Instagram embed + JS-heavy blogs |
| Files | R2 | Thumbnails (re-hosted privately; IG CDN URLs expire), uploads, backups |
| Push | ntfy (as lifeD) | Free, already on your phone |
| UI | Vite + React SPA served by the same Worker's static assets | Mobile-first, installable, share target |
| Tooling | TS, oxlint, prettier, vitest + `@cloudflare/vitest-pool-workers` | Same as your repos |

"New batching APIs" used where they pay: **D1 `batch()`** (atomic writes) and
the **Workers AI async Batch API** (`queueRequest: true`) for backfills — e.g.
re-scoring every recipe's gut profile after the rules change.

### 4.4 MCP server

- **Protocol:** MCP **2026-07-28** (stateless) with legacy 2025 compatibility.
  Two ways, pick one in review:
  - (a) `createMcpHandler(factory)` from the official MCP TS SDK v2
    (`@modelcontextprotocol/server`), which Cloudflare now uses for its own
    MCP server; Zod tool schemas; SDK owns protocol churn.
  - (b) Port lifeD's `src/mcp.ts` (276 lines, dual-era, zero deps, proven with
    Claude web/desktop/Android and ChatGPT).
  Planner leans **(a)** — you said you don't want to hand-roll again — but
  only if the SDK v2 build is stable on npm on the day.
- **Auth:** port lifeD's `src/auth/*` (OAuth 2.1, CIMD + stateless DCR, owner
  passphrase, D1 grant row, `ADMIN_TOKEN` bearer for scripts). Proven with
  every client you use. One user, so no accounts table.
- **Tools** (few, short descriptions — they ride in every chat):

| Tool | Read/Write | What |
| :-- | :-- | :-- |
| `kitchen` | R | No args: the brief — phase, targets, avoids, pantry highlights, last 7 days of meals, inbox drafts to review, today's plan. With `q`: best-matching recipes. Call first. |
| `suggest` | R | Ranked options for a slot (`breakfast/lunch/dinner/snack`), with `mode: whole|combo`. Scores pantry coverage, days since eaten, cuisine/protein rotation, goal fit, `flare_ok`, time, ₹. Returns reasons + missing items. |
| `find` | R | Filtered one-liners: q, role, cuisine, max_min, flare_ok, uses[], not_eaten_days, min_protein, max_inr, tag. |
| `get` | R | Full recipe(s) incl. sources (video links, thumbnails), variants, notes, your log of it. |
| `write` | W | One call, many ops (lifeD style, `$i` refs): `recipe.upsert` (incl. variant via `parent`), `meal.log`, `pantry.set/add/use`, `profile.set`, `plan.set`, `shop.add/done`, `price.set`. |
| `import` | W | url / text / image URL → starts ingest, waits ≤20 s, returns the draft (or capture id + status) so Claude can fix it in the same chat. |

- **Server instructions** carry the operating rules: call `kitchen` first,
  respect phase and avoids, offer 3 options with one-line "why", cite the
  source video, never give medical advice, save adaptations as variants.

## 5. Data model (D1)

- `captures` — id, at, via (share/mcp/app), url, text, files (R2 keys), status
  (`queued|working|ready|needs_input|failed`), reason, recipe_id, run_id.
- `recipes` — id, slug, title, title_ta, summary, cuisine, roles (json:
  `main|base|protein|veg|gravy|side|breakfast|snack|drink|topping`), servings,
  prep/cook/total min, difficulty, equipment, status (`draft|ready|archived`),
  parent_id + variant_note, rating, notes, steps (json), nutrition (json, per
  serving, + confidence), gut (json: residue, spice, fat, lactose, fibre,
  flare_ok, reasons), inr_per_serving, created/updated.
- `ingredients` — recipe_id, pos, section, qty, unit, food_id, raw, note,
  optional, swaps.
- `foods` — id, name, aliases (Tamil/Hindi/English: kothamalli = coriander),
  category, staple, price (₹ per unit), gut flags, IFCT code (v3).
- `sources` — id, recipe_id, platform, url (canonical), author, title, caption,
  transcript, thumb_r2, embed_url, fetched_at. **Many per recipe** — three reels
  of the same dish live under one recipe.
- `tags`, `recipe_tags`.
- `pantry` — food_id, qty, unit, updated_at, use_by.
- `meals` — at, slot, recipe_id?, text, portion, rating, felt (free text), note.
- `plan` — date, slot, recipe_id, status. `shopping` — food_id/text, qty, done.
- `profile` — single JSON row (section 3). `settings` — ntfy topic etc.
- FTS5 virtual table over recipes + ingredients + tags.

Symptoms and meds stay in **lifeD** (your health source of truth, already fed
by your Food & Health assistant). Claude, holding both connectors, correlates
on request. A direct bridge is Phase 3.

## 6. Android

- **v1: installable PWA** (Chrome → Install). Manifest `share_target` with
  `method: POST`, `enctype: multipart/form-data`, params `title/text/url` and
  `files: image/*, video/*`. Shows in the share sheet next to WhatsApp.
  Instagram and YouTube share *text with a URL*; screenshots share *files*.
  lifeD already proves the GET/text half of this on your phone.
- The share page answers in < 1 s ("Saved ✓ — reading it now") and returns you
  to the app you came from; the push arrives when the recipe is ready.
- **Also works with no app at all:** share the link into the Claude app and
  say "save this" → Claude calls `import`. (Whether Claude's Android app
  accepts share intents for links: verify on device.)
- **Native app: not in v1.** Revisit only for a home-screen "What should I eat?"
  widget, offline cooking mode, or if PWA share proves flaky. If so, a TWA
  (Bubblewrap) wrapper first, Kotlin second.

## 7. The app UI (PWA)

- **Today**: 3 suggestions (same engine as `suggest`) with photo, ⏱, ₹, protein,
  gut badge; tap → recipe. "Flare mode" toggle at the top.
- **Inbox**: new captures — ready drafts to approve, `needs_input` to fix.
- **Library**: grid with filters (role, cuisine, time, flare_ok, tags), search.
- **Recipe**: source video embed/thumbnail + "open in Instagram/YouTube", scale
  servings, ingredients (in-pantry ticks), steps with timers, variants, log it.
- **Mix**: pick base + protein + veg + gravy from compatible components.
- **Pantry / Shopping**: quick add, "used up", list grouped by shop aisle.
- Auth: owner passphrase → 30-day cookie (lifeD's `owner.ts`).

## 8. Cost (single user, ~100 imports/month)

Workers Paid $5/mo covers Workers, D1, R2, Workflows at this scale. Workers AI
(Gemma) ≈ cents/month. Browser Run within the Paid allowance. Gemini only for
video fallbacks ≈ < $1/month. **≈ $5–6/month total.** AI Search (billing from
2026-11-01) and Vectorize are not used in v1.

## 9. Reuse map

| From | Take |
| :-- | :-- |
| lifeD | `auth/*` (OAuth, passphrase, ADMIN_TOKEN), `mcp.ts` shape + instructions builder, `write` ops with `$i` refs, ntfy push (`remind.ts`), nightly R2 backup (`archive.ts`), manifest + share route, tooling configs |
| autogram | Gemma 4 call pattern + neuron accounting (`lib/ai-writer.ts`), Gemini upload/poll (`lib/video-analysis.ts`), AI spend cap (`lib/ai-budget.ts`), Drizzle + drizzle-kit migration discipline, Zod-validated tool args (`lib/mcp-args.ts`) |
| Mealie | schema.org Recipe field set; prompts `parse-recipe-video`, `parse-recipe-ingredients`, `build-recipe` (as starting text); the qty/unit/food ingredient split; idea of many-sources-per-recipe |

## 10. Phases and "done" checks

**P0 — talkable (1–2 days).** Repo from lifeD skeleton; D1 schema + migrations;
OAuth + MCP (`kitchen/find/get/write`); connector added on claude.ai, used from
Claude Android. Paste a recipe into Claude → it structures and saves it (Claude
does the parsing, no Workers AI yet).
*Done when:* "save this recipe: <paste>" then "what can I cook with eggs and
rice?" works on the phone.

**P1 — share sheet (3–5 days).** PWA + share_target; `captures` + Inbox; ingest
Workflow with web JSON-LD, YouTube, Instagram embed, image, text adapters;
gut profile + nutrition ≈ + cost ≈ enrichment; ntfy push; `import` tool.
*Done when:* 10 real shares (4 IG reels, 3 YouTube, 2 blogs, 1 screenshot)
→ ≥ 8 land as usable recipes without typing; failures say why.

**P2 — decide for me (week 2).** `suggest` (whole + combo), pantry, meal log,
profile + flare mode, shopping list, Today screen, weekly variety digest.
*Done when:* "what should I eat today?" returns 3 options you would actually
cook, none eaten in the last 4 days, flare-safe when flare mode is on.

**P3 — polish (later, each optional).** MCP Apps recipe cards inside Claude;
IFCT nutrition; Vectorize "similar recipes" + dedupe; week plan; lifeD bridge
(meals → lifeD events, symptoms → correlation); Containers + yt-dlp if IG
success < 70%; TWA/native.

## 11. Risks

| Risk | Mitigation |
| :-- | :-- |
| Instagram blocks/changes the embed page | Layered fallbacks, `needs_input` never loses the share, per-platform success metric, yt-dlp container held in reserve |
| Model invents ingredients/quantities | Prompt: "never invent; null if unknown"; keep raw caption/transcript beside the recipe; drafts below a confidence go to Inbox, not Library |
| Nutrition numbers trusted too much | Always "≈" + confidence; IFCT later |
| Health advice overreach | Flags only; doctor overrides; no doses |
| Copyright | Private app, stores link + caption + transcript + private thumbnail; never republishes videos |
| Protocol churn (MCP spec, Claude connector bugs) | SDK v2 or lifeD's proven dual-era handler; `ADMIN_TOKEN` path for scripts |

## 12. Questions only you can answer

1. Veg / egg / non-veg? Any foods your doctor has told you to avoid?
2. Kitchen in the new 1BHK: stove burners, pressure cooker, mixie, oven/air
   fryer, fridge? Who cooks — you, a cook, or both?
3. Targets: protein per day? weight goal? Is ₹5k/month all food or groceries only?
4. Want weekly meal plans, or only "what now?"
5. Tamil dish names alongside English?
6. Name: keep "Annam"?
