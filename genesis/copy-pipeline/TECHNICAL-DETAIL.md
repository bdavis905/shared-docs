# The Exodus Copy Pipeline 


**Five terms, defined once:**
- **Convex** — the app's database and backend. Holds runs, ideas, brands, everything.
- **Trigger.dev** — the job-runner in the cloud. When a run starts, a Trigger.dev "task" (a script we wrote) does the actual work step by step.
- **Genesis server** — bot server at gas.copycoders.ai. It holds the bots' system prompts (their personalities and instructions). **The prompts are NOT in this repo** — this repo only sends the bots their *inputs* (primer, brief, instructions).
- **Primer** — the brand-foundation document (winning ads + brand facts) that gets fed to a bot before it writes anything.
- **Claude Code (local)** — the AI assistant running on the customer's own machine. It reads "skill" instruction files and runs the `exodus` command-line tool.

---

## 1. The Whole Flow, Numbered (mark your gates between these lines)

This is the default run: a brief goes in, 2 ad variants come out in a Google Doc.

```
WHERE THE WORK HAPPENS:        [Local]   = Claude Code on the customer's machine
                               [Cloud]   = Trigger.dev script
                               [Bot]     = a Genesis bot on server
                               [DB]      = Convex backend

 1. [Local] Customer asks for ads (gives an idea, a reel, a swipe, or a brief)
 2. [Local] Claude Code WRITES THE BRIEF — freeform, improvised, no template enforced
            (shows it to the customer as a "gut check" but nothing requires approval)
 3. [Local] Claude Code fires `exodus genesis` → the run is created in the database
 4. [DB]    Backend checks the brand is ready (primer exists, customer's AI keys work),
            snapshots the brand foundation, and hands the job to the cloud
 5. [Cloud] Step "seed-capture"  — reads any creative seeds passed with the run
 6. [Cloud] Step "primer-select" — picks the right primer for the awareness level
 7. [Bot]   Step "hook-generation-mario"  — ad-hook-bot-1 writes ~20 hooks   ─┐ run at the
 8. [Bot]   Step "hook-generation-infeed" — in-feed-vsl-bot writes ~20 hooks ─┘ same time
 9. [Cloud] Step "hook-selection" — a SCRIPT (not a human, not a bot) picks the
            N most diverse hooks from the ~40-hook pool (N = number of variants)
10. [Bot]   Step "body-copy-mario"  — mariobot writes body copy from hook #1,
            then writes 4 headlines in the same conversation
11. [Bot]   Step "body-copy-infeed" — in-feed-vsl-bot does the same from hook #2
            (if more passes were requested, more body-copy rounds happen here)
12. [Cloud] Step "qa-checklist" — a script runs a 15-point check + auto-fixes
13. [Cloud] Step "doc-write" — formats everything and creates the Google Doc
            (one tab per variant)
14. [DB]    Results saved; dashboard run page and Doc link go live
15. [Local] Claude Code surfaces the Doc to the customer
            (OPTIONAL afterwards: per-variant editing menu → edit-pass bot)
```

**The key fact for your re-architecture: from step 3 to step 14 there is NO stopping point.** Once `exodus genesis` fires, the whole chain runs straight through to a finished Doc. Every human choice happens either before step 3 or after step 14.

---

## 2. Step-by-Step Detail Table

For each step: what it is, what goes in, what comes out, who does it, whether it's a defined skill / a script / improvisation, and whether a human can intervene today.

| # | Step (plain English) | Goes IN | Comes OUT | WHO does it | Skill / script / improvised? | Human can intervene? |
|---|---|---|---|---|---|---|
| 1 | Customer asks for ads | An idea, reel URL, swipe link, or rough brief | A conversation with Claude Code | Claude Code (local) | Skills route the request (`write`, `genesis`, `idea`, `swipe` skills) | Yes — this IS the human |
| 2 | **Brief is written** | Whatever the customer gave | A 1–2 paragraph freeform brief | Claude Code (local) | **Improvised.** No brief skill, no template, no validation — skills give style *guidance* only (see §5) | Soft — brief is shown as a "gut check," but nothing blocks the run |
| 3 | Run is fired | Brief + awareness level + variant count + optional seeds/steering | A run record (status "pending") | Claude Code calls the `exodus` CLI → API | Script (`exodus/commands/genesis.ts` → `POST /api/v2/genesis`, `convex/http.ts:2510`) | No |
| 4 | Preflight + handoff to cloud | The run record | Brand-foundation snapshot; job queued on Trigger.dev | Convex (`genesis.startRun`, `convex/genesis.ts:39-539`) | Script. Fails fast if primer or customer keys are missing | No (it can only *fail*, not pause) |
| 5 | seed-capture | Seeds passed with the run (optional creative angles, one per line) | Seeds available to the hook step | Trigger.dev (`scout/src/trigger/genesis-pipeline.ts`) | Script | No |
| 6 | primer-select | Awareness level + the brand's primers | The primer text the bots will be fed (falls back to the full brand primer if the awareness-specific one is empty) | Trigger.dev | Script | No (steering text baked into the primer applies here — see §6) |
| 7 | Hook writing, Mario voice | Primer → then an instruction containing the brief, seeds, hook bank, steering | ~20 hooks | **Genesis bot `ad-hook-bot-1`** | Bot (system prompt on Luke's server). The *instruction* is a script template (`scout/src/prompts/hook-generation.ts`). Runs a 3-call routine: prime → generate → "push to level 3" for more visceral hooks | No |
| 8 | Hook writing, Infeed voice | Same inputs | ~20 hooks | **Genesis bot `in-feed-vsl-bot`** | Same 3-call routine, runs in parallel with step 7 | No |
| 9 | **Hook selection** | The merged ~40-hook pool | N hooks (default 2), picked for diversity | Trigger.dev | Script (`selectDiverseHook`). **A manual override field (`selectedHookIndices`) already exists in the code** but no surface lets a customer use it today | **No today — but this is the easiest gate to add** (the plumbing for human hook-picking is already half-built) |
| 10 | Body copy + headlines, Mario | Primer → hook #1 → brief → headline bank → steering | 700–1500 words of body copy + 4 headlines | **Genesis bot `mariobot`** | Bot. 3-call conversation: prime → write body → write headlines (`scout/src/trigger/genesis-body-copy.ts`) | No |
| 11 | Body copy + headlines, Infeed | Primer → hook #2 → same extras | Body copy + 4 headlines | **Genesis bot `in-feed-vsl-bot`** | Same 3-call conversation | No |
| 12 | QA checklist | All variants | Cleaned variants (15-point check + auto-fixes) | Trigger.dev | Script (`scout/src/cleanup/qa-checklist.ts`) — automated, no judgment call | No |
| 13 | Google Doc written | All variants | A formatted Doc, one tab per variant, hooks merged into the full ad | Trigger.dev (`genesis-google-doc` task) | Script | No |
| 14 | Results saved | Doc URL + variants | Run flips to "completed"; dashboard page renders | Convex (callback → `updateRun`) | Script | No |
| 15 | Doc surfaced + optional edits | The finished run | Customer reads the Doc; can request per-variant edits | Claude Code (local) + **Genesis bot `edit-pass`** for edits | Edit menu is a defined feature (Natural-language / Shorten / Cut / Simplify / Make Better) | **Yes — the only built-in feedback loop, and it's after everything is done** |

---

## 3. How the Variance Works (two voices, hooks first)

The pipeline gets variety from **two separate mechanisms**:

**Mechanism 1 — hooks are generated separately, BEFORE any copy is written.**
Two different bot voices (`ad-hook-bot-1` = Mario voice, `in-feed-vsl-bot` = Infeed voice) each produce ~20 hooks at the same time. The pools are merged (~40 hooks) and a script picks the most diverse N. Important detail: the chosen hook is then **injected into the body-copy prompt** — the body bot writes the ad *around* the hook it was given. The hook is not bolted on afterwards.

**Mechanism 2 — two different writers produce the variants.**
One "pass" = one variant from `mariobot` + one variant from `in-feed-vsl-bot`, both writing from the brand primer. That's why the default output is 2 ads: same brief, same brand, two different voices.

**More passes = more variants**, with a twist on pass 2 (`planTracks`, see Appendix A2):

| Passes requested (`--passes`) | Variants | What each pass uses |
|---|---|---|
| 1 (default) | 2 | Mario × Brand primer + Infeed × Brand primer |
| 2 | 4 | adds: Mario + Infeed × **Top-Ads-Biased primer** (built from the top-performing ads the brand has shared — falls back to the brand primer when none are available) |
| 3+ | 6+ | repeats the brand-primer pair |

Each variant is labeled by its **track** (mario-brand / infeed-brand / topads) and gets its own tab in the Google Doc.

**Headlines are not a separate bot.** After a bot finishes the body copy, the same conversation continues with one more message: "now write 4 headlines" (fed the brand's winning-headline bank if one exists). So every variant arrives as a package: hook + body + 4 headlines, all from one bot in one conversation.

---

## 4. The Three Sources: One Pipeline, Different Front Doors

Gambit, organic, and swipe are **not three pipelines**. They are three different *capture* steps that all deposit into the same place — the **Idea Bank** — and from there one shared escalation path feeds the exact same Genesis writer described above.

```
 GAMBIT                      ORGANIC                      SWIPE (URL)
 customer brain-dumps        customer pastes an           customer pastes a Facebook
 raw angles as text          Instagram/TikTok reel        Ad Library link
      │                           │                            │
      ▼                           ▼                            ▼
 [Bot: gambit-splitter]      scrape + transcribe +       scrape ad (copy, video
 splits the dump into        read on-screen text,        transcript, on-screen text),
 distinct ideas              then [Bot: organic-         then [Bot: swipe-concept-
      │                      idea-extractor]             extractor]
      │                           │                            │
      └──────────────┬────────────┴────────────────────────────┘
                     ▼
              IDEA BANK  (one shared table; each idea = a hook + a short
              concept + optional notes; keys: G1… / O1… / A1…)
                     │
            ── THE REAL CONVERGENCE POINT ──
            idea.description  →  becomes the brief, as-is
            idea.notes        →  appended as "Direction / texture: …"
            idea.hook         →  becomes a creative seed
                     │
                     ▼
              GENESIS PIPELINE  (steps 4–14 in §1 — identical for all three)
```

**Where they're the same:** everything from the Idea Bank onward. One dispatch function (`buildEscalationArgs`, Appendix A4) converts any banked idea into a standard brief-mode run. The bots never know which source an idea came from.

**Where they differ:** only at capture — what gets scraped and which *extraction* bot forms the idea. Side by side:

| | Gambit | Organic | Swipe (URL) |
|---|---|---|---|
| Customer gives | freeform text dump | reel URL | FB Ad Library URL |
| What's scraped | nothing (text only) | caption, transcript, first-frame on-screen text | headline, body, CTA, video transcript, on-screen text |
| Extraction bot | `gambit-splitter` | `organic-idea-extractor` | `swipe-concept-extractor` |
| Ideas per input | several (one dump → many ideas) | one per reel | one per ad |
| Idea Bank key | G1, G2… | O1, O2… | A1, A2… |
| Extra input | — | — | `--steering` text ("what to model from this ad") saved as the idea's notes |

**Two real caveats:**

1. **Bank-then-write vs. write-now.** The customer can either *bank* ideas and write later (`idea` skill — capture, curate, then `idea write O3`), or fire capture-and-write in one shot (`genesis --reel <url>`, `genesis --swipe-url <url>`). The one-shot versions run the exact same capture → bank → escalate chain; they just don't pause at the bank. So the curation gate exists, but the one-shot commands skip past it by design.

2. **The one true divergence: saved swipes.** Writing from a *previously saved* competitor ad (`exodus genesis --swipe <id>`) does NOT go through the Idea Bank or the Genesis pipeline at all — it sends the ad text as a "paste" input, which routes to a completely different cloud task called **remix-pipeline** (`convex/genesis.ts:297-299`). If you want all swipe writing on one rail, this is a seam to know about.

(House-keeping note: there's also a legacy "scout bank" — an older table filled by an automated Instagram scraper, visible only in the admin view. The Idea Bank above is the live system; the scout bank is being phased out and doesn't feed the normal copy flow.)

---

## 5. The Brief-Building Step — Confirmed Still Improvised

You asked whether brief-building is still "not a defined skill — Claude Code just does it ad hoc." **Confirmed, with one nuance.**

- There is **no brief-building skill, no template enforced, no schema, no validation.** The backend accepts the brief as any non-empty text (`brief: v.string()` — Appendix A5). The dashboard's brief box is a plain free-text area.
- The nuance: two skills (`write` and `genesis`) contain *written guidance* telling the local Claude Code what a good brief looks like — "a 1–2 paragraph description of the ad, not the ad itself," covering scenario, emotional core, mechanism bridge, tone — and a suggested format (`BRIEF: / Hook: / Source: / paragraphs`). Direct quote from the genesis skill: *"If the user gave you an idea, a winning ad, or a rough angle, **you write the brief from it**; surface it for a quick gut-check before running."*
- So: guidance exists, enforcement does not. Whether the brief follows the format, whether the gut-check actually happens, and whether the customer really looks at it — all depend on the local Claude Code's behavior in the moment. **Two customers asking for the same thing can get structurally different briefs.**
- And when writing from a banked idea, **there is no brief-writing step at all** — the idea's stored description is shipped to Genesis verbatim (Appendix A4). The quality ceiling is whatever the extraction bot wrote at capture time.

For your modular redesign, this is the most consequential finding: **the single highest-leverage input to the whole system (the brief) is the least defined step in it.**

---

## 6. Current Intervention Points (what exists today)

Honest framing: **nothing in the system hard-blocks a run.** Today's intervention points are all either *before* the pipeline fires or *after* it finishes.

| # | Where | What a human can do | Strength |
|---|---|---|---|
| 1 | Idea Bank curation (CLI) | List banked ideas, edit a concept (`idea edit O2 "…"`), attach direction notes (`idea note O2 "lean on the cortisol angle"`), delete, and — crucially — **choose which ideas get written at all** (`idea write G1,S4`) | Real gate, but only on the bank-then-write path; one-shot commands (`--reel`, `--swipe-url`) skip it |
| 2 | Idea dismissal (dashboard) | Browse and dismiss scraped ideas so they don't show in lists | Cosmetic filter — doesn't block anything |
| 3 | Brief gut-check (conversation) | The `write`/`genesis` skills tell Claude Code to show the brief before firing | Transparency only — convention, not a mechanism; nothing waits for a "yes" |
| 4 | Brand-level steering (persistent) | Per-brand "always do / never mention" text in four lanes — body (cold), body (warm), hooks, headlines — plus `###IMPORTANT` steering slots inside the primers. Rides into every future run | Real influence, but brand-wide and ahead of time — not per-run, not per-step |
| 5 | Per-variant edit menu (after the run) | On the finished run page: Natural-language edit / Shorten / Cut / Simplify / Make Better — each fires the `edit-pass` bot on one variant and writes a new Doc tab | Real feedback loop — but only after all 10 steps already ran |

What does **not** exist today: any way to see or pick hooks before body copy is written, any way to approve the brief as a blocking step, any pause between pipeline steps, and any way to redirect a run in flight.

---

## 7. Candidate Intervention Points (the natural seams)

These are the places where the code already has a clean boundary — where a gate could be inserted without re-architecting the step on either side. Ordered by flow position; ★ = most natural / least invasive.

| Seam | Between steps (§1 numbering) | What the human would do | Why it's a natural cut |
|---|---|---|---|
| **A. Idea approval** | 1 → 3 | Approve/refine ideas before any are written | Already exists on the bank path. The change is making it universal: route the one-shot commands (`--reel`, `--swipe-url`) through the same review instead of skipping it |
| **B. Brief approval** ★ | 2 → 3 | Read, edit, and explicitly approve the brief before the run fires | The "gut-check" convention already exists — this just turns a custom into a mechanism. Cheapest gate in the whole list, and it guards the most important input |
| **C. Hook review** ★ | 8/9 → 10 | See the ~40-hook pool, pick (or rank) the hooks the body bots write from | **Half-built already**: the pipeline accepts manual hook choices (`selectedHookIndices`) — what's missing is a pause + a surface to show the pool. This is also where splitting into two pipelines is most natural: a "hooks pipeline" (steps 1–9) and a "copy pipeline" (steps 10–15), with a human between them |
| D. Pre-body brief+hook check | 9 → 10 | Confirm each hook-and-direction pairing before the long body-copy writes start | Same seam as C, slightly richer: approve hook + any per-variant direction together |
| E. Draft review before Doc | 11 → 12/13 | Read raw variants, kick weak ones back for a rewrite before QA/Doc formatting | Variants exist as structured data at this point; the QA step is already a checkpoint shape — it's just automated. A human (or human + bot) could sit here |
| F. Per-variant edit loop | after 15 | Already exists (edit menu). Could grow from "fix this variant" into "regenerate from step X with this feedback" | The callback/edit plumbing is built; extending it backward into the pipeline is the natural evolution |

**If you only add two gates:** B (brief approval) and C (hook review). B guards the input, C sits at the pipeline's one already-half-built pause point, and together they turn the current "fire and pray" run into idea → **approve brief** → hooks → **pick hooks** → copy → Doc — without touching how any bot works.

**If you split into separate pipelines**, the seams that cost the least are: **capture pipeline** (source → Idea Bank), **hooks pipeline** (brief → hook pool), and **copy pipeline** (chosen hooks → variants → Doc). Each boundary already exists as a data handoff in the code; today they're just welded together.

---
---

## Appendix — File Map and Key Code (for whoever implements)

### File map: where each step lives

| Step | File (all paths relative to repo root) |
|---|---|
| CLI entry + flags | `exodus/commands/genesis.ts` |
| API route (start/poll a run) | `convex/http.ts:2510-2680` (`/api/v2/genesis`) |
| Preflight, dispatch-by-input-type, run record | `convex/genesis.ts:39-539` (`startRun`) |
| Pipeline orchestrator (the 10 steps) | `scout/src/trigger/genesis-pipeline.ts` |
| Pipeline/bot definitions (dashboard step cards) | `convex/seedGenesisPipeline.ts:70-297` |
| Hook generation task (3-call routine) | `scout/src/trigger/genesis-write.ts` |
| Body copy + headlines task (3-call routine) | `scout/src/trigger/genesis-body-copy.ts` |
| Bot instruction templates (the "user prompts") | `scout/src/prompts/hook-generation.ts`, `body-copy.ts`, `headline-generation.ts` |
| Call to Luke's Genesis server (streaming + retries) | `scout/src/genesis/genesis-call.ts` |
| QA checklist + output cleanup | `scout/src/cleanup/qa-checklist.ts`, `strip-bot-junk.ts` |
| Google Doc creation | `scout/src/trigger/genesis-google-doc.ts` |
| Run completion callback | `convex/http.ts:6612-6681` (`/api/genesis/callback`) |
| Edit menu (edit-pass) | `convex/genesis.ts:742-809` + `scout/src/trigger/genesis-edit-pass.ts` |
| Idea Bank table + capture | `convex/ideaBank.ts`; capture tasks `scout/src/trigger/idea-gambit.ts`, `idea-organic.ts`, `idea-ad-library.ts` |
| Idea → Genesis escalation | `convex/ideaBankDispatch.ts:28-106` |
| Customer-facing skills (brief guidance) | `workspace/.claude/skills/write/SKILL.md`, `genesis/SKILL.md`, `idea/SKILL.md`, `swipe/SKILL.md` |
| Run page renderer | `src/.../runs/[runId]` → `genesis-output.tsx` |

Bot **system prompts** (personality/instructions for ad-hook-bot-1, mariobot, in-feed-vsl-bot, edit-pass): on the Genesis server at gas.copycoders.ai — not in this repo. The repo stores only placeholder stubs noting "system prompt managed externally" (`convex/seedGenesisPipeline.ts:26`).

### A1. The 10 pipeline steps as defined in code

`scout/src/trigger/genesis-pipeline.ts:155-166`:

```ts
const PIPELINE_STEPS = [
  { key: "seed-capture",           label: "Reading creative seeds" },
  { key: "primer-select",          label: "Selecting awareness primer" },
  { key: "hook-generation-mario",  label: "Hook gen — MarioBot voice (~20 hooks)" },
  { key: "hook-generation-infeed", label: "Hook gen — InfeedVSL voice (~20 hooks, parallel)" },
  { key: "hook-selection",         label: "Selecting N diverse hooks from pool" },
  { key: "body-copy-mario",        label: "Body copy — Mario × Brand (2 variants)" },
  { key: "body-copy-infeed",       label: "Body copy — Infeed × Brand (2 variants)" },
  { key: "body-copy-topads",       label: "Body copy — Top-Ads-Biased (2 variants)" },
  { key: "qa-checklist",           label: "Running QA checklist (all variants)" },
  { key: "doc-write",              label: "Writing Google Doc (one tab per variant)" },
];
// "edit-menu" is on-demand, not part of the auto-run sequence.
```

### A2. Variant/track planning (the variance logic)

`scout/src/trigger/genesis-pipeline.ts:214-235` — one pass = Mario + Infeed; pass 2 switches to the top-ads primer when available:

```ts
function planTracks(variantCount, hasTopAdsPrimer) {
  const plan = [];
  const passes = Math.ceil(variantCount / 2);
  for (let p = 0; p < passes; p++) {
    const useTopAds = p === 1 && hasTopAdsPrimer;
    const primerSource = useTopAds ? "top-ads" : "brand";
    plan.push({ track: useTopAds ? "topads" : "mario-brand",  botSlug: "mariobot",        primerSource });
    plan.push({ track: useTopAds ? "topads" : "infeed-brand", botSlug: "in-feed-vsl-bot", primerSource });
  }
  return plan.slice(0, variantCount);
}
```

### A3. The 3-call hook routine (how a bot is actually driven)

`scout/src/trigger/genesis-write.ts:61-125` (abridged) — prime, generate, then push for more visceral hooks:

```ts
// Call 1: prime the bot with the brand primer
const conversation = [{ role: "user", content: payload.primer }];
const primerResponse = await callGenesis({ botSlug, messages: conversation, ... });

// Call 2: replay history + the hook instruction → first batch of hooks
conversation.push({ role: "assistant", content: primerResponse.content },
                  { role: "user", content: instruction });
const hookResponse = await callGenesis({ botSlug, messages: conversation, ... });

// Call 3: replay full history + LEVEL_3_PUSH → more visceral hooks
conversation.push({ role: "assistant", content: hookResponse.content },
                  { role: "user", content: LEVEL_3_PUSH });
const level3Response = await callGenesis({ botSlug, messages: conversation, ... });

allHooks = [...extractHooks(hookResponse.content), ...extractHooks(level3Response.content)];
```

Body copy uses the same shape (`genesis-body-copy.ts:69-138`): Call 1 primer → Call 2 body copy (the chosen hook is inside the instruction) → Call 3 headlines, same conversation.

### A4. Idea → Genesis escalation (the convergence point)

`convex/ideaBankDispatch.ts:33-48` — a banked idea becomes a brief with zero transformation:

```ts
export function buildEscalationArgs(idea, opts) {
  const notes = typeof idea.notes === "string" ? idea.notes.trim() : "";
  const brief = notes
    ? `${idea.description}\n\nDirection / texture: ${notes}`
    : idea.description;                       // ← description IS the brief
  return {
    brief,
    awarenessLevel: opts.awarenessLevel,
    inputMethod: "brief",
    seeds: [idea.hook],                       // ← the captured hook rides along as a seed
    ...
  };
}
```

### A5. What the backend accepts as a brief (no schema)

`convex/genesis.ts:40-44` — the brief is just a string; and the dispatch switch that sends brief-mode runs to genesis-pipeline (and "paste" runs — saved swipes — to remix-pipeline):

```ts
args: {
  brief: v.string(),            // any non-empty text; no template, no validation
  awarenessLevel: v.string(),
  inputMethod: v.optional(v.string()),
  ...
}
// convex/genesis.ts:297 — } else if (inputMethod === "winner" || inputMethod === "paste") {
//                              → .../tasks/remix-pipeline/trigger        (saved swipes go HERE)
// convex/genesis.ts:490 — } else { // Brief mode
//                              → .../tasks/genesis-pipeline/trigger      (everything in this doc)
```

### A6. The brief guidance the skills give Claude Code (not enforced)

From `workspace/.claude/skills/genesis/SKILL.md`:

> Genesis writes from a **brief** — a 1–2 paragraph natural-language *description* of the ad, not the ad itself. If the user gave you an idea, a winning ad, or a rough angle, you write the brief from it; **surface it for a quick gut-check before running.**
>
> A brief covers: what the ad is about · what happens in it · the emotional core (heartbreak, rage, shame, mischief…) · the mechanism bridge to the product · the tone · why it's vicious. It is NOT the copy, NOT a hook list, NOT a format spec, and NOT over-specified — leave the bots room.
>
> ```
> BRIEF: "[short evocative title]"
> Hook: "[verbatim hook if from a swipe/organic source, else a hook DIRECTION or leave open]"
> Source: [where the idea came from]
> [1–2 paragraphs: scenario, emotional core, mechanism bridge, tone, why it works]
> ```
