# Annam — a personal recipe brain on Cloudflare (plan v2)

Working name **Annam** (அன்னம், "food"). Rename freely; it is one string.

Status: **plan only, no code.** v3 = v1 + review rounds 1–2 (see §13).

---

## 0. Where you left off

- Branch `ccr-a0cc12c7-3urwd5` had **zero commits of yours**; it was upstream
  Mealie v3.28.0. It is now merged with upstream **v3.30.0** (`bc21069`,
  2026-11-01) and carries only this plan. The fork's `mealie-next` is synced
  to upstream too.
- So this fork was only ever "look at Mealie". The new app gets its own repo
  (`omesh7/annam`); this fork stays as a reference.

## 1. Can Mealie itself run on Cloudflare?

| Option | Verdict |
| :-- | :-- |
| Workers (incl. Python Workers) | **No.** FastAPI + SQLAlchemy + Alembic + yt-dlp + ffmpeg + a local image folder. |
| Cloudflare Containers (GA Apr 2026) | **Technically yes, practically bad.** Ephemeral disk loses SQLite and images; needs external Postgres via Hyperdrive; a heavy Python app cold-starts after sleep; multi-tenant model (groups → households) you don't need; no MCP; an upstream fork to carry forever. |
| Mealie on a VPS + MCP proxy | Two systems and a server to patch, and Mealie still knows nothing about Crohn's, variety, or your saved reels. |

v3.30.0 added weekly meal planning, in-app AI prompt editing, step timers and
an iCal feed. Good ideas to borrow (§9), not a reason to host it.

**Decision: a small Cloudflare-native app — "lifeD for food".**

## 2. What it is for

1. **Capture in 5 seconds.** Instagram / YouTube / blog / WhatsApp → Share →
   Annam. A push says what it understood.
2. **Clear the backlog.** Import the reels and videos you already saved, in
   every Instagram account and YouTube playlist, once. (The "digging through
   saved videos across accounts" problem.)
3. **"What should I eat today?"** in Claude (Android app, connector): 3–5
   options that fit what you have, what you haven't had lately, your gut and
   goals, ₹ and time — each with its source video.
4. **Adapt.** "Low-residue / more protein / for one / under ₹60" → a saved
   **variant** linked to the original.
5. **Mix and match.** Recipes are tagged by **role** (base, protein, veg,
   gravy, side, topping, breakfast, snack). Claude combines roles from
   `recipes(role:…, have:true)`; no combo engine needed.
6. **Remember.** "Had the pesarattu, felt fine" → meal log → real variety,
   favourites rise, and (with lifeD's symptom notes) personal triggers show up.

Non-goals: multi-user, social, public pages, rehosting videos, calorie
tracking, medical advice.

## 3. "Good for me" — the health model

From lifeD (#357): **Crohn's on Upadacitinib**, low **B12 / vitamin D / iron**,
food budget **₹5k/month**, new **Tiruppur 1BHK** kitchen.

**Principle: store facts about food; decide "suitable" at query time from
rules you (and your doctor) own.** A model never stores "safe for Crohn's".

- **Per-recipe factual flags**, derived by code, not by a model: the
  extractor outputs each ingredient's **canonical English name** (kothamalli →
  coriander, kadalai paruppu → chana dal) and whether it is a `main`
  ingredient or a garnish/tempering; a **static TypeScript map of ~150 foods**
  turns names + method words into flags: `fried`, `chilli: 0–3`,
  `seeds_skins`, `raw_veg`, `insoluble_fibre: lo/med/hi`, `dairy`, `lactose`,
  `sprouts_raw`, `egg_undercooked`, `unpasteurised`, `grapefruit`.
- **Nutrient-source flags** (same map, `main` ingredients only, so a
  coriander garnish is not "a source of iron"): `protein`, `iron`, `b12`,
  `vit_d`, `calcium`. Plus one **protein ≈ g/serving** estimate marked "≈",
  and `main_protein` (`dal | legume | egg | paneer | curd | chicken | fish |
  mutton | soy | none`) for rotation.
- **Rules** live in the profile and are edited by chat:
  `phase: remission | flare | recovering`; per-phase filters (e.g. flare:
  `chilli ≤ 1, no fried, insoluble_fibre = lo, no seeds_skins, no raw_veg`);
  hard avoids; learned triggers ("rajma → bloating"). Low-residue applies in
  flares (and strictures), **not** by default in remission — your call (§12).
- **Upadacitinib defaults** (flagged, never forbidden; confirm with your
  doctor): **grapefruit / grapefruit juice** = hard avoid (patient leaflets;
  CYP3A4). JAK inhibitors raise infection risk, so **food-safety flags**
  (raw sprouts, undercooked egg, unpasteurised milk/paneer, street-food raw
  chutneys) are shown on every match — many "healthy" reels are exactly
  sprouted-moong salads.
- **Ranking for "tasty"**: tried-and-rated beats untried; untried
  (`want_to_try`) gets a novelty slot so the backlog actually gets cooked.
- **Guardrail** (in MCP server instructions): flags, not prescriptions; no
  supplement doses; the doctor overrides everything.

## 4. Architecture

```
 Android share (GET, text+url) ─► /share ─┐        ┌─ Telegram bot (files/videos) ─┐
 Claude / ChatGPT ─► /mcp (OAuth) ────────┤        │                               │
 PWA (/app, owner cookie) ────────────────┼────────┴───────────────────────────────┘
                                          ▼
                  Hono Worker ── D1 (Drizzle) ── R2 (thumbs, uploads, backups)
                      │
                      ├─ Workflow "ingest" (one run per capture; retried steps)
                      │    classify → fetch source → (Gemma | Gemini) → validate (Zod)
                      │    → flags (rules) → dedupe → save → ntfy (failures only)
                      │    (backlog: same Workflow, fanned out with createBatch ≤100/call, paced)
                      ├─ AI Gateway → Workers AI (Gemma 4) / Gemini
                      └─ Cron: nightly R2 backup
```

### 4.1 Who structures a recipe

| Path | Structurer | Why |
| :-- | :-- | :-- |
| In a chat (Claude) | **Claude.** `fetch_source(url)` returns raw caption / description / transcript + metadata; Claude writes the recipe via `kitchen_write`. | Free (your subscription), better, no polling inside a tool call. |
| Share sheet / Telegram (no chat open) | **Gemma 4 26B A4B** (Workers AI), thinking off, **one call** doing extraction + ingredient normalisation + method words. | Cheap (~$0.10/$0.30 per M tokens). |
| Video with no usable caption | **Gemini** (Flash tier), async: YouTube URL directly, or uploaded MP4. Reuse `autogram/.../lib/video-analysis.ts`. | Sees and hears the video. |
| Backlog (hundreds) | **Gemma, through the same `ingest` Workflow**, fanned out with Workflows **`createBatch`** (≤ 100 instances per call) and paced with `step.sleep`. | ~500 recipes ≈ 1M tokens ≈ $0.10–0.40; fetching is the slow part, not the model. One pipeline, not two. |

One **Zod schema** is the contract for all four; `kitchen_write` validates
everything the same way. Gemma 4 is **not** on Cloudflare's JSON-mode list,
so: Zod-validate → one repair retry → fall back to Gemini `responseSchema`.

### 4.2 Source adapters — decided by a day-1 test

| Source | Plan A | Plan B | Plan C |
| :-- | :-- | :-- | :-- |
| Blog | `fetch` + `HTMLRewriter` → JSON-LD `Recipe` (no AI) | Browser Run `/markdown` → Gemma | `needs_input` |
| YouTube / Shorts | Data API v3 description → Gemma if it has a recipe | Gemini on the YouTube URL | `needs_input` |
| Instagram | Browser Run render of `/reel/<code>/embed/captioned/` → caption + thumb | Screenshot of caption (share → Telegram/PWA) → Gemma vision | Downloaded video → Telegram → Gemini |
| Screenshot | Gemma 4 vision | Gemini | `needs_input` |
| Text | Gemma 4 | — | — |

- **Day-1 spike (before anything else):** 20 real reels from your feed through
  Browser Run from a deployed Worker. ≥ 70% captions → Plan A. Otherwise the
  screenshot path *is* the Instagram path, and the share flow is optimised for
  it. A Containers + yt-dlp adapter stays in reserve, not in the plan.
- **A share is never lost:** `/share` writes a `captures` row before any fetch.
- **Auto-ready, quiet nights:** a result that passes Zod with ≥ 3 ingredients
  and ≥ 1 step is saved `ready` (badge "auto-imported" until first cooked; raw
  caption kept beside it). The Inbox holds only `needs_input`/`failed`. Push
  **only** on `needs_input`/failure; successes show up as "3 new since
  yesterday" in the `kitchen` brief.
- **Screenshot + link:** a file shared within 10 minutes of a link share joins
  that link's capture (on Telegram: the previous message), so the reel URL is
  kept.
- **`needs_input`** ends the run with a reason. Adding a screenshot/text to the
  capture in the Inbox starts a fresh run (no `waitForEvent`: it times out
  after 24 h by default and kills the instance).
- **Dedupe:** canonical URL (strip `igsh`, `si`, `utm_*`; `youtu.be` / shorts →
  video id; IG → shortcode) with a unique index. A re-share answers
  "already saved: <title>".
- **Language:** captions and audio are often Tamil/Tanglish. Keep the original
  text + `lang`; the recipe is English with the dish's Tamil name.

### 4.3 Stack

| Layer | Choice |
| :-- | :-- |
| HTTP | Hono 4 |
| DB | D1 + Drizzle + drizzle-kit migrations; `db.batch()` for atomic recipe + ingredients + source writes |
| Validation | Zod v4 — tool args, model output, API bodies |
| Search | D1 FTS5 (title, ingredients, tags, caption). No Vectorize / AI Search in v1. |
| Jobs | One Workflow, `ingest`; backlog fans out into it with `createBatch` |
| AI | Workers AI Gemma 4 + Gemini (paid AI Studio key), via AI Gateway with a spend cap |
| Pages | Browser Run binding (10 browser-hours/month included on Paid) |
| Files / push | R2 / ntfy (as lifeD) |
| UI | Hono JSX server-rendered + a little client JS; no build step (lifeD's pattern) |
| Tooling | TS, oxlint, prettier, vitest + `@cloudflare/vitest-pool-workers` |

### 4.4 MCP server

- **Protocol:** MCP 2026-07-28, stateless, legacy-2025 compatible.
  `@modelcontextprotocol/server` **2.4.0** (stable; ships a `validators/cf-worker`
  export) via `createMcpHandler(factory)` from **`agents/mcp/server`**
  (`agents` 0.32.0). The handler does no auth.
  *P0 hour 1 check:* deploy one tool, connect from Claude Android. If the SDK
  path misbehaves, port lifeD's `src/mcp.ts` (known-good with every client you use).
- **Auth:** lifeD's `src/auth/*` in front (OAuth 2.1, CIMD + stateless DCR,
  owner passphrase, one D1 grant row, `ADMIN_TOKEN` bearer for scripts).
- **Five tools, names that don't clash with lifeD's `context/find/get/write`**
  (both connectors will be on in the same chat):

| Tool | What |
| :-- | :-- |
| `kitchen` | `{slot?}` → brief ≤ ~500 tokens: phase + active rules, avoids, `have` list, titles eaten last 7 days, "N new since yesterday", items needing you — **plus 3 ranked options** (§4.5), each with a why, the video link and the Annam page link. Call first. |
| `recipes` | Filtered one-liners: `q, role, cuisine, max_min, have, not_eaten_days, status (tried/want), flags…` |
| `recipe` | Full recipe(s) by id: ingredients, steps, sources (video links), variants, your log of it. |
| `kitchen_write` | Many ops, one call, `$i` refs (lifeD style): `recipe.upsert` (variants via `parent`), `meal.log`, `have.set/add/remove`, `profile.set`, `rule.set`, `capture.resolve`. |
| `fetch_source` | `url` → raw caption/description/transcript + metadata (or "already saved"). If only video can answer, starts Gemini async, returns capture id; push when ready. |

- **Server instructions:** call `kitchen` first; respect phase rules and
  avoids; show 3 options with a why; link the source video **and** the Annam
  page (`/app/r/<id>`); save adaptations as variants; log meals when told
  (infer slot from time of day); if `have` is > 3 days old, ask "still have
  eggs, rice, spinach?" before ranking; if lifeD shows GI symptoms in the last
  3 days while `phase = remission`, ask before ranking; flags not advice;
  doctor overrides.

### 4.5 `kitchen` ranking (v1 formula — tune on a week of real logs)

**Slot → roles:** breakfast → `breakfast`; lunch/dinner → `main` or
`base + gravy/protein + veg`; snack → `snack`.

**Filter:** `status = ready` · slot roles · hard avoids · current phase rules ·
equipment you have · `total_min ≤ max` if asked.

**Score** (each term 0–1):

```
score = 3.0 · cov      # non-staple, non-optional ingredients found in `have`
                       #   (0.5 flat if `have` is stale, > 3 days)
      + 2.0 · gap      # min(days since last eaten, 14) / 14; never eaten = 1
      + 1.5 · taste    # (rating − 3) / 2 if tried; 0 if untried
      + 1.0 · nut      # 0.5 per nutrient-source flag matching a deficiency, max 1
      − 1.0 if main_protein or cuisine matches a meal in the last 48 h
      − 2.0 per learned trigger present
      − 1.0 if ₹/serving > 1.5 × per-meal budget
```

**Pick:** top 2 *tried* recipes (never the same `main_protein`) + 1 slot for
the best *want-to-try*. Ties broken by a date-seeded shuffle, so asking twice
in a day gives the same answer. `have: true` in `recipes` = cov ≥ 0.7.

**Why line:** the two biggest score terms in words + up to 3 missing items.
("In stock, not had in 9 days · missing: curry leaves")

## 5. Data model (v1)

- `captures` — id, at, via (`share|telegram|mcp|backlog|app`), url,
  canonical, text, files (R2), status (`queued|working|ready|needs_input|failed`),
  reason, attempts, recipe_id.
- `recipes` — id, slug, title, title_ta, summary, cuisine, roles[], servings,
  total_min, status (`draft|ready|archived`), cooked (`want|tried`), rating,
  auto_imported, parent_id + variant_note, notes, steps (json; optional timer
  per step), main_protein, protein_g≈, inr_per_serving≈, flags (json,
  derived), nutrient_sources[], method[], created/updated.
- `ingredients` — recipe_id, pos, section, raw, name (canonical English),
  qty, unit, main (bool), optional.
- `sources` — recipe_id, platform, canonical_url (**unique**), url, author,
  title, caption, transcript, lang, thumb_r2, fetched_at. Many per recipe.
- `meals` — at, slot, recipe_id?, text, main_protein, cuisine, portion,
  rating, felt, note. (Text-only meals like "had biryani" still feed variety.)
  **Annam owns meals** (variety needs `recipe_id`); symptoms/meds stay in lifeD.
  Update the Food & Health assistant's rule to log meals here. Logging starts in P0.
- `profile` — one JSON row: phase, rules per phase, avoids, triggers,
  deficiencies, targets, `have[]` (names only) + `have_updated_at`, staples,
  equipment, budget (per meal), cuisine, spice, diet.
- `tags`, `recipe_tags`, `settings`, FTS5 table.

**Deferred:** `foods` table + aliases, price table, pantry quantities/use-by,
meal plan, shopping list, kcal/macros, IFCT.

## 6. Android

- **v1 share = PWA `share_target` GET** (`title, text, url`) — proven by lifeD on
  your phone. Instagram and YouTube share text + URL, which is all GET needs.
  Owner cookie lifetime **1 year** (single user), with "sign out everywhere"
  in Settings.
- **Files (screenshots, videos):** day-1 test decides between
  (a) PWA `share_target` POST handled by a **service worker** (store in
  IndexedDB → 303 to a GET page → same-origin upload; never hits the login
  wall), or (b) **Telegram bot**: share to the bot, no cookie at all. lifeD's
  `telegram.ts` handles text only, so photo/document/video + `getFile` is new
  work, and the Bot API caps downloads at **20 MB** (short clips and
  screenshots only). **Build one, not both** — whichever wins the day-1 test.
- The share page answers in < 1 s ("Saved ✓ — reading it"). Returning to the
  source app automatically is not promised; Back works.
- Sharing straight into the Claude Android app: nice if it works, not relied on.
- **Native app: not in v1.** Later only for a home-screen "What should I eat?"
  widget or offline cooking mode; TWA wrapper first.

## 7. The app (PWA, 4 screens)

1. **Share** — the landing page for shares; shows status.
2. **Inbox** — only what needs you: `needs_input` / `failed` (add screenshot / paste).
3. **Recipe** — thumbnail + "open video", ingredients (✓ in `have`), steps with
   timers, flags/badges, variants. Screen stays awake while open (Wake Lock).
   **"Cooked it"** logs a meal (slot from time of day), sets `tried`, and shows
   1-tap stars right there — ratings are what "tasty" ranks on.
4. **Library** — search + filters (role, time, tried/want, flare-friendly now).

"Good app" = fast on a phone, thumbnail-first cards, one-thumb actions, no
spinners. "Today", "Mix", pantry and planning are Claude's job in v1.

## 8. Cost

Workers Paid $5/mo covers Workers, D1, R2, Workflows. Gemma ≈ cents/month.
Browser Run: ~100 renders/month ≈ 17 min; a 500-reel backlog ≈ 1.5 h — inside
the 10 h included. Gemini (video fallback only) < $1/month. **≈ $5–6/month.**

## 9. Reuse map

| From | Take |
| :-- | :-- |
| lifeD | `auth/*`, `owner.ts` (cookie, extend to 1 y), `write` ops with `$i`, `instructions` builder, ntfy (`remind.ts`), R2 backup (`archive.ts`), `telegram.ts`, manifest + share route, page layout, tooling |
| autogram | Gemma call + neuron accounting (`ai-writer.ts`), Gemini upload/poll + `responseSchema` (`video-analysis.ts`), spend cap (`ai-budget.ts`), Drizzle/migration discipline, MCP arg validation (`mcp-args.ts`), mutation-proven tests |
| Mealie | schema.org Recipe fields; prompts `parse-recipe-video`, `parse-recipe-ingredients`, `build-recipe` as starting text; qty/unit/name split; step timers (v3.30); editable AI prompts (store prompts in `settings`) |

## 10. Phases and "done" checks

**Day 1 — spikes (half a day).** (1) 20 reels through Browser Run from a
deployed Worker. (2) On your phone: PWA install, GET share, SW-POST screenshot
share vs Telegram. (3) One-tool MCP via SDK v2 connected from Claude Android.
*Output:* the Instagram plan, the file-share plan and the MCP path, written into §4.2/§6.

**P0 — talkable (1–2 days).** D1 schema; auth; 5 MCP tools; profile + rules;
meal logging. Recipes enter by Claude (`fetch_source` for blogs/YouTube
descriptions, paste for the rest).
*Done when:* on the phone, "save this: <YouTube link>", "had dosa for
dinner", and "what can I make with eggs and rice?" all work.

**P1a — share sheet (2–3 days).** PWA + GET share; captures + Inbox; ingest
Workflow: blog JSON-LD, YouTube description, Instagram per the spike; Gemma
extraction + Zod; flags; dedupe; ntfy; Recipe + Library screens.
*Done when:* 10 real link shares → ≥ 8 usable recipes with no typing;
failures say why; a re-share says "already saved".

**P1b — files and video (2 days).** Screenshots (SW-POST or Telegram), Gemma
vision, Gemini video path.

**P2 — backlog + decide-for-me (week 2).**
- *Instagram:* "Download your information" gives **links only** (no captions;
  check field names on a real export, per account). So captions are fetched
  **once, by a local script on your home connection** (residential IPs are
  blocked far less than Cloudflare's): logged-out, slow (~1 reel / 20 s),
  posting caption + thumbnail URL to `/api/backlog` with `ADMIN_TOKEN`.
  Independent of the day-1 result; no Containers needed.
- *YouTube:* make the recipe playlists unlisted for a day and read them with an
  API key (`playlistItems.list`), no Google login. Watch Later is invisible to
  the Data API since 2016 → Takeout CSV (verify).
- All items → the same `ingest` Workflow via `createBatch`, paced; land as
  `want`. Ranking in `kitchen` tuned on a week of real logs.
*Done when:* every saved reel across accounts is searchable in one place, and
"what should I eat today?" gives 3 options you'd cook, none eaten in 4 days,
phase rules respected.

**P3 — optional.** MCP Apps recipe cards inside Claude; pantry quantities +
shopping list; weekly plan; IFCT nutrition; Vectorize "similar"; symptom ↔
meal correlation job with lifeD; Containers + yt-dlp; TWA/native widget.

## 11. Risks

| Risk | Mitigation |
| :-- | :-- |
| Instagram blocks Browser Run | Day-1 spike decides; screenshot path is first-class; never lose the share |
| Backlog script trips Instagram | Logged-out, one reel per ~20 s, one-time run, resumable; never with your main account's session |
| Share POST hits the login wall | GET for links; SW-POST or Telegram for files; 1-year cookie |
| Model invents quantities | "Null if unknown"; raw caption/transcript kept beside the recipe; Zod + repair; drafts go to Inbox |
| Health overreach | Facts stored, rules decide; flags not advice; doctor overrides |
| Two connectors confuse the model | Distinct tool names; short descriptions |
| Copyright | Private app; stores link, caption, transcript, private thumbnail; never republishes |

## 12. Questions only you can answer

1. Veg / egg / non-veg? Foods your doctor told you to avoid?
2. **Low-residue always, or only in flares?** (Confirm with your doctor; also
   the grapefruit and food-safety defaults.)
3. Kitchen: burners, pressure cooker, mixie, air fryer/oven, fridge? Who cooks?
4. Protein target? Weight goal? Is ₹5k/month all food or groceries only?
5. How many Instagram accounts / YouTube accounts hold saved recipes?
6. Tamil dish names alongside English? Name "Annam" OK?

## 13. Review log

**Round 1 (accepted):** day-1 spikes before building (IG via Browser Run,
Android share + cookie); GET-first share, files via SW-POST or Telegram;
tool names distinct from lifeD; `suggest` folded into `kitchen`, 5 tools;
Claude structures in chat (`fetch_source` returns raw text), Gemma only for
share/backlog; **backlog import added**; factual flags + query-time rules
instead of a stored `flare_ok`; Upadacitinib grapefruit + food-safety
defaults; Annam owns meals, logging from P0; v1 data model cut; canonical-URL
dedupe, `lang`, `want/tried`, `attempts`; no `waitForEvent`; Gemma JSON via
Zod + repair + Gemini fallback; SDK facts corrected (`agents/mcp/server`,
SDK 2.4.0); 4 server-rendered screens; P1 split; Browser Run cost checked.

**Round 1 (adjusted):** nutrition kept as protein ≈ + rule-based
nutrient-source flags (B12/iron/vit D matter for you) rather than protein only;
₹/serving kept as a model estimate; UI scope cut but "good app" kept as a
design bar; Telegram added as a file-share option.

**Round 2 (accepted):** Instagram backlog = links only → one-time local
caption fetch on your home IP; Workers AI async batch dropped, backlog goes
through `ingest` via `createBatch`; Telegram files are new work (20 MB cap),
build one file path not two; YouTube via unlisted playlists + API key, Takeout
for Watch Later; `main_protein`, slot→roles, `have_updated_at` and a written
ranking formula (§4.5); canonical names + static food map + `main`-only
nutrient flags; P0 check uses a YouTube link; auto-ready + Inbox for problems
only + push only on failure; screenshot joins its link's capture; stale-`have`
and phase-drift questions; Annam page link on every option; Wake Lock and
1-tap stars after "Cooked it".
