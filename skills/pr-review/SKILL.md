---
name: pr-review
description: Review a GitHub PR diff across three axes — correctness, standards, and spec — with adversarial verification, and draft findings for approval. Use when asked to review a pull request, audit a PR diff for bugs, check a PR before approving, or produce review findings a reviewer can act on. Chains into pr-comment to post them.
---

# PR review

Three-axis review of a GitHub PR diff. Correctness-led. Every axis runs in its own
sub-agent so the main context stays clean, and every surviving finding is refuted by an
independent verifier before it is written down.

The draft is shown in the TUI for approval. Posting is a **separate, explicit** step —
this skill never posts. When you approve, hand off to `pr-comment`.

**Assumes** GitHub + `gh`. The Atlassian MCP (`getJiraIssue`) supplies ticket intent.

## 0. PROJECT_DIR gate — run before anything else

```
PROJECT_DIR: /Users/jay.requizo/projects
```

`PROJECT_DIR` is the directory where sibling Doshii repos are cloned (the one that also
holds `doshii-docker-environments`). The line above is the source of truth — this skill
edits it in place on first run.

If it reads `<unset>`, **stop. Do not review.** Configure it first:

1. Scan likely roots for a directory containing `doshii-docker-environments`:
   ```bash
   for d in ~/projects ~/dev ~/code ~/src ~/work "$HOME"; do
     [ -d "$d/doshii-docker-environments" ] && echo "$d"
   done
   ```
2. Show the candidate(s) and ask the user to confirm the root (or supply one).
3. Rewrite the `PROJECT_DIR:` line in **this file** with the confirmed absolute path,
   using the `Edit` tool. Confirm the write.
4. Only then continue to the review.

Once set, `PROJECT_DIR` is where cross-repo reads and any local reproduction happen (§4).

## 1. Load intent

Before reviewing, gather what the PR is *for* — this feeds the Spec axis and grounds the
other two:

- PR body and title, existing review comments, the commit list.
- The Jira ticket: derive the `<KEY>` from the branch name or PR body
  (`<PROJECT>-<NUMBER>`), fetch it with the Atlassian MCP `getJiraIssue`. No key → note it;
  the PR body becomes the spec, at reduced confidence.
- Repo conventions: `cat CLAUDE.md AGENTS.md .github/copilot-instructions.md 2>/dev/null`.
  Anything a human already raised in review comments is out of scope — don't re-find it.

Get the real changed surface — headline diff size lies, lockfiles dominate it:

```bash
gh api --paginate 'repos/<owner>/<repo>/pulls/<n>/files?per_page=100' \
  --jq '.[] | select(.filename | test("lock|\\.snap$") | not) | "\(.changes)\t\(.filename)"'
```

An empty file list is a broken command, never "no changes".

## 2. Spec discovery is interactive — never skipped

The Spec axis reviews against what the PR was *asked* to do. Tickets are usually sparse,
so the spec is **discovered, not assumed**:

1. Infer the intended spec from the Jira ticket + PR description.
2. **Grill the user up front** to fill the gaps the ticket leaves — use the `grilling`
   skill's design-tree method: settle the frontier of open spec questions in rounds, each
   with a recommended answer, before the Spec sub-agent runs.
3. The Spec sub-agent may surface *further* ambiguities as it reviews; bring those back to
   the user rather than guessing.

Settle the spec, then review against it. Don't review against a moving target.

## 3. Three axes, three parallel sub-agents

Spawn all three concurrently. Each takes the diff command, the changed-file list, and its
own sources. Each accepts a configurable **effort** (low → max) — spend it where being
wrong is expensive (correctness, security), not on cosmetic axes.

| Axis | Looks for | Sources |
|---|---|---|
| **Correctness** | Data loss, outage, auth bypass, silently-wrong money, plain bugs | The diff, plus cross-repo reads (§4) |
| **Standards** | Deviations from the engineering standard + code smells | `~/.claude/engineering-standards.md` (baseline), repo-local `CLAUDE.md`/`AGENTS.md`/`.github/copilot-instructions.md` (**override the baseline**), Fowler smell baseline (below) |
| **Spec** | Missing/partial requirements, scope creep, requirements implemented wrong | The settled spec from §2 |

Read the diff yourself in each sub-agent. Do **not** depend on any external `/code-review`
skill — this skill is self-contained.

### Standards: the Fowler smell baseline

Applies on top of the documented standard, even when a repo documents nothing. Two rules
bind it: **the repo overrides** (a documented standard always wins), and **every smell is a
judgement call** ("possible Feature Envy"), never a hard violation. Skip anything tooling
already enforces.

- **Mysterious Name** — name doesn't reveal what it does/holds → rename; no honest name = murky design.
- **Duplicated Code** — same logic shape in >1 hunk/file → extract, call from both.
- **Feature Envy** — a method reaches into another object's data more than its own → move it onto that data.
- **Data Clumps** — the same few fields/params keep travelling together → bundle into one type.
- **Primitive Obsession** — a primitive standing in for a domain concept → give the concept its own type.
- **Repeated Switches** — same `switch`/`if`-cascade on the same type recurs → polymorphism, or one shared map.
- **Shotgun Surgery** — one change forces scattered edits across many files → gather what changes together.
- **Divergent Change** — one module edited for several unrelated reasons → split by reason.
- **Speculative Generality** — abstraction/hooks for needs the spec doesn't have → delete, inline back.
- **Message Chains** — long `a.b().c().d()` the caller shouldn't depend on → hide behind one method.
- **Middle Man** — a class/function that mostly just delegates → cut it, call the target direct.
- **Refused Bequest** — a subclass ignoring most of what it inherits → composition over inheritance.

Standards findings also carry the engineering standard's own review lens: did the change
**deepen a module boundary or just spread complexity**? Did it **reduce or increase what a
future maintainer must understand**? Was a pattern used because needed, or because familiar?

## 4. PROJECT_DIR — cross-repo reads and local reproduction

Sub-agents may **read** sibling repos under `PROJECT_DIR` to check things the diff alone
can't show: callers in dependent services, shared contracts, breakage in downstream repos.
Read-only by default.

A verifier (§5) may **propose** spinning up `doshii-docker-environments` to reproduce a
specific claim locally — but must **ask the user before running anything**. Local runs are
slow and side-effectful; gate them behind approval, reserve them for a blocker worth
reproducing, and say what was reproduced (vs. reasoned) in the finding.

## 5. Adversarial verification — one refuter per candidate

Every candidate that survives its axis gets **one independent verifier sub-agent** whose
job is to **refute** it: find the caller that makes it unreachable, the guard that already
handles it, the framework wiring that keeps it live. Default posture: **refuted if
uncertain.** Only survivors reach the draft.

- Reachability is "does this run once the PR merges", not "does the diff add a caller".
  Framework wiring counts (autoload, DI, decorators, convention routing) — no diff line,
  still live. Genuinely unwired code is dropped; can't tell → it's a `NOT VERIFIED` line.
- Contradicting a trade-off the author **already states in the PR body** is a Should fix
  worded "your framing understates X" — never a Blocker.

## 6. The draft — shown in the TUI, never written to a file by default

```
<one line: findings per axis + the single worst issue>

### Blockers
- [<axis>] <file> — <claim, ≤ 40 words>
  <≤ 5 quoted lines, fenced>
  Trigger: <input or state>   Fix: <one line>

### Should fix
- [<axis>] <file> — <claim, ≤ 80 words, one code quote>
  Fix: <one line>

### NOT VERIFIED
- <what you could not check, and why>
```

- `<axis>` is one of `correctness` / `standards` / `spec`.
- **Blockers**: data loss, outage, auth bypass, or silently-wrong money. 0 is normal.
- Name the **file**, quote the fragment — never a line number in prose (it goes stale on
  the next push; the anchor holds the position). Quoted source in fenced blocks, never
  inline backticks.
- Lean and concise throughout: **what, why, suggested fix** — nothing else. No praise, no
  hedging, no recap of what the author's PR changes. A blocker needing >150 words is a
  conversation: post the gist, offer to walk through it.
- Delete empty sections rather than writing "(none)". Zero findings is a valid review —
  don't manufacture a fifth thing.

Show this as **message text in the TUI**. Never save it to a file unprompted. Scratchpad
files stay fine for genuine intermediates (fetched lockfiles, parsing scripts, oversized
output).

## 7. Handoff to pr-comment

Chaining is **explicit**. After the user approves the draft, invoke **`pr-comment`** to post
the findings as **inline-anchored comments**, one per finding on its line. Pass it the
approved findings — each with its file, quoted fragment, severity, and axis. `pr-comment`
owns all anchor math, transport safety, and read-back verification.

Do not post from this skill. Do not chain automatically.
