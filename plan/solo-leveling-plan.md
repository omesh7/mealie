# Solo Leveling — level-up layer of lifeD (plan v8 — final, ready to build)

Status: **plan only, no code.** v8 = v7 + review round 6 (final gate passed). v7 = round 5 (safer quest ladder, settings as events). v6 added owner answers (gym + home, anime
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
   **Done 2026-11-01:** `main` fast-forwarded to it (`ea83e74` → `c110435`).
3. **Use lifeD's "formats as data" (already shipped):** a `kind` record can
   carry `props.schema`, enforced by `write`; a `form` record (title = a log
   type) fixes the shape of that log's data. So `meal`, `workout`, `checkin`,
   `quest` and `setting` are just `form` records, and `recipe`, `exercise`, `routine` are just `kind`
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
6. **Everything is changeable by talking.** Code (`src/level/`) holds the
   *shape* (Zod schemas), the *defaults* and the tests. A change is a
   **`setting` log event** (`form: setting`, required `key` + `value`): "make
   the quest 80 push-ups" or "count cycling as AGI" is one `write`. The latest
   event per key wins, and its timestamp gives history and "applies from
   today" for free. `src/level/` validates overrides **on read**; an invalid
   one falls back to the default and the next push / `suggest` line says
   which setting was ignored and why. Core `write` stays untouched.
   **Tunable keys (the one list):** quest targets, progression %, deload,
   run-walk ratio, XP weights, stat sources, ranker weights, phase rules,
   avoids/triggers, `have`, quiet hours, side-quest preferences.

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
| Workout | `form: workout` → `refs: ["bench press","overhead press"]` (exact titles) + `data.ex: [{ex:"bench press", sets:[[8,40],[8,40],[6,42.5]]}]`, mode, min, rpe, energy (matched to refs by title in code) | form record |
| Quest | `form: quest` → `numbers: push, core, squat, run_km, walk_km` · several per day allowed; **the day's total = sum of that day's quest events** | form record |
| Setting | `form: setting` → `key`, `value` (Decision 6) | form record |
| Check-in | `form: checkin` → mood 1–5, energy 1–5, sleep_h, stress 1–5, gut 0–5, pain 0–3 | form record |
| Weight, symptoms, meds | as today (metric events, Food & Health assistant) | — |
| URL dedupe | one small system table `urls(canonical PK, ent)` (a mechanic, like `notified`) | migration |
| Food-flag map (~150 foods), Zod schemas, XP table, quest defaults | **TypeScript constants in `src/level/` (defaults), unit-tested**; overridden by `setting` events | yes |
| Pinned | **one** rule ≤ 220 chars pointing to the above (pinned rules ride in every chat) | — |

FTS uses `unicode61` without stemming: store singular canonical names and
normalise query words the same way ("eggs" → "egg").

**Tools: 5 today + 2 = 7.**

| Tool | What |
| :-- | :-- |
| `source` | `url` → raw caption/description/transcript + metadata, screenshots as MCP image content, or "already saved #id". (Not `fetch`, so Claude doesn't confuse it with its own web fetch.) |
| `suggest` | `{for: meal\|workout, slot?, treat?, minutes?}` → 3 options with a one-line why, video link and `/app` link. |

`context()` gains at most two short lines (≤ 60 tokens together, measured
with `scripts/smoke.mjs`): from L1 "N shares waiting — structure them?"; from
L1b `Quest (C): push 32/50 · core 20/50 · squat 0/70 · 3.1/5 km`; from L3 the
stats are appended to that line: `Lv 7 · STR 12 AGI 6 VIT 15 INT 20 SEN 9`.

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

### 4.1 The Daily Quest (goal #470) — starts in week 0, not after L3

**Goal:** 100 push-ups · 100 core · 100 squats · 10 km. **January milestone
(honest version):** one full **S-rank day** — 100/100/100 in any number of
sets + 10 km moved (run + walk) — plus a separate **5 km run-walk test**.
**The S-rank day unlocks only after every rep target has been ≥ 75 (B band)
for 2 qualifying weeks** — no single-day jump from ~65 to 100. If that isn't
reached by January, the milestone moves to February and the push says so,
without blame. Daily S and a 10 km run-walk are the Feb–Mar goals. Gym and home both: the
quest needs no equipment; gym days add strength work (§4.2).

**Week 0 (first week after the doctor's OK; ideally after the iron injection
#438 — Nov 2–8 if both are done):** baseline on day 1 and day 7 (max push-ups,
max core reps, max squats, each 2 min; easy 2 km run-walk, time it). Light
practice only, no progression yet.

**Daily targets grow from *your* baseline, not a fixed table:**
- Reps: start at **1.5 × baseline max**, spread through the day. Each
  **qualifying week** adds **+10 % (at least +5)**, capped at 100.
- Distance moved: start at **1.2 × your current daily km**, +10 % per
  qualifying week, to 10 km. Running part: run-walk intervals, run minutes
  +10 % a week, at most **one longer day** a week, **no running on
  symptomatic days**.
- **Deload every 4th calendar week counted from week 0:** targets × 0.7,
  nothing lost; progression resumes from the pre-deload targets.
- **A `setting` such as `quest.push = 80`** becomes the new current target and
  progression continues from it; `quest.push.hold = true` pins it.
- **Core 100** = any mix of crunches, dead bugs, leg raises, planks (10 s = 1
  rep) — easier on the gut than 100 straight sit-ups.

**Bands** (anime ranks; editable), from the **lowest** of the three rep
targets: E < 30 · D 30–49 · C 50–69 · B 70–84 · A 85–99 · **S = 100 on all
three + 10 km**. **Title = the highest band ever reached** — it never drops
when a deload or Re-awakening lowers today's targets, and a rank-up is
announced only the first time a band is reached.

**A qualifying week** = ≥ 4 *real* cleared days (the day's summed quest events
meet all four targets) **and** no day that week with check-in energy ≤ 2 or
pain ≥ 2. Missing check-ins don't block. (A hold is safe, so there's little
reason to fudge a score.)
- **Recovery and rest days protect your streak and XP but never count toward
  progression.** Seven flare days don't level the quest; they just don't hurt.
- **Re-awakening:** after ≥ 4 **consecutive** days without a cleared quest
  (flare, illness, infusion, travel), targets × 0.7 for 7 days, then resume.
- **Low-mood shrink** (§6): the push shows one quest item, at Re-awakening
  targets, until mood lifts.

**Body first:** doctor's OK before week 0. Tell the doctor about the new
training before blood tests: high-rep work raises CK, and Upadacitinib can
too. Stop on abdominal pain, dizziness or dark urine; hydrate on high-rep
days. A flare day is a **Recovery Quest** (rest, meds, fluids, soft food, log
symptoms).

**Logging:** "quest: 30 push, 25 core, 2.1 km walk" any number of times a day
→ `quest` events; the day's total is their sum. Distance from any phone app,
typed, or later via Health Connect.

**Day 1, before any code:** seed an unpinned rule **`daily quest`** holding
these rules in words (1.5 × baseline, +10 % min +5, deload, bands, title,
S-rank unlock, Re-awakening), plus the pinned pointer — so Claude can run the
quest by hand until L1b.

**L1b (small code, ships with its guardrails, before the first progression
Monday — Nov 16 if week 0 is Nov 2–8):** today's targets + title in the 08:00
push and in `context()`; weekly progression, deload, Re-awakening; rank-up
announcement ("[SYSTEM] Rank up: D → C"); **Recovery Quest on flare, low-mood
shrink, quiet mode / no pushes after 21:00**. The `daily quest` rule then
changes to "computed by the System; change it with a `setting`".

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
  sessions, PRs) · AGI (quest distance, cardio, mobility) · VIT (core reps,
  meals logged, sleep logged, meds taken) · SEN (check-ins, journal) · INT
  (lifeD tasks done, learning — **on**; never lowers anything; "stop counting
  work" by chat turns it off).
- **Never goes down:** a nightly rollup stores XP per day (rebuildable);
  today is computed live. Level = max(high-water mark, computed). XP-table
  changes apply from today, versioned.
- **Side quests** in the 08:00 push (separate from the fitness Daily Quest):
  2–3 small logging/doing items from your side-quest preferences ("log your
  meals", "evening check-in").
- **Guardrails (already live since L1b, §4.1):** flare = Recovery Quest that
  protects streak and XP; **low mood** — if mood ≤ 2 on 3 of your last 5
  check-ins, quests shrink to one item and the System says once, plainly:
  talk to <the person you name> or Tele-MANAS (free, 24×7, India: **14416**),
  not repeated daily; **quiet** — "pause the System for a week" by chat, no
  pushes after 21:00.
- Flags, not medical advice; your doctor overrides everything.

## 7. Android

lifeD's installed PWA stays the one app. Share target already exists (GET
`/app/share`). Files (screenshots/videos): service-worker POST or Telegram
files, decided by the Day-1 test. Owner cookie: 30 days today → change to 1 year
with "sign out everywhere". New screens: Recipe (L1), Cooking mode (L2), Status
(L3). Native Android only later, for Health Connect (steps, sleep, heart rate)
or a widget.

## 8. Changes to lifeD itself

- ~~Merge `ccr-7a2881ef-38kr1m` → `main`~~ — done 2026-11-01 (fast-forward, no prod change).
- Docs out of date with the Paid plan: `DESIGN.md:27`, `STATUS.md:22,114`,
  `wrangler.jsonc:21` comment, AGENTS.md D1/CPU notes (account is on Workers
  Paid per autogram's config) — fix, keep "frugal".
- Dated STATUS/DESIGN entries: the `src/level/` domain-module exception; and
  later, the server-side-LLM decision if you say yes.
- New bindings over time: Browser Run (L1), Workflows (L2 backlog), Workers
  AI + AI Gateway (only if you allow server-side AI for the backlog in L2).

## 9. Phases

**Step zero (½ day):** (merge done 2026-11-01) confirm migrations
`0008` and `0009` are applied to the remote D1 (STATUS says 0008 was pending)
before seeding new forms; Paid-plan doc fixes.

**Day 1 — spikes:** 20 reels through Browser Run (pass = ≥ 14 full captions);
phone share tests (link → inbox; screenshot path).

**Quest track (parallel, from week 0 — the January milestone can't wait):**
**All** kinds and forms (recipe, exercise, routine; meal, workout, checkin, quest, setting) + the `daily quest` rule + the pinned pointer seeded once, on day 1 (zero code; Claude reads
the `daily quest` rule + your baseline + logs and tells you today's targets). **L1b
(~1 day):** targets + title in the 08:00 push and `context()`; weekly
progression, deload, Re-awakening; rank-up announcement; the guardrails
(Recovery Quest, low-mood shrink, quiet mode).

**L1 — capture + log (~2 days):** (kinds and forms already seeded on day 1);
share URL → `inbox` + raw fetch; `urls`
dedupe; `source` tool; Recipe page with "Cooked it".
*Done when:* 10 real shares → saved raw, Claude turns ≥ 8 into recipes in one
chat; meals, workouts and check-ins log by chat.

**L2 — decide + backlog (week 2):** `suggest {for: meal}` + treat mode;
cooking mode; backlog import; re-decide server-side AI (Gemma) for the backlog.
*Done when:* "what should I eat?" gives 3 dishes you'd cook, none eaten in 4
days, phase rules respected; every saved reel is searchable.

**L3 — The System (after ≥ 2 weeks of logs):** XP rollup, stats, level,
side quests, Status screen.

**L4 — Training:** `suggest {for: workout}`, PRs, routine from reels.

**L5 — Mind & body:** weekly-review observations; optional Health Connect app.

## 10. Risks

| Risk | Mitigation |
| :-- | :-- |
| The new layer damages your life memory | Writes only through `write`; `src/level/` isolated; nightly R2 backup; tests |
| Every chat pays for it | 2 new tools; one ≤ 220-char pinned rule; no settings in pinned rules; measured |
| Gamification becomes pressure or disordered eating | XP for logging not values; no intake targets; title never down; flare protects streak; low-mood step; quiet mode |
| Training too fast (rhabdomyolysis, bone stress, flare) | Baseline-based targets; +10 %/week; deload every 4th week; Re-awakening after breaks; doctor's OK; CK caveat before blood tests |
| Instagram blocks Browser Run | Day-1 spike; screenshot path; home-connection backlog |
| Building on stale code | Step zero merges the live branch first |

## 11. Before Day 1 (you)

20 reel links with recipes in the caption · YouTube Data API key · ntfy topic
(lifeD's) · **doctor's OK for the quest** · **date of the iron injection
(#438)** · the week-0 baseline (day 1 and day 7) · answers to §12.

## 12. Questions only you can answer

Answered: no server-side AI in L1 (re-decide for the backlog in L2); gym +
home; anime stats kept; INT counts work; first goal = the Daily Quest, January milestone = one S-rank day (#470)
+ 5 km run-walk test; everything tunable by chat.

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
- **Review round 5 → v7 (accepted):** baseline-based targets (1.5 × max,
  +10 %/qualifying week, deload every 4th week) instead of a fixed weekly
  ladder; week 0 after the doctor's OK and ideally the iron injection;
  qualifying week = ≥ 4 real cleared days + energy ≥ 3 + no pain; recovery/rest
  days protect but never progress; title never down, target steps back after
  breaks ("Re-awakening"); core 100 instead of sit-ups; January = one S-rank
  day + a 5 km run-walk test; CK caveat; guardrails moved into L1b; tunables
  as `setting` log events validated on read (core `write` untouched); workout
  refs by exact title; quest form with run/walk split and daily sum; "Side
  quests" renamed; one tunables list; step zero confirms migrations 0008/0009.
- **Review round 6 → v8 (final gate, accepted):** S-rank day unlocks only
  after 2 qualifying weeks with every rep target ≥ 75, else the milestone
  moves to February; band = lowest rep target, title = highest band ever
  reached, rank-up announced once; `pain` added to `checkin`; qualifying week
  = ≥ 4 cleared days and no day with energy ≤ 2 or pain ≥ 2 (missing
  check-ins don't block); Re-awakening after ≥ 4 *consecutive* missed days,
  targets × 0.7 for 7 days; deload every 4th calendar week from week 0;
  low-mood shrink = one item at Re-awakening targets; a `daily quest` rule
  seeded on day 1 so Claude can run the quest by hand until L1b; L1b due
  before the first progression Monday; `setting` semantics (new current
  target, `.hold` pins); all kinds and forms seeded once on day 1.
  Reviewer verdict: ready to hand to a builder.
- **lifeD `main`** fast-forwarded to the live branch (2026-11-01). This plan
  now also lives in lifeD as `docs/LEVEL.md`.
