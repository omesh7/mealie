# Solo Leveling — level-up layer of lifeD (plan v4)

Status: **plan only, no code.** v4 replaces the separate "Annam" app (v1–v3)
with a layer inside lifeD. The food design from v1–v3 survives almost
unchanged; only where it lives changed. History: §13.

---

## 0. The decision: one system, not two

**You're right — it is part of life.** lifeD is already a ledger of your life:
records (tasks, projects, goals, habits, health), events (what happened),
rules, one connector, one Inbox, one share target, one backup, one phone app.
Meals, workouts, mood and recipes are more of the same: things you keep and
things that happened. Splitting them would cost:

- two connectors in every chat (we already had to rename tools to avoid a clash),
- a "who owns what" contract between two databases (meals here, symptoms there),
- two logins, two backups, two share targets, two Inboxes,
- and the best analysis — food vs symptoms vs mood vs training vs work — would
  always be a cross-database join done by hand.

**The one real tension** is lifeD's own rule: *"No domain logic in code. If you
are about to write `if (kind === "task")`, stop."* A recipe ranker and an
Instagram importer are domain-ish. The plan honours the rule this way:

1. **Everything you can describe is data** (rule records the AI edits by chat):
   kinds, extraction prompts + JSON schemas, the food-flag map, ranking
   weights, phase rules, XP table, quest pool.
2. **The code that remains is generic engines and integrations**, the same
   category as lifeD's existing `github.ts`, `telegram.ts`, `google/calendar.ts`:
   an **ingest** integration (URL/file → record of some kind), a **ranker**
   (scores records by weights stored in a rule), and a **level** engine (XP
   from events by a table stored in a rule). None of them say `if kind === "recipe"`.
3. **Isolated folder** (`src/level/`), writes only through lifeD's existing
   `write` path, nightly R2 backup already covers it, one feature flag.

## 1. Name and theme

**Solo Leveling** — lifeD is "the System"; you are the player.
Daily Quests arrive in the morning push, stats grow from real events, level-ups
are announced. Private app, so the name is fine. If any of this ever goes public
(your AI-education brand), pick an original name: "Solo Leveling" is someone
else's trademark.

## 2. What it covers

| Module | Keeps | Logs | Helps you |
| :-- | :-- | :-- | :-- |
| **Food** (first) | recipes from reels/YouTube/blogs/screenshots | meals, ratings | "what should I eat?", adapt, mix and match, cooking mode |
| **Training** | exercises, routines from reels | workouts (sets/reps/kg, cardio min) | "what should I train today?", progressive overload, PRs |
| **Mind & body** | — | mood/energy/sleep check-ins, weight, symptoms (already logged) | trends, weekly review, gentle flags |
| **The System** | XP table, quest pool | (derived) | stats, level, daily quests, streaks without punishment |

## 3. How it maps onto lifeD (no new core tables)

| Thing | lifeD shape |
| :-- | :-- |
| Recipe | entity `kind: recipe` · `props`: ingredients[], steps[] (timer in min or whistles), roles[], cuisine, total_min, servings (default 1), equipment[], main_protein, protein_g≈, inr≈, flags, nutrient_sources[], cooked (want/tried), rating, sources[] (url, canonical, platform, author, caption, lang, thumb) · `body`: notes · variant = `rel: variant_of` |
| Exercise | entity `kind: exercise` · props: muscles[], equipment, cues, sources[] |
| Routine (from a workout reel) | entity `kind: routine` · props: exercises[] with sets/reps/rest, level, minutes, sources[] |
| Shared link waiting/failing | entity `kind: inbox` (exists) · props: url, status, reason, attempts |
| Meal | event `type: meal` → refs recipe · data: slot, portion, rating, felt, main_protein, cuisine |
| Workout | event `type: workout` → refs routine/exercises · data: mode, min, sets[[reps,kg]], rpe, energy |
| Check-in | event `type: checkin` · data: mood 1–5, energy 1–5, sleep_h, stress 1–5, gut 0–5 |
| Weight etc. | existing `metric` events ("weighed 70.1") |
| Symptoms, meds | existing (Food & Health assistant) |
| `have` list, phase, food-flag map, rankers, XP table, quest pool, extraction prompts+schemas | pinned **rule** records (edited by chat) |
| URL dedupe | one system table `urls(canonical PK, ent)` — a mechanic, like `notified` |

All existing tools then work for free: `find kind:recipe`, `get #id`, `write`
(upsert a recipe, log a meal/workout/check-in in one call), Journal, Search,
Inbox, backups, "on this day".

**New MCP tools: two** (lifeD keeps "few tools, short schemas"):

| Tool | What |
| :-- | :-- |
| `suggest` | `{for: meal\|workout, slot?, treat?, minutes?}` → 3 ranked options with a one-line why + video link + `/app` link, using the ranker rule for that `for`. |
| `fetch` | `url` → raw caption/description/transcript + metadata, or "already saved #id". Lets Claude structure in-chat (free). Video-only → async, lands in Inbox. |

`context()` (no args) gains a 2-line **System** block: `Lv 7 · STR 12 VIT 15 AGI 6
INT 20 SEN 9 · Today: 2/4 quests` — only lines, no tables. Server
instructions gain the food/training/health rules from v3 §4.4.

## 4. The System (gamification that helps, never punishes)

- **Stats from real events** (XP table is a rule record you can change):
  STR ← strength workouts (volume, PRs) · AGI ← cardio/steps/mobility ·
  VIT ← meals logged with protein/iron/B12 sources, sleep ≥ 7 h, meds taken,
  water · INT ← lifeD tasks/projects done in work spaces, learning ·
  SEN ← journal, check-ins, meditation, a calm day.
- **Level** = f(total XP), recomputed from events any time (nothing stored that
  could drift). Level-ups and new titles ("Kitchen Hunter: 25 recipes cooked")
  go out by ntfy.
- **Daily Quest** in the 08:00 push (lifeD's digest already exists), picked
  from the quest-pool rule by phase and energy: e.g. *"Daily Quest: 30-min
  walk · 2 meals logged · 80 g protein · mood check-in · 1 deep-work block."*
- **Guardrails (written into the rules):**
  - **Flare = Recovery Quest** (rest, meds, fluids, soft food, log symptoms).
    It counts as a full day. Never a workout push during a flare.
  - **No penalty quests, no lost levels, no streak shame.** Rest days count;
    streaks pause, they don't reset.
  - **Low-mood days shrink the quest** to one small thing. If check-ins stay low
    for several days, the System says so plainly and suggests talking to
    someone you trust or your doctor — it doesn't push harder.
  - Flags, not medical advice; the doctor overrides everything.

## 5. Food module (v3 design, re-homed)

Unchanged from v3 unless noted:

- **Ingest** = an integration Workflow: share/`fetch` → classify (**recipe |
  routine | exercise | other**) → fetch source → structure → validate → flags →
  dedupe → `write`. One pipeline for food *and* workout reels.
  - Blog: JSON-LD `Recipe` (no AI). YouTube: Data API description (API key, no
    login), Gemini on the URL if needed. Instagram: Browser Run embed page (Day-1
    spike decides), else screenshot path. Screenshots: Gemma 4 vision. Text: Gemma 4.
  - In chat, Claude structures (free) via `fetch` + `write`. Share sheet with no
    chat → Gemma 4 (one call; Zod/JSON-schema check; one repair; Gemini fallback).
  - Extraction prompt + JSON schema per kind are **rule records** (edit by chat).
  - Auto-ready when valid (≥ 3 ingredients, ≥ 1 step); Inbox only for problems;
    push only on failure; a screenshot within 10 min joins its link.
- **Backlog:** Instagram (≤ 2 accounts) via "Download your information" links +
  a one-time slow caption fetch from your home connection; YouTube old saves by
  sharing a playlist URL. Fanned out with Workflows `createBatch`.
- **Health facts, rules decide:** factual flags from a food-flag map (a rule
  record, ~150 foods, canonical English names, `main` ingredients only);
  diet = veg + egg + non-veg; low-residue **only in flares**; Upadacitinib
  defaults: grapefruit avoid, food-safety flags.
- **Ranker (meal):** v3 §4.5 formula, weights in the `ranker: meal` rule;
  treat mode = taste first; ₹55/meal default; 2 tried + 1 want; date-seeded.
- **Cooking mode** in `/app`: Wake Lock; several big timers (minutes or cooker
  whistles) with sound + vibration; servings stepper (default 1, sane rounding,
  whistles/tempering not scaled); Scale / Tumbler / Ratio toggle; "Cooked it" →
  meal event + 1-tap stars (+XP).

## 6. Training module

- **Log by chat:** "push day: bench 3×8 @ 40, OHP 3×10 @ 20, 20 min walk" →
  one `write` → workout event (+ exercise records created/matched).
- **Workout reels** go through the same ingest → `routine`/`exercise` records
  with the video linked.
- **`suggest {for: workout}`**: ranker rule over days-since-muscle-group, energy
  from today's check-in, phase (flare → none/gentle mobility), time available,
  equipment; next load = last + small step if last RPE ≤ 8 (progressive overload).
- PRs detected from events; a PR = XP + a push.

## 7. Mind & body module

- **Check-in** in 10 seconds (app tiles or chat: "mood 3, slept 6h, gut ok").
- **Trends** in the weekly review (lifeD has a Sunday review): sleep vs mood,
  meals vs gut, training vs energy — as observations ("bloating followed rajma
  2 of 3 times"), never diagnoses.
- Weight and symptoms keep flowing in as today.

## 8. Android

- lifeD's installed PWA stays the one app. Its share target (`/app/share`,
  GET) learns one thing: a URL from Instagram/YouTube/a blog → ingest; anything
  else → quick add as today. Screenshots/videos: SW-POST or Telegram files,
  decided by the Day-1 test (v3 §6). Owner cookie → 1 year.
- New `/app` screens: **Status** (level, stats, today's quests), **Recipe /
  cooking mode**, **Library** (recipes, routines), **Check-in** tiles. Inbox,
  Journal, Search already exist.
- **Native app only if** you want Health Connect sync (steps, sleep, heart rate
  from a watch) or a home-screen widget. That's the one strong reason for it;
  later, not now.

## 9. What changes in lifeD itself

- **Plan & limits:** the account runs Workers Paid (autogram's config says
  so), so lifeD's "Cloudflare Free" principle and the D1/CPU notes in
  `AGENTS.md` are out of date. Update them in the first commit; keep "frugal".
- **New bindings:** Workflows (`ingest`), Browser Run, Workers AI, AI Gateway,
  R2 (exists: thumbnails go next to backups).
- **Docs:** `docs/LEVEL.md` (this plan, trimmed), `STATUS.md` row, `DESIGN.md`
  principle note on engines vs domain logic.
- **Token budget:** measure `context` + 2 new tools with `scripts/smoke.mjs`;
  the System block must stay ≤ 60 tokens.

## 10. Phases

**Before Day 1 (you):** 20 reel links with recipes in the caption; YouTube API
key; Gemini key (later); confirm Workers Paid; answers to §12.

**Day 1 — spikes:** Instagram via Browser Run (≥ 14/20 full captions = pass);
phone share tests (GET link, file path). The MCP spike is gone — lifeD's
connector already works on your phone.

**L1 — Food capture (3–4 days):** rule records seeded (kinds, flag map,
extraction prompts, phase rules); `fetch` tool; ingest Workflow + share
routing; `urls` dedupe; recipe + cooking-mode screens; meal events.
*Done when:* 10 real shares → ≥ 8 usable recipes, and "had dosa for dinner"
logs a meal.

**L2 — Backlog + "what should I eat" (week 2):** Instagram/YouTube backlog;
`suggest {for: meal}`; treat mode.
*Done when:* every saved reel is searchable in lifeD, and suggestions are 3
dishes you'd cook, none eaten in 4 days, phase rules respected.

**L3 — The System (2–3 days):** XP table, stats, level, Daily Quest in the
morning push, Status screen, guardrails.
*Done when:* a normal day's logs move the bars, a flare day gets a Recovery
Quest, and nothing ever goes down.

**L4 — Training (week 3):** workout logging, exercise/routine kinds, workout
reels via ingest, `suggest {for: workout}`, PRs.

**L5 — Mind & body (later):** check-ins, weekly-review trends, optional Health
Connect companion app.

## 11. Risks

| Risk | Mitigation |
| :-- | :-- |
| A bug in the new layer damages your life memory | Writes only through `write`; isolated `src/level/`; feature flag; nightly R2 backup; tests before deploy |
| Tool/context bloat in every chat | 2 new tools; System block ≤ 60 tokens; measured with `smoke.mjs` |
| Gamification turns into pressure | No penalties, flare = full day, low mood shrinks quests |
| Instagram blocks Browser Run | Day-1 spike; screenshot path; home-connection backlog script |
| Health overreach | Facts stored, rules decide; observations not diagnoses; doctor overrides |
| Name clash if public | Private only; rename before anything public |

## 12. Questions only you can answer

Answered: diet open; low-residue only in flares; ≤ 2 Instagram accounts;
YouTube by URL; cooking for one; treat meals; scale + tumbler + ratio; timers.

Still open:
1. Foods your doctor told you to avoid? OK with the grapefruit + food-safety defaults?
2. Kitchen kit and weighing scale? Tumbler size (or the app asks once)?
3. Protein target, weight goal; is ₹5k/month all food or groceries only?
4. Training today: gym, home, or both? What equipment? Any exercises your
   doctor told you to avoid?
5. The five stats (STR, AGI, VIT, INT, SEN) — keep the anime set or rename?
6. Should work (INT from lifeD tasks) count toward your level, or only body and mind?

## 13. History

- **v1–v3 (Annam, separate app):** three review rounds. Kept: day-1 spikes,
  Claude structures in chat, Gemma only for share/backlog, backlog import,
  factual flags + query-time rules, Upadacitinib defaults, ranking formula,
  auto-ready imports, cooking mode, failure-only pushes. Full text in git
  history (`plan/annam-plan.md`).
- **Owner answers (2026-11-01):** see §12.
- **v4:** merged into lifeD as the Solo Leveling layer; scope widened to
  training and mind & body; tool-name clash and the two-database ownership
  contract removed; MCP spike dropped.
