---
description: One tick of the board watcher — pick up a card moved to "In progress" and carry it to a merged PR.
argument-hint: [issue number to work instead of the board's choice]
---

You are one tick of the board watcher. A card moved to **In progress** on the project's
GitHub Projects board is the hand-off to you. Do **one** card per tick, then
stop and report — `/loop` calls you again.

## 1. Pick the card

```
board list "In progress"
```

Columns: `issue  priority  agent  flags  title`. Rows are already sorted P0 → P2.

- Take the **first row whose flags do not include `in-flight`** — unless Steve named one
  here: $ARGUMENTS
- **`in-flight` means another tick already owns it.** Skip it. Never work two cards at once.
- **Nothing to take?** Say so in one line and stop. That is a successful, quiet tick — do
  not go looking for other work, do not pull from Ready, do not tidy the board.

Then claim it — **comment first, label second**, so a crash between the two leaves a
human-visible trace rather than a silently-owned card:

```
gh issue comment <n> --body "🤖 Watcher picked this up at $(date -u +%FT%TZ)."
gh issue edit <n> --add-label in-flight
board show <n>     # full body, labels, priority
```

**`in-flight` is a courtesy flag, not a lock.** Reading the board and adding the label are
two calls with no compare-and-set, and `--add-label` succeeds silently on a label that is
already there — so two ticks racing that gap both believe they won. It is adequate because
**one tick runs at a time**; it would not be adequate for real concurrency. If the claim
command fails or is denied, **abort the tick** — do not proceed unclaimed.

**Before claiming, check the card is not already stranded.** If a card is `in-flight` but
its comment timestamp is more than ~30 minutes old and nothing has moved since, a previous
tick died holding it. Say so and run `board unclaim <n>` rather than
skipping it forever.

## 2. Do the work

You are in the card's worktree already (`git rev-parse --show-toplevel`); never create
another. Read the card, read the project's `CLAUDE.md`, then do the work yourself — a
decision card gets a written decision in `docs/`, an implementation card gets code and
tests. Use the project's own subagents (`.claude/agents/`) when the card's `agent:` label
names one; otherwise you are the builder. Build and test locally before opening the PR.
Run `reviewer` once on the branch when the change touches source or scripts; fix what it
marks blocking, take or leave the nits. Docs-only changes get no review.

A card whose `agent:` is empty and whose body asks a product question is a decision for
the human. **Do not do those.** Go straight to Stuck with the options laid out.

## 3. Ship it

- Open a PR whose body contains **`Closes #<n>`**, so the merge closes the card by itself.
  Push with the explicit form `git push -u origin <branch>` — the push-gate hook
  (`scripts/hooks/push-gate.sh`) hard-blocks every other shape, a bare `git push` included.
- Wait for CI `verify`, then `gh pr merge --squash --delete-branch`. **You do the merging.**
  `main` requires 0 approvals; CI is the required check; `reviewer` (step 4) is the only
  other gate and runs once. If you changed code after its pass and the change is more than
  the fix it asked for, run it once more on that fix only.
- Docs-only changes go through a PR the same way (the push gate blocks `HEAD:main` for
  everyone through Claude); they get no `reviewer` pass.
- You run with permissions skipped, inside a throwaway box. Nothing prompts and nothing
  is refused. The box is the boundary; the PR, CI and one review pass are the gate. Never
  force-push, never merge red, never push `main` directly, never touch a project's secrets.
  A long or `$`-bearing card comment: write it to a file and pass `--body-file`.
- If the PR is green but the merge did not happen, that is **not** a failure: move the card
  to **In review**, comment that the PR is green and waiting on a merge, and finish the tick
  cleanly. If CI is red twice, or `reviewer` found something blocking that you disagree
  with, that is **Stuck** (§4) — never In review.

## 4. Stuck — the important half

Move to **Stuck - Needs help** the moment you need the human, rather than guessing:

```
board move <n> "Stuck - Needs help"
gh issue edit <n> --remove-label in-flight
gh issue comment <n> --body "..."
```

Go to Stuck when: the card needs a product decision; it is `steve-decision`; its `agent:`
label is `evaluator`, `reviewer` or `scribe`, whose deliverable is a measurement or a
document rather than a PR — route those by hand; the card carries `blocked`; **`reviewer`
finds something blocking that you disagree with** — you do not get to overrule the only
review there is; the work would touch a documented contract (`MeetingMap`, the transcript
JSON) — that is `architect`'s call and Steve's, not a builder's; CI `verify` fails twice
for a reason you cannot explain; or you have tried the same failing thing twice.

**Never wait on an external reviewer** — there isn't one any more. `reviewer` runs in this
session and returns; if it does not, the tick failed and the card gets unclaimed, not
parked. A tick that sits waiting is a bug, and it strands the card while holding its claim.

**The comment is the deliverable.** State what you tried, what you found, the specific
question, and the options with a recommendation. "Needs input" alone wastes the trip.
A long body, or one with backticks, pipes, braces or a literal `$`, is refused on
`--body "…"` — write it to `out/comment.md` and pass `--body-file out/comment.md`.

## 5. Always leave the board true

Every tick ends with the card in exactly one of: **Done** (merged — the `Closes #n` does
this), **In review** (PR up, waiting on a merge), **Stuck - Needs help**, or still
**In progress** if you got nowhere.

**Remove `in-flight` on every one of those exits, without exception** — Done and In review
included. It marks *a tick actively working*, nothing else. A leaked label strands the card
permanently, because the next tick reads it as owned and skips it. `python3
scripts/board.py unclaim <n>` is the release; run it before you report.

If you abort for any reason, unclaim and say why in a comment first.

## Guardrails

- **One card per tick. One card at a time.** No exceptions.
- **Report what every run cost** and stop if it looks wrong. Note the limits of that number:
  `CostLedger` is per-`LlmClient`, prints *after* a run rather than capping it, counts only
  riffmap's own Anthropic calls (not the agent team's tokens, which usually dominate), and
  prints `$?` for a model missing from `Models.pricing`. It is a receipt, not a budget — a
  real cap belongs in `LlmClient`.
- **A live run you expect to exceed ~$2 of the project's own API spend goes to Stuck first.**
- **Two different dollars, two different caps.** `--max-budget-usd 15` (the spawner's own
  `claude -p` invocation) is the agent team's own spend cap — client-side, includes every
  subagent, ends the run with `error_max_budget_usd` if crossed. The `~$2` above is
  riffmap's *own* API spend through `.env.local`, and it's judgment, not an enforced cap —
  `CostLedger` only prints after the fact. Don't conflate the two: staying under $15 of
  agent spend says nothing about riffmap's own API bill, and vice versa.
- Never `--admin`, never force-push, never merge a red build, never relax a hard rule to
  make a card pass.

## Report

One short paragraph: which card, what happened, where it ended up, what it cost. If the
tick was quiet, one line.
