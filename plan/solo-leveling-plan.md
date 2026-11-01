# Solo Leveling — level-up layer of lifeD (plan v6)

Status: **plan only, no code.** v6 = v5 + owner answers (gym + home, anime
stats, the Daily Quest goal) + "everything changeable by chat". History: §13.
lifeD records: goal **#470** (the Daily Quest), project **#471** (this build).

---

## 0. Decisions

1. **One system, not two.** Solo Leveling lives inside lifeD. Meals, workouts,
   check-ins, symptoms and work sit on one timeline behind one connector,
   one Inbox, one share target, one backup. Two apps would mean two connectors
   in every chat, a contract over who owns meals vs symptoms, and hand-made
   cross-database analysis.
2. **Build on the live branch, not `main`.** lifeD's deployed code (Worker
   `db29e580`, 5 tools incl. `spend`, *formats as data*) is on branch
   `ccr-7a2881ef-38kr1m` (2026-10-17). `main` stops at `ea83e74` (2026-09-28).
   **Step zero: merge that branch into `main`.**
3. **Use lifeD's "formats as data" (already shipped):** a `kind` record can
   carry `props.schema`, enforced by `write`; a `form` record (title = a log
   type) fixes the shape of that log's data. So `meal`, `workout`, `checkin`
   are just `form` records, and `recipe`, `exercise`, `routine` are just `kind`
   records. **Logging needs no new code.**
4. **Declare the domain module honestly.** lifeD says "no domain logic in
   code". A recipe ranker, an importer and an XP engine *are* domain logic.
   They live in `src/level/` as a **dated, documented exception** (STATUS +
   DESIGN), with TypeScript constants and tests — not disguised as "generic
   engines" with schemas hidden in rule records.
5. **No server-side LLM in L1** (lifeD's own rule: "the AI understands, the
   server structures"; you said yes to keeping it for now). A share is saved
   raw; Claude structures it in your next chat. Gemma for the 500-reel
   backlog is re-decided in L2.
6. **Everything is changeable by talking.** Code holds the *shape* (Zod
   schema), the *defaults* and the tests. Every number you might want to change
   — quest ladder, rank-up rule, XP weights, stat sources, ranker weights,
   phase rules, avoids, `have`, quiet hours — lives in an unpinned record that
   overrides the default. "Make the quest 80 push-ups" or "count cycling as
   AGI" is one `write`; an edit that doesn't fit the schema is refused with a
   message that says why, so nothing breaks silently.

## 1. Name and theme

**Solo Leveling** — lifeD is "the System", you are the player: daily quests,
stats from real events, level-ups. Private, so the name is fine; rename it
before anything public (someone else's trademark).

## 2. What lives where

| Thing | lifeD shape | New code? |
| :-- | :-- | :-- |
| Recipe | `kind: recipe` record. **Small `props`:** ingredient names (singular, canonical English), roles, cuisine, main_protein, total_min, servings, equipment, flags, nutrient_sources, inr≈, protein_g≈, source url + platform. **`body`:** caption/transcript, full ingredient lines (qty, unit, tumbler/ratio), steps with timers, notes. Variant = `rel: variant_of`. | `props.schema` on the kind record |
| Rating, tried, last eaten | **Derived from `meal` events** at query time, never stored on the recipe (no re-index, no stale card on "Cooked it") | — |
| Exercise / routine | `kind: exercise` (muscles, equipment, cues, source) / `kind: routine` (exercise list in body) | schema |
| Shared link waiting | `kind: inbox` (exists) · props: url, platform, status, reason · raw text in body | — |
| Meal | `form: meal` → data: slot, portion, rating, felt · refs the recipe | form record |
| Workout | `form: workout` → `data.ex: [{ref:"$bench", sets:[[8,40],[8,40],[6,42.5]]}]`, mode, min, rpe, energy · one `event_refs` row per exercise | form record |
| Check-in | `form: checkin` → mood 1–5, energy 1–5, sleep_h, stress 1–5, gut 0–5 | form record |
| Weight, symptoms, meds | as today (metric events, Food & Health assistant) | — |
| URL dedupe | one small system table `urls(canonical PK, ent)` (a mechanic, like `notified`) | migration |
| Food-flag map (~150 foods), Zod extraction schema, XP table | **TypeScript constants in `src/level/`, unit-tested** | yes |
| Things you tune by chat: phase, `have`, avoids/triggers, ranker weights, quest preferences | **unpinned** records, looked up by title | — |
| Pinned | **one** rule ≤ 220 chars pointing to the above (pinned rules ride in every chat) | — |

FTS uses `unicode61` without stemming: store singular canonical names and
normalise query words the same way ("eggs" → "egg").

**Tools: 5 today + 2 = 7.**

| Tool | What |
| :-- | :-- |
| `source` | `url` → raw caption/description/transcript + metadata, screenshots as MCP image content, or "already saved #id". (Not `fetch`, so Claude doesn't confuse it with its own web fetch.) |
| `suggest` | `{for: meal\|workout, slot?, treat?, minutes?}` → 3 options with a one-line why, video link and `/app` link. |

`context()` gains one line only after L3: `Lv 7 · STR 12 VIT 15 AGI 6 SEN 9 ·
quests 2/4` (≤ 60 tokens, measured with `scripts/smoke.mjs`). Before that it
gains "N shares waiting — structure them?".

## 3. Food

- **Capture (L1):** Android share → `/app/share` (today it only pre-fills quick
  add; L1 makes a recognised URL **save straight to `inbox`**). The server
  fetches raw text only — plumbing, not interpretation: blog JSON-LD / page
  text, YouTube Data API description (API key, no login), Instagram caption via
  Browser Run (Day-1 spike decides; else screenshots). Next chat: "3 shares
  waiting" → Claude reads them via `source`, writes recipes via `write`.
- **Dedupe:** canonical URL (strip `igsh`/`si`/`utm`; youtu.be/shorts → id; IG
  shortcode) → "already saved".
- **Health facts, rules decide:** factual flags from the food-flag map
  (`main` ingredients only); diet = veg + egg + non-veg; low-residue **only in
  flares**; Upadacitinib defaults: grapefruit avoid, food-safety flags.
- **`suggest {for: meal}`** (L2): v3's formula (cov · gap · taste · nut,
  48-h repeat penalty, triggers, ₹55/meal default), 2 tried + 1 want-to-try,
  date-seeded; **treat mode** = taste first. Two cheap queries: ready recipes
  by `ent_kind`; last meal per recipe from `event_refs ⨝ events` (60 days).
- **Recipe page (L1):** video link, ingredients, steps, one timer, "Cooked it"
  → meal event + 1-tap stars.
- **Cooking mode (L2):** Wake Lock, several big timers (minutes or cooker
  whistles), servings stepper (default 1, sane rounding; whistles/tempering not
  scaled), Scale / Tumbler / Ratio toggle.
- **Backlog (L2):** ≤ 2 Instagram accounts ("Download your information" links +
  one slow caption fetch from your home connection); YouTube by sharing a
  playlist URL. Lands raw in `inbox`, searchable by caption at once. Structuring
  500 by chat is heavy → §12 Q1.

## 4. Training

### 4.1 The Daily Quest (goal #470) — starts in week 1, not after L3

Goal: 100 push-ups · 100 sit-ups · 100 squats · 10 km — by January 2027.
Gym and home both, so the quest needs no equipment; gym days add strength work.

**Rank ladder** (anime hunter ranks; every number editable by chat):

| Rank | Push-ups | Sit-ups | Squats | Distance |
| :-- | --: | --: | --: | --: |
| E | 20 | 20 | 30 | 2 km |
| D | 35 | 35 | 50 | 3 km |
| C | 50 | 50 | 70 | 4–5 km |
| B | 70 | 70 | 85 | 6 km |
| A | 85 | 85 | 100 | 8 km |
| **S** | **100** | **100** | **100** | **10 km** |

- Reps in any number of sets through the day. Knee/incline push-ups count up
  to C; full push-ups from B.
- **Start rank** from a baseline on day 1: max push-ups, sit-ups, squats (2 min
  each), and an easy 2 km run/walk.
- **Rank up** next Monday when you clear ≥ 5 of the week's quest days (flare
  and planned rest days are left out of the count). **Never rank down.**
  E→S in 6 steps fits the 9 weeks from Nov 3 to Jan 4 with a week to spare.
- **Distance, honestly:** 10 km of *running every day* is ~70 km a week —
  months of build-up for most people, and riskier with Crohn's, low iron
  (injection still pending, #438) and low vitamin D (bone stress). So the
  daily quest counts **10 km moved** (run + walk, ~13k steps), with the
  *running* part growing about 10–15 % a week. **January milestone = one full
  S-rank day: 100/100/100 + 10 km run (run-walk allowed).** Daily 10 km
  *running* becomes the next goal when you say so.
- **Body first:** a quick OK from your doctor before starting. Sit-ups → dead
  bugs or planks if your gut complains. Abdominal pain or a flare → stop;
  that day is a **Recovery Quest** and still counts. One or two rest days a
  week are quest days too.
- **Logging:** "quest: 50/50/70, 4.2 km" → one `write` → a `quest` event
  (a `form` record, zero code). Run distance from any phone app, typed or
  later via Health Connect.
- **The System for the quest (small code, L1b):** the 08:00 push carries
  today's quest at your rank; rank-ups are announced ("[SYSTEM] Rank up: D → C").

### 4.2 Gym and strength

- **Logging from L1, zero code:** "push day: bench 3×8 @ 40, OHP 3×10 @ 20" →
  one `write` → `workout` event with per-exercise refs.
- Workout reels → `inbox` → Claude makes `routine`/`exercise` records.
- **`suggest {for: workout}`** (L4): days since each muscle group, today's
  energy, phase (flare → rest or gentle mobility), minutes, equipment; next load
  = last + small step when last RPE ≤ 8. PRs = max of the last events per
  exercise.

## 5. Mind & body

- **Check-ins from L1, zero code:** "mood 3, slept 6h, gut ok".
- **Weekly review** (lifeD's Sunday review) adds observations: sleep vs mood,
  meals vs gut, training vs energy — "bloating followed rajma 2 of 3 times",
  never a diagnosis.

## 6. The System (L3 — after ≥ 2 weeks of real logs)

- **XP for showing up, not for numbers.** Logging a meal, a check-in, a
  workout, a journal line earns XP. Mood/energy *values* never change XP (or
  you'd learn to inflate them). No XP for weight, no intake-amount quests
  ("log your meals", never "eat 80 g protein") — restrictive eating is a real
  risk with IBD and flares cut appetite.
- **Stats (the anime set, kept):** STR (quest push-ups/squats, strength
  sessions, PRs) · AGI (quest distance, cardio, mobility) · VIT (sit-ups/core,
  meals logged, sleep logged, meds taken) · SEN (check-ins, journal) · INT
  (lifeD tasks done, learning — **on**; never lowers anything; "stop counting
  work" by chat turns it off).
- **Never goes down:** a nightly rollup stores XP per day (rebuildable);
  today is computed live. Level = max(high-water mark, computed). XP-table
  changes apply from today, versioned.
- **Daily Quest** in the 08:00 push: 3–4 small logging/doing items from your
  quest preferences. **Flare = Recovery Quest** (rest, meds, fluids, soft food,
  log symptoms) and counts as a full day.
- **Low mood:** if mood ≤ 2 on 3 of your last 5 check-ins, quests shrink to one
  item and the System says once, plainly: talk to <the person you name> or
  Tele-MANAS (free, 24×7, India: **14416**). Not repeated daily.
- **Quiet:** "pause the System for a week" by chat; no level-up pushes after 21:00.
- Flags, not medical advice; your doctor overrides everything.

## 7. Android

lifeD's installed PWA stays the one app. Share target already exists (GET
`/app/share`). Files (screenshots/videos): service-worker POST or Telegram
files, decided by the Day-1 test. Owner cookie: 30 days today → change to 1 year
with "sign out everywhere". New screens: Recipe (L1), Cooking mode (L2), Status
(L3). Native Android only later, for Health Connect (steps, sleep, heart rate)
or a widget.

## 8. Changes to lifeD itself

- Merge `ccr-7a2881ef-38kr1m` → `main` (step zero).
- Docs out of date with the Paid plan: `DESIGN.md:27`, `STATUS.md:22,114`,
  `wrangler.jsonc:21` comment, AGENTS.md D1/CPU notes (account is on Workers
  Paid per autogram's config) — fix, keep "frugal".
- Dated STATUS/DESIGN entries: the `src/level/` domain-module exception; and
  later, the server-side-LLM decision if you say yes.
- New bindings over time: Browser Run (L1), Workflows (L2 backlog), Workers
  AI + AI Gateway (only if §12 Q1 = yes).

## 9. Phases

**Step zero (½ day):** merge the live branch to `main`; Paid-plan doc fixes.

**Day 1 — spikes:** 20 reels through Browser Run (pass = ≥ 14 full captions);
phone share tests (link → inbox; screenshot path).

**Quest track (parallel, from week 1 — the January deadline can't wait):**
`quest` form + rank-ladder record seeded on day 1 (zero code; Claude reads the
ladder and your logs and tells you today's quest). **L1b (~1 day):** today's
quest + rank in the 08:00 push and in `context()`; rank-up rule; announcement.

**L1 — capture + log (~2 days):** kinds + forms seeded (recipe, exercise,
routine; meal, workout, checkin, quest); share URL → `inbox` + raw fetch; `urls`
dedupe; `source` tool; Recipe page with "Cooked it"; one pinned pointer rule.
*Done when:* 10 real shares → saved raw, Claude turns ≥ 8 into recipes in one
chat; meals, workouts and check-ins log by chat.

**L2 — decide + backlog (week 2):** `suggest {for: meal}` + treat mode;
cooking mode; backlog import; §12 Q1 decision (Gemma or not).
*Done when:* "what should I eat?" gives 3 dishes you'd cook, none eaten in 4
days, phase rules respected; every saved reel is searchable.

**L3 — The System (after ≥ 2 weeks of logs):** XP rollup, stats, level,
quests, Status screen, guardrails.

**L4 — Training:** `suggest {for: workout}`, PRs, routine from reels.

**L5 — Mind & body:** weekly-review observations; optional Health Connect app.

## 10. Risks

| Risk | Mitigation |
| :-- | :-- |
| The new layer damages your life memory | Writes only through `write`; `src/level/` isolated; nightly R2 backup; tests |
| Every chat pays for it | 2 new tools; one ≤ 220-char pinned rule; no settings in pinned rules; measured |
| Gamification becomes pressure or disordered eating | XP for logging not values; no intake targets; never down; flare = full day; low-mood step; quiet mode |
| Instagram blocks Browser Run | Day-1 spike; screenshot path; home-connection backlog |
| Building on stale code | Step zero merges the live branch first |

## 11. Before Day 1 (you)

20 reel links with recipes in the caption · YouTube Data API key · ntfy topic
(lifeD's) · answers to §12.

## 12. Questions only you can answer

Answered: no server-side AI in L1 (re-decide for the backlog in L2); gym +
home; anime stats kept; INT counts work; first goal = the Daily Quest by
January (#470); everything tunable by chat.

Still open (all have defaults; change any time by chat):
1. Foods or exercises your doctor told you to avoid? OK with the grapefruit +
   food-safety defaults, and a quick doctor OK for the quest?
2. Kitchen kit, weighing scale, tumbler size; gym equipment you use?
3. Protein target, weight goal; is ₹5k/month all food or groceries only?
4. Who should the System suggest you talk to on a low stretch?

## 13. History

- **v1–v3 (Annam, separate app):** 3 review rounds; full text in git history
  (`plan/annam-plan.md`).
- **Owner answers (2026-11-01):** diet open; low-residue only in flares; ≤ 2
  Instagram accounts; YouTube by URL; cooking for one; treat meals; scale +
  tumbler + ratio; timers; name Solo Leveling; merge into lifeD.
- **v4:** merged into lifeD; training, mind & body and the System added.
- **Review round 4 → v5 (accepted):** no settings in pinned rules (they ride
  in every chat, cut to 220 chars); lifeD's "no server-side LLM" rule surfaced
  → L1 without it, decision to you; small recipe `props`, caption/transcript in
  `body`, rating/tried/last-eaten from meal events; singular canonical names
  for FTS; workout shape `data.ex[]` with per-exercise refs; XP high-water mark
  + nightly rollup + versioned table; XP for logging not values, no intake
  targets, INT off by default, concrete low-mood step (Tele-MANAS 14416),
  quiet mode; `src/level/` declared as a domain module with tested constants;
  re-sequenced (logging in L1, System after 2 weeks); `fetch` → `source`;
  tool count from 5. **Added by planner:** lifeD's live code is on
  `ccr-7a2881ef-38kr1m`, not `main`; its *formats as data* makes the
  meal/workout/check-in forms zero-code.
- **v6 (owner answers):** gym + home; anime stats kept, INT on; Daily Quest
  goal (#470) with an E→S rank ladder starting week 1 in parallel; "10 km
  moved" daily with a January S-rank day as the milestone; everything tunable
  by chat (code = schema + defaults + tests, records = overrides). lifeD
  records #470/#471 created. Moving this plan into lifeD's `docs/` needs push
  access to `omesh7/lifed`, which this session was denied.
