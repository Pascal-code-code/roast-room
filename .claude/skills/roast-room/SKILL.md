---
name: roast-room
description: Prüft Ideen, Angebote, Kurse und Geschäftsentscheidungen in einem Roast Room aus unabhängigen Beratern. Nutze bei ausdrücklich gewünschtem Roast, Ideenrat oder kritischer Geschäftsprüfung. Drei Debattenrunden liefern gewichtete Optionen, Gegenargumente und den kleinsten Praxistest; optional im lokalen visuellen Raum. Unterstützt einzelne Berater und Board-Verwaltung.
---

## Local integration (takes precedence over upstream defaults)

- Repo root: the folder the Roast Room was cloned into. Installed into another project (recommended: your personal assistant project), that is `tools/roast-room` inside it; run directly in the clone, it is the current folder. Resolve all repo-relative paths there, including agent profiles, brains and sessions. The caller's working project supplies the business context.
- Answer in German, including cards and reports. Keep the parser field labels, stance/vote enums, and verdict JSON labels in English exactly as documented.
- Pick relevant seats automatically (`board: auto`); announce the selection. Ask only for missing facts that block the requested decision. If the user asks for an interview first, run it in chat, one question at a time, and wait for each answer before starting the council; never skip it or replace it with assumptions. Unknown pricing, revenue or time budgets stay unknown; use labeled scenarios, never fabricated facts or a financial verdict without the inputs.
- Use independent Sonnet agents unless the user explicitly authorizes another model. In Claude Code explicitly set `model: sonnet`. If the host agent tool does not offer Sonnet, use the installed Claude CLI with `--model sonnet`; `scripts/run_cli_round.py` supplies isolated, tool-free seats with the selected brains and session context, at most three in parallel. If Sonnet is unavailable, report the concrete failure and ask once before changing model; an authorized override persists for that task. Never silently claim a sequential role-play was independent agents.
- Read the user's current canonical artifact and latest decisions. Separate verified facts, simulated buyer reactions, assumptions, and tests. A familiar expert's perspective is not customer validation; its score is not a success probability.
- Preserve the old roast's useful output: German decision, strongest objection, concrete opportunity costs, and the smallest useful 48-hour test. Do not manufacture prices or promise a test validates retention within 48 hours.
- When the decision concerns an existing plan, offer or course, present concrete proposed changes in chat and verify them together before modifying it. “Possibly refine afterwards” is not approval to insert recommendations, even as drafts; wait for explicit approval of the changes.
- A critique does not authorize sending messages, publishing, or changing commercial terms. When refinement is requested, use the existing canonical artifact and preserve settled user choices unless the user changes them. Mark substantive new strategy as a proposal.
- Run the local viewer with `npm start` here. It binds to loopback. Do not enable the optional API runner or add credentials for a CLI roast.
- In Round 3 repeat `CONFIDENCE`, `RUIN RISK`, and `DOOR` from the updated position: upstream scoring needs these final-round values, although its abbreviated Round 3 prompt omits them.
- Upstream source and pinned revision are in `references/upstream.json`. Before an update preserve local changes and user-created seats; do not overwrite this integration blindly.

## What this does

The Roast Room: runs a board meeting on one decision. A board of seats (seven out of the box), one per business pillar, each a sub-agent that reasons and talks the way its expert does, grounded in a brain compiled from that person's videos, essays, letters, talks, and books as they teach them in public. The seats argue across three rounds: open, debate, vote. You act as the **Chair**: you never argue a side, you run the room, keep the score, and write the verdict.

Read once before the first run: `roster.json` (every seat and the default board), `references/seats.md` (picking the board, weights, tensions), `references/round-prompts.md` (what each seat gets), `references/scoring.md` (math and overrides), `references/report-template.md` (output), `references/live-room.md` (the small JSON files the visual room plays). A finished example council lives in `sessions/2026-10-09-example-take-the-seed/`.

## Modes

- **Full council** (default): a board recommended for this decision and confirmed by the user (see "Pick the board"), three rounds. For decisions that are expensive or hard to undo.
- **Quick** (`--quick`): four seats, recommended per "Pick the board" and `seats.md`, Rounds 1 and 2, Round 3 only if the scores did not converge. For reversible calls.
- **Named seats** (`--seats a,b,c`): only those seats, three rounds. Any seat in `roster.json` can be named, on the default board or not.
- **Everyone** (`--all`): every seat in `roster.json`, up to nine.
- **Consult** (`ask <seat> ...`): one seat, one answer, using the single-seat prompt. No scoring, no session folder needed. Works for any seat in the roster.
- **Board settings** (`board`): show and change who sits at the table by default. See "Board settings" below. No council runs.

## Pick the board

Who sits at the table comes from `roster.json`, in this order:

1. `--seats a,b,c` or `--all` on this run wins. No question. `--quick` then picks four from those seats.
2. Otherwise `"board"` in `roster.json` decides:
   - `"ask"` (out of the box): recommend a board for this decision and ask the user to confirm it (below).
   - `"auto"`: recommend a board the same way and seat it without asking. Say who and why in one line.
   - `"default"`: seat the `"default"` list. No question.
3. Never seat more than nine. Every seat at the table must have an agent file (`.claude/agents/council-<key>.md`) and a brain folder; skip and name any that do not.

### Recommend a board

Read the decision, then pick from every seat in `roster.json`:

- Every seat whose pillar the decision lives in or materially affects (the seats that would get weight 2 or 3).
- One seat that guards the downside (Buffett, or the roster's money, capital, or risk seat), whether or not it is on the default board.
- The seat most likely to disagree with the rest, so the council argues instead of agreeing.
- Leave out seats whose pillar barely touches the decision. Two seats with close pillars (Martell and Leila Hormozi on people, Hormozi and Codie Sanchez on deals) sit together only when they would disagree.
- Five to seven seats for a full council, four for `--quick`.

### Ask

Show the recommendation in chat first, as a short list: each recommended seat with one plain line on why it fits this decision, then one line naming the seats left out. Then ask with one AskUserQuestion question (header "Board"):

1. **Recommended board (Recommended)**: the names, in the description.
2. **Default board**: the `"default"` list, by name.
3. **Everyone**: every seat in the roster (only when there are nine or fewer; otherwise skip this option).

The user can pick "Other" and type names to swap in or out ("recommended, but add Naval and drop Wickman"). Seat what they chose. When Step 1 also needs to ask about missing facts, put the board question in the same AskUserQuestion call so the user answers once. If the user said "just run it", seat the recommended board without asking.

Tell the user who is at the table in one line when the council starts, and that `--seats`, `--all`, or `/roast-room board` change it.

## Step 1: The brief

The council is only as good as the brief. Build it from what the user gave you. If any of the first four items is missing, ask for them in one batch (AskUserQuestion, at most four questions), then proceed. If the user says "just run it", fill gaps with stated assumptions.

1. **The decision** in one line, and the options on the table as A, B, C. Always include "don't do it" (or "keep doing what you do now") and any alternative the founder mentioned. If the founder brought one idea, add the obvious smaller version as an option.
2. **The business**: what it sells, to whom, stage, monthly revenue, margin if known, team size.
3. **The resources**: cash or runway, founder hours, deadline.
4. **What success looks like** and the time horizon.
5. **Which way the founder is leaning, and why** (the council tests the lean, it does not echo it).

Context lookup: if this project holds notes about the founder's business (a `CLAUDE.md`, a `business.md`, or similar), read them for real numbers before asking.

Keep the brief under 300 words. Mark every assumption as `(assumed)`.

## Step 2: Set up the room

- Seat the board (see "Pick the board"). Set a relevance weight per seat at the table (1 to 3) with a one-line reason, per `seats.md`.
- Create the session folder: `sessions/YYYY-MM-DD-<decision-slug>/` at the project root. Save `brief.md` (the brief, the options, and the weights table) and `council.json` (`live-room.md`).
- Tell the user once: "Watch it live at http://localhost:4730" (the Roast Room plays the council as these files land; it is optional and the council runs the same without it).

## Step 3: Round 1, opening positions

Spawn every seat in **one message**, in parallel, foreground (`run_in_background: false`), `subagent_type: council-<seat>`, with the Round 1 prompt. If a `council-*` type is not registered in this session, use `general-purpose` with the fallback lines from `round-prompts.md`.

When the cards come back, save them verbatim to `round-1.md`, one `## <Seat>` heading per card (the seat's `short` name from `roster.json`: Bezos, Graham, Hormozi, Gary V, and so on), and write the file right away so the room can start playing. Check each card has every field; send a card back once if it is missing scores or a stance. Then add a `## Chair notes` section at the bottom of `round-1.md`:

- The **cruxes**: the one to three underlying disagreements that explain why seats split (a factual question, a numbers question, or a values question such as speed versus safety), with the seats on each side.
- The **unanimity check** (`scoring.md`): if Round 1 converged, make the strongest unargued alternative a crux and name which seats must argue it at full strength.
- Any card that broke format or misread the brief.

Then write `cruxes.json` (`live-room.md`).

## Step 4: Round 2, debate

Spawn every seat again, in one message, with the Round 2 prompt (it points them at `brief.md` and `round-1.md`). Save the cards to `round-2.md` (same `## <Seat>` headings, written as soon as the cards are back) with `## Chair notes`: who moved and why, which cruxes resolved, which numbers are still open, and the interim option scores.

## Step 5: Draft resolution

Write `draft-resolution.md`: the recommendation (named option and verdict label), the conditions the seats asked for (merge duplicates, keep them checkable, pick one number when seats differ and say whose), the next steps with dates, the cheapest test, and kill criteria. Weigh arguments, not headcount: a single seat with a decisive fact beats three seats with vibes. Apply the overrides. Also write `draft.json` (`live-room.md`).

## Step 6: Round 3, ratification

Spawn every seat with the Round 3 prompt. Save to `round-3.md` (same headings, written as soon as the cards are back), and add a `## Chair` section listing each amendment and whether it was adopted, with the reason. Adopt an amendment when no seat objects or the argument for it went unanswered; when seats conflict, adopt the better-argued side and put the other in the minority report. Do not run a fourth round.

Quick mode skips this round when the Round 2 spread on the top option is 3 or less and no seat is at `RUIN RISK: likely`.

## Step 7: Verify, then verdict

1. Run `python .claude/skills/roast-room/scripts/check_quotes.py <session folder>` (try `python3` if `python` is missing). Every quote attributed to a source id must be in a brain. Remove any unverified quote from the verdict (and say so in the Chair notes). If Python is not installed, do not stop or ask the user to install it: check each attributed quote yourself by searching that seat's brain folder for the exact words, and treat any you cannot find as unverified.
2. Compute the numbers from the last round's cards per `scoring.md`, apply the overrides, and write `verdict.md` from `report-template.md`. Write `verdict.json` last (`live-room.md`); it tells the room the council is over.
3. Show the user a short verdict in chat, not the full report: the label and the call, the option scores on one line, the three arguments that carried the room (one line each, with the seat), the strongest objection (one line, with the seat), and the next steps (at most three). Then the path to `verdict.md` for the full detail, one line saying the branded PDF is one click away ("Download PDF" on the verdict at http://localhost:4730), and one line offering a follow-up with any seat or a rerun with new facts. Keep it under 200 words: a founder should be able to decide from it at a glance.

## Chair rules

- You never argue a side and never add your own pillar. If no seat raised an important point, put it to the closest seat as a crux in the next round instead of inserting it yourself.
- Quote seats faithfully. Never put words in a seat's mouth or rewrite a card.
- Seats speak in their expert's voice; your report is plain English that follows the house writing rules (no em dashes, no AI tells, no tee-ups).
- Treat a unanimous Round 1 as a warning, not a win.
- Cost: a full council is three sub-agent runs per seat (21 for seven seats), about 1M tokens. Use `--quick` for small calls, and say which mode you used.
- The seats are simulations built from public work. Never present a seat's line as something the real person said unless it is a verbatim quote with a source id from the brain.

## Board settings

`/roast-room board` (or "change my board", "who's on my board", "add Naval to my default board") edits `roster.json` and runs no council.

1. Show the roster in a short table: seat, pillar, on the default board or not. Then the current `"board"` setting in plain words ("the Chair recommends a board for each decision and asks you", "the Chair picks per decision without asking", or "every council seats your default board").
2. Ask what to change, with AskUserQuestion when the user has not already said: add or remove seats on the default board, or switch the setting between `ask`, `auto`, and `default`.
3. Edit `roster.json` in place: only the `"board"` and `"default"` fields. Keep the default board between one and nine seats, using keys that exist in `"seats"`. Never remove a seat from `"seats"` here; taking someone off the default board keeps them available for `--seats` and `ask`.
4. Confirm the new board in one line. The room picks it up on its next page load.

To add a brand new advisor, read `.claude/skills/newroaster/SKILL.md` from the repo root and follow it (or invoke `/newroaster` when registered).

## Without sub-agents (Codex or a runtime that cannot spawn agents)

Run the same protocol sequentially yourself. For each seat, read `.claude/agents/council-<seat>.md` and its brain, write that seat's card, then clear your head before the next seat: do not let one seat's card shape the next one's in Round 1. Rounds 2 and 3 work the same way with the prior cards in view.

## Files

- `brains/<slug>/`: one brain per seat (`_hot.md`, `frameworks.md`, `principles.md`, `positions.md`, `voice.md`, `stories.md`, `books.md`, `sources.md`).
- `scripts/check_quotes.py`: verifies every attributed quote in a session against the brains.
- `roster.json`: every seat (key, name, pillar, the question it owns, scorecard dimension, color) and the default board. `/newroaster` adds seats; `/roast-room board` changes the board.
- Agents: `.claude/agents/council-<key>.md`, one per seat in the roster.
- The room app: `app/` (start at the project `README.md`).
