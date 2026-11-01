# Solo Leveling — the level-up layer of lifeD

Plan **v8, final** (reviewed in 6 rounds, 2026-11-01). Plan only; no code yet. Self-contained: everything a builder
needs is in this file. lifeD records: goal **#470** (the Daily Quest), project **#471** (this build).

---

## Start here (for the building agent)

**Read, in order:** `AGENTS.md` → `docs/STATUS.md` → `docs/DESIGN.md` → this file. Then build **one phase at a time**
from §9, in order, and stop at each phase's "done when" for the owner.

**The goal is leveling Omesh up** — food, training, mind and body — not building a platform. If a task doesn't move a
phase's "done when", don't do it. Don't pivot the design; if something in this plan is wrong, say so and propose the
smallest fix.

**How to work:**

- **Think first, keep it simple, change surgically.** State the plan for a phase before coding. Write the least code
  that meets "done when". Touch only what the phase needs; no drive-by refactors. Verify every step (tests, `npm run
check`, smoke), then say what you verified.
- **Modular.** New domain code lives only in `src/level/` (one file per concern: `quest.ts`, `rank.ts`, `ranker.ts`,
  `source.ts`, `settings.ts`, `xp.ts`, …). It talks to the rest of lifeD only through the existing `write`/read paths
  and the MCP tool registry. Core files get the smallest possible hook.
- **Edit with the editor tools** (Read/Edit/Write). Use the shell to run checks, migrations, tests and git — not to
  edit files with sed, heredocs or Python.
- **Follow `AGENTS.md` exactly:** tokens are the product (measure with `scripts/smoke.mjs`), nothing is deleted, OAuth
  code is not "simplified", `npm run check` before every commit, `docs/STATUS.md` kept current with dated decisions.
- **Another agent writes to lifeD live** (Omesh's Grok bot: Food & Health, reminders, logging). Every change is
  **additive**: never rename or remove a tool, kind, form, status or rule it may use; never change an existing tool's
  arguments or output shape; seed data idempotently.
- **Deploy** only after the §"Checks before deploy" list in `AGENTS.md` passes **and** Omesh says go. Record each
  deploy in `STATUS.md` with its Worker version and rollback point.

---

## 0. Decisions

1. **One system, not two.** Solo Leveling lives inside lifeD. Meals, workouts, check-ins, symptoms and work sit on
   one timeline behind one connector, one Inbox, one share target, one backup.
2. **`main` is the live code.** Fast-forwarded on 2026-11-01 to the deployed branch (`c110435`, Worker `db29e580`,
   5 tools: `context, find, get, spend, write`, _formats as data_).
3. **Use formats as data (shipped 2026-10-17).** A `kind` record can carry `props.schema` (enum fields, enforced by
   `write`); a `form` record (title = a log type) fixes that log's `required/defaults/numbers`. So `meal`,
   `workout`, `checkin`, `quest`, `setting` are `form` records and `recipe`, `exercise`, `routine` are `kind`
   records. **Logging needs no new code.**
4. **A declared domain module.** A recipe ranker, a source fetcher, the quest engine and XP are domain logic. They
   live in `src/level/` as a **dated, documented exception** to "no domain logic in code" (add the decision row to
   `STATUS.md` and a note in `DESIGN.md` in the first code commit), with TypeScript constants and tests.
5. **No server-side LLM in L1** (lifeD's founding rule). A share is saved raw; Claude structures it in the next chat.
   Server-side AI (Workers AI Gemma 4) for the 500-reel backlog is re-decided with Omesh in L2.
6. **Everything is changeable by talking.** `src/level/` holds Zod schemas, defaults and tests. A change is a
   **`setting` log event** (`form: setting`, required `key`, `value`): "make push-ups 80" is one `write`. Latest event
   per key wins; its timestamp gives history and "applies from today". Overrides are validated **on read**; an
   invalid one falls back to the default and the next push / `suggest` line says which setting was ignored and why.
   Core `write` is untouched. **Tunable keys:** quest targets, progression %, deload, run-walk ratio, XP weights, stat
   sources, ranker weights, phase rules, avoids/triggers, `have`, quiet hours, side-quest preferences.

## 1. Name and theme

**Solo Leveling** — lifeD is "the System", Omesh is the player: a Daily Quest, anime stats (STR, AGI, VIT, INT, SEN),
ranks E→S, level-ups. Private app; rename before anything public (the name is someone else's trademark).

## 2. What lives where

| Thing                     | lifeD shape                                                                                                                                                                                                                                                                                                              |
| :------------------------ | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Recipe                    | `kind: recipe`. **Small `props`:** ingredient names (singular, canonical English), roles, cuisine, main_protein, total_min, servings, equipment, flags, nutrient_sources, inr≈, protein_g≈, url, platform. **`body`:** caption/transcript, full ingredient lines, steps with timers, notes. Variant = `rel: variant_of`. |
| Rating, tried, last eaten | **Derived from `meal` events** at query time; never stored on the recipe (no re-index, no stale card on "Cooked it").                                                                                                                                                                                                    |
| Exercise / routine        | `kind: exercise` (muscles, equipment, cues, url) / `kind: routine` (exercise list in body, url).                                                                                                                                                                                                                         |
| Shared link waiting       | `kind: inbox` (exists) · props: url, platform, status, reason · raw text in body.                                                                                                                                                                                                                                        |
| Meal                      | `form: meal` → slot, portion, rating, felt · refs the recipe.                                                                                                                                                                                                                                                            |
| Workout                   | `form: workout` → `refs: ["bench press","overhead press"]` (exact titles) + `data.ex: [{ex:"bench press", sets:[[8,40],[8,40],[6,42.5]]}]`, mode, min, rpe, energy.                                                                                                                                                      |
| Quest                     | `form: quest` → `numbers: push, core, squat, run_km, walk_km`. Several a day; **the day's total = sum of that day's quest events.**                                                                                                                                                                                      |
| Check-in                  | `form: checkin` → mood 1–5, energy 1–5, sleep_h, stress 1–5, gut 0–5, pain 0–3.                                                                                                                                                                                                                                          |
| Setting                   | `form: setting` → key, value (Decision 6).                                                                                                                                                                                                                                                                               |
| Weight, symptoms, meds    | As today (metric events; the Grok bot's Food & Health logging).                                                                                                                                                                                                                                                          |
| URL dedupe                | One small system table `urls(canonical PK, ent)` — a mechanic, like `notified`.                                                                                                                                                                                                                                          |
| Defaults                  | Food-flag map (~150 foods), Zod schemas, XP table, quest defaults: **TypeScript constants in `src/level/`, unit-tested**, overridden by `setting` events.                                                                                                                                                                |
| Pinned                    | **One** rule ≤ 220 chars pointing to the unpinned `daily quest` rule and this doc (pinned rules ride in every chat).                                                                                                                                                                                                     |

FTS uses `unicode61` without stemming: store singular canonical names, normalise query words the same way.

**MCP tools: 5 today + 2 = 7.**

| Tool      | What                                                                                                                                                                   |
| :-------- | :--------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `source`  | `url` → raw caption / description / transcript + metadata, screenshots as MCP image content, or "already saved #id". (Not `fetch`: avoids clashing with Claude's own.) |
| `suggest` | `{for: meal\|workout, slot?, treat?, minutes?}` → 3 options, each with a one-line why, the video link and the `/app` link.                                             |

`context()` (no args) gains at most two lines, ≤ 60 tokens together: from L1 `N shares waiting — structure them?`;
from L1b `Quest (C): push 32/50 · core 20/50 · squat 0/70 · 3.1/5 km`; from L3 `Lv 7 · STR 12 AGI 6 VIT 15 INT 20
SEN 9` appended.

**Server instructions** (add to lifeD's instructions builder, short): call `context` first; respect phase rules and
avoids; give 3 options with a why; link the source video and `/app/r/<id>`; save adaptations as variants; log meals,
workouts, quests and check-ins when told (infer meal slot from the time); if `have` is > 3 days old ask "still have
…?" before ranking; if GI symptoms were logged in the last 3 days while phase = remission, ask before ranking; flags
are not medical advice; the doctor overrides everything.

## 3. Food

**Owner facts:** eats veg, egg and non-veg; cooks mostly for one; South Indian kitchen (Tiruppur); food budget
₹5k/month (default **₹55 per meal** until he says otherwise); measures by weighing scale, by tumbler and by eye
(ratios); wants treat meals sometimes; ≤ 2 Instagram accounts; YouTube by sharing URLs.

**Capture (L1).** Android share → `/app/share` (GET, exists; today it pre-fills quick add). L1: a recognised URL is
**saved straight to an `inbox` record** and the page answers in < 1 s ("Saved ✓"). The server fetches raw text only
(plumbing, not interpretation):

| Source          | Raw text from                                                                                                                               | If that fails                                          |
| :-------------- | :------------------------------------------------------------------------------------------------------------------------------------------ | :----------------------------------------------------- |
| Blog            | `fetch` + `HTMLRewriter` → JSON-LD `Recipe`, else page text                                                                                 | Browser Run `/markdown`                                |
| YouTube/Shorts  | Data API v3 `videos.list` description (API key, no login)                                                                                   | inbox `needs_input`                                    |
| Instagram       | Browser Run render of `instagram.com/reel/<code>/embed/captioned/` → caption + thumbnail (plain fetch and oEmbed no longer return captions) | screenshot of the caption (decided by the Day-1 spike) |
| Screenshot/text | stored as-is; `source` returns images as MCP image content                                                                                  | —                                                      |

Next chat: `context` shows "3 shares waiting" → Claude reads each via `source` and writes the recipe via `write`.
**Canonical URL:** strip `igsh`, `si`, `utm_*`; `youtu.be` and `/shorts/` → video id; Instagram → shortcode. A
re-share answers "already saved #id". Captions are often Tamil/Tanglish: keep the original in `body`, write the
recipe in English with the Tamil dish name as an alias.

**Health facts, rules decide.** From lifeD #357: Crohn's on Upadacitinib; low B12, vitamin D, iron.

- Per-recipe **factual flags** from the food-flag map over `main` ingredients + method words: `fried`, `chilli 0–3`,
  `seeds_skins`, `raw_veg`, `insoluble_fibre lo/med/hi`, `dairy`, `lactose`, `sprouts_raw`, `egg_undercooked`,
  `unpasteurised`, `grapefruit`. **Nutrient sources** (main ingredients only): protein, iron, b12, vit_d, calcium.
  `main_protein`: dal | legume | egg | paneer | curd | chicken | fish | mutton | soy | none.
- **Phase rules** (a setting): phase = remission | flare | recovering. **Low-residue only in flares** (chilli ≤ 1,
  no fried, insoluble_fibre lo, no seeds_skins, no raw_veg). Learned triggers ("rajma → bloating") are avoids.
- **Upadacitinib defaults** (flag, never forbid; doctor confirms): grapefruit = hard avoid; food-safety flags (raw
  sprouts, undercooked egg, unpasteurised milk/paneer) shown on every match.

**`suggest {for: meal}` (L2).** Ranks single recipes; combos (rice + gravy + poriyal) are Claude's job via
`find kind:recipe`.

- Slot → roles: breakfast → breakfast; lunch/dinner → main | gravy | protein; snack → snack.
- Filter: ready; slot roles; avoids; current phase rules; recipe equipment ⊆ owner's; `total_min ≤ minutes`.
- Score: `3·cov + 2·gap + 1.5·taste + 1·nut − 1·(main_protein or cuisine eaten in last 48 h) − 2·(per trigger) −
1·(₹ > 1.5 × per-meal budget)`. cov = share of non-staple, non-optional ingredients in `have` (0.5 flat if `have`
  is > 3 days old); gap = min(days since eaten, 14)/14, never eaten = 1; taste = (mean rating − 3)/2 if tried, else 0;
  nut = 0.5 per nutrient source matching a deficiency, max 1.
- **Treat mode** (`treat: true`): `3·taste + 1·gap + 1·cov`; no budget, nutrient or fried penalties. Avoids,
  food safety and flare rules still apply.
- Pick: top 2 tried (different `main_protein`) + best want-to-try; empty tried slots fill from want-to-try;
  date-seeded ties (same answer twice a day). Why line = two biggest terms + up to 3 missing items.
- Two queries: ready recipes via `ent_kind`; last meal per recipe via `event_refs ⨝ events` (60 days).

**Recipe page (L1):** thumbnail + open video, ingredients (✓ in `have`), steps, one timer, **"Cooked it"** → meal
event + 1-tap stars.
**Cooking mode (L2):** Wake Lock; several big timers (minutes or pressure-cooker whistles) with sound + vibration;
servings stepper (default 1; round to measurable amounts; whistles, tempering and "to taste" not scaled);
**Scale / Tumbler / Ratio** toggle (tumbler size set once).
**Backlog (L2):** Instagram "Download your information" gives links only → a one-time, slow (~1 reel / 20 s),
logged-out caption fetch from Omesh's home connection posts to `/api/backlog` with `ADMIN_TOKEN`; YouTube old saves
by sharing a playlist URL (`playlistItems.list`, public/unlisted). Everything lands raw in `inbox`, searchable at once.

## 4. Training

### 4.1 The Daily Quest (goal #470) — starts in week 0

**Goal:** 100 push-ups · 100 core · 100 squats · 10 km. **January milestone:** one **S-rank day** (100/100/100 in any
number of sets + 10 km moved, run + walk) plus a separate **5 km run-walk test**. The S-rank day **unlocks only after
every rep target has been ≥ 75 for 2 qualifying weeks**; otherwise the milestone moves to February and the push says
so, without blame. Daily S and a 10 km run-walk are the Feb–Mar goals. Trains at gym and home; the quest needs no
equipment.

**Week 0** (after the doctor's OK, ideally after the iron injection #438; Nov 2–8 if both are done): baseline on day
1 and day 7 — max push-ups, max core reps, max squats (2 min each), an easy timed 2 km run-walk. Light practice, no
progression.

**Targets grow from his baseline:**

- Reps start at **1.5 × baseline max**, spread through the day; each qualifying week **+10 % (min +5)**, cap 100.
- Distance moved starts at **1.2 × current daily km**, +10 % per qualifying week, to 10 km. Running: run-walk
  intervals, run minutes +10 %/week, at most one longer day a week, no running on symptomatic days.
- **Deload** every 4th calendar week from week 0: targets × 0.7, then resume from the pre-deload targets.
- **Core 100** = any mix of crunches, dead bugs, leg raises, planks (10 s = 1 rep).
- A `setting` like `quest.push = 80` becomes the current target and progression continues; `quest.push.hold = true`
  pins it.

**Bands** from the **lowest** of the three rep targets: E < 30 · D 30–49 · C 50–69 · B 70–84 · A 85–99 · **S = 100 on
all three + 10 km**. **Title = highest band ever reached**; announced once ("[SYSTEM] Rank up: D → C").

**Qualifying week** = ≥ 4 real cleared days (summed quest events meet all four targets) and no day with energy ≤ 2
or pain ≥ 2. Missing check-ins don't block.

- Recovery and rest days protect streak and XP but **never** count toward progression.
- **Re-awakening:** after ≥ 4 consecutive days without a cleared quest, targets × 0.7 for 7 days, then resume.
- **Low-mood shrink:** the push shows one quest item, at Re-awakening targets.

**Body first:** doctor's OK before week 0; tell the doctor about the training before blood tests (high-rep work raises
CK, and Upadacitinib can too). Stop on abdominal pain, dizziness or dark urine. Flare day = **Recovery Quest** (rest,
meds, fluids, soft food, log symptoms).

**Day 1, before any code:** seed an unpinned rule **`daily quest`** stating all of §4.1 in words, so Claude can run the
quest by hand until L1b. At L1b it changes to "computed by the System; change it with a `setting`".

### 4.2 Gym and strength

- Log by chat: "push day: bench 3×8 @ 40, OHP 3×10 @ 20" → one `write` → `workout` event with refs per exercise.
- Workout reels → `inbox` → Claude makes `routine` / `exercise` records with the video linked.
- `suggest {for: workout}` (L4): days since each muscle group, today's energy, phase (flare → rest or gentle
  mobility), minutes, equipment (gym or home); next load = last + small step when last RPE ≤ 8. PRs = max over recent
  events per exercise.

## 5. Mind & body

- **Check-in** in 10 seconds by chat or `/app` tiles: "mood 3, slept 6h, gut ok, pain 0".
- **Sunday review** (exists) adds observations — sleep vs mood, meals vs gut, training vs energy ("bloating followed
  rajma 2 of 3 times") — never a diagnosis.

## 6. The System (L3 — after ≥ 2 weeks of real logs)

- **XP for showing up, not numbers:** logging a meal, check-in, workout, quest or journal line earns XP. Mood/energy
  values never change XP; no XP for weight; no intake-amount quests ("log your meals", never "eat 80 g protein").
- **Stats:** STR (quest push-ups/squats, strength, PRs) · AGI (quest distance, cardio, mobility) · VIT (core reps,
  meals, sleep logged, meds taken) · SEN (check-ins, journal) · INT (lifeD tasks done, learning; "stop counting work"
  turns it off).
- **Never goes down:** nightly rollup of XP per day into `settings` (rebuildable); today computed live; level =
  max(high-water mark, computed); XP-table changes apply from today.
- **Side quests** in the 08:00 push: 2–3 small logging items, separate from the fitness Daily Quest.
- **Guardrails (live from L1b):** flare = Recovery Quest; low mood — mood ≤ 2 on 3 of the last 5 check-ins → one
  quest item, and the System says **once**: talk to <person Omesh names> or Tele-MANAS (free, 24×7, India: **14416**);
  quiet — "pause the System for a week", no pushes after 21:00. Flags, not medical advice.

## 7. Android

lifeD's installed PWA is the one app. Share target: GET `/app/share` (exists). Files (screenshots/videos): a
service-worker POST share target (store in IndexedDB → 303 → same-origin upload) **or** Telegram files (new work; Bot
API caps downloads at 20 MB) — **build one**, chosen by the Day-1 spike. Owner cookie 30 days → 1 year with "sign out
everywhere". New screens: Recipe (L1), Cooking mode (L2), Status (L3). Native Android only later, for Health Connect
or a widget.

## 8. Changes to lifeD itself

- The account is on Workers Paid (autogram's config). Fix the Free-plan notes in `DESIGN.md`, `STATUS.md`,
  `wrangler.jsonc` and `AGENTS.md` (D1/CPU); keep "frugal".
- Dated decision rows: the `src/level/` exception (first code commit); later, any server-side-AI decision.
- New bindings over time: Browser Run (L1), Workflows (L2 backlog, `createBatch`), Workers AI + AI Gateway only if
  Omesh allows server-side AI.

## 9. Phases — one at a time, stop at "done when"

**Step zero (½ day).** `main` = live code (done 2026-11-01). Confirm migrations `0008` and `0009` are applied to the
remote D1 (STATUS said 0008 was pending). Paid-plan doc fixes.

**Day 1 — seed + spikes.**

- Seed once, idempotently: kinds `recipe`, `exercise`, `routine` (with `props.schema`); forms `meal`, `workout`,
  `checkin`, `quest`, `setting`; the unpinned `daily quest` rule; the one pinned pointer rule. Measure `context`
  tokens before/after with `scripts/smoke.mjs`.
- Spike 1: 20 reels (Omesh's list) through the Browser Run **Workers binding** from the deployed Worker. Pass = ≥ 14
  return the full caption with the ingredient list → Instagram via Browser Run; else screenshots are primary.
- Spike 2 (on Omesh's phone): link share lands in `inbox` without a login prompt 5/5; screenshot path (SW-POST vs
  Telegram) — pick one.
- _Done when:_ Claude can log a quest, a meal, a workout and a check-in by chat, and tell today's quest targets
  from the `daily quest` rule.

**L1 — capture (~2 days).** Share URL → `inbox` + raw fetch; `urls` dedupe; `source` tool; Recipe page with "Cooked
it". _Done when:_ 10 real shares are saved raw and Claude turns ≥ 8 into recipes in one chat.

**L1b — the quest runs itself (~1 day; before the first progression Monday, Nov 16 if week 0 is Nov 2–8).** Today's
targets + title in the 08:00 push and `context()`; progression, deload, Re-awakening; rank-up announcement;
guardrails. _Done when:_ a week of logs moves targets correctly, a flare day gets a Recovery Quest, nothing ever
goes down.

**L2 — decide + backlog (week 2–3).** `suggest {for: meal}` + treat mode; cooking mode; backlog import; server-side-AI
decision with Omesh. _Done when:_ "what should I eat?" gives 3 dishes he'd cook, none eaten in 4 days, phase rules
respected; every saved reel is searchable.

**L3 — The System (after ≥ 2 weeks of logs).** XP rollup, stats, level, side quests, Status screen.

**L4 — Training.** `suggest {for: workout}`, PRs, routines from reels.

**L5 — Mind & body.** Weekly-review observations; optional Health Connect companion.

## 10. Risks

| Risk                                  | Mitigation                                                                                           |
| :------------------------------------ | :--------------------------------------------------------------------------------------------------- |
| The new layer damages the life memory | Writes only through `write`; `src/level/` isolated; additive changes; nightly R2 backup; tests       |
| The Grok bot breaks on a change       | No renames/removals; tool contracts unchanged; idempotent seeds                                      |
| Every chat pays for it                | 2 new tools; one ≤ 220-char pinned rule; ≤ 60-token context lines; measured                          |
| Gamification becomes pressure         | XP for logging not values; no intake targets; title never down; flare protects; low-mood step; quiet |
| Training too fast                     | Baseline-based targets; +10 %/week; deload; Re-awakening; S-rank gate; doctor's OK; CK caveat        |
| Instagram blocks Browser Run          | Day-1 spike; screenshot path; home-connection backlog                                                |

## 11. Owner inputs (Omesh)

Doctor's OK for the quest and the date of the iron injection (#438) · week-0 baseline (day 1, day 7) · 20 reel links
with recipes in the caption · YouTube Data API key · lifeD's ntfy topic.

## 12. Open questions (all have defaults; change any time by chat)

1. Foods or exercises the doctor said to avoid? OK with the grapefruit and food-safety defaults?
2. Kitchen kit (pressure cooker, mixie, air fryer, scale), tumbler size, gym equipment used?
3. Protein target, weight goal; is ₹5k/month all food or groceries only?
4. Who should the System suggest he talks to on a low stretch?

## 13. History

Planned 2026-11-01 in a session on `omesh7/mealie` (branch `ccr-a0cc12c7-3urwd5`, `plan/`): v1–v3 a separate recipe
app ("Annam"), v4 merged into lifeD, v5–v8 after review rounds 4–6. Mealie itself was rejected as a host (Python +
local disk, no MCP, multi-tenant); its schema.org Recipe fields and prompts were the reference for the recipe shape.
