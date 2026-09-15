---
name: pr-comment
description: Write and post inline review comments on a GitHub PR. Use when asked to comment on a PR, raise a review finding, or post PR feedback — including the handoff from pr-review after its draft is approved. Handles anchor math, safe transport, and read-back verification.
---

# PR comment

Posts findings as **inline-anchored** comments on a GitHub PR — one per finding, on its
line. Usually invoked by `pr-review` after its draft is approved, but works standalone.

The machinery here — anchoring, transport, read-back — exists because comments fail
silently: a wrong anchor lands on an unrelated line, a heredoc eats characters, a 201
response means "accepted", not "correct". Every rule below is a guard against a failure
that actually happened.

## Register — lean, or don't post

Reader is a senior engineer who has the diff open. Keep every comment to **what, why,
suggested fix** — nothing else.

- Finding, then evidence, then fix. No preamble, no recap of the PR.
- Bullets and code blocks. Never a paragraph where a diff will do.
- Every claim carries a file name and a quoted fragment. Verify before writing.
- No praise, no hedging, no "happy to help". Suggest; don't rule on open decisions.
- Don't correct the author's prose or restate what their PR changes — they know.

Drop any part you can't fill honestly. What + fix is a complete comment; why is often
carried by the quoted evidence itself. If a pasted command output already proves the
point, don't also narrate it in prose.

## Quoting

| Quoted | Form |
|---|---|
| A line or more of source, config, or command output | Fenced block |
| An identifier, path, flag, value, file name | Inline backticks |

Inline backticks wrap mid-token in the GitHub UI, so a quoted *line* renders as prose and
stops looking like code. Quote the smallest fragment that carries the claim.

## Name the file, never the line

Body prose names the file and stops there.

- The comment is already anchored to a line; a second number in the prose adds nothing.
- One fixup push and every line number in the body is wrong, with nothing marking it stale.

Line numbers belong only where they're mechanical — the anchor payload, and the `@@`
arithmetic that computes it. Never in a sentence.

## Anchor the block, not the token

When the finding is about a construct rather than one token, anchor the whole construct:
`start_line` at its first line, `line` at its last, both sides `RIGHT`. GitHub highlights
the range, so the reader sees the shape without scrolling. Both ends must be in the same
hunk of the same file — otherwise use a single-line anchor.

## Explaining a flow

Anything that happens **over time** — a race, a lifecycle, a retry loop, before/after
state — goes in a fenced ASCII diagram, not a table. A table implies its rows are
order-free; a sequence's whole point is that they aren't.

Time runs **down**, actors run **across** — one column per actor, plus one for the state
they mutate. An arrow that crosses columns reaches the column it affects.

```
      REQUEST A                  REQUEST B                  DISK
      ---------------------      ---------------------      ------------------
 t1   read {a:1}                                            {a:1}
 t2                              read {a:1}                 {a:1}
 t3   WRITE --------------------------------------------->  {a:1, b:2}
 t4                              DELETE ------------------>  (gone)
```

Two mechanics, or it renders crooked:

1. **Emoji occupy two cells.** Pad by display width, not character count, and assert every
   row ends its last column at the same offset.
2. **No variation-selector emoji** (`✍️` `🗑️` `⚠️` carry `U+FE0F`, render 1–2 cells
   depending on client). Use unambiguous double-width codepoints — `📖 🧠 💾 🔥 💥 📝`.

Build the block in a script that computes the padding, not by eye.

## Draft first

Draft as **message text in the TUI**. Get approval. Then post. (When invoked from
`pr-review`, its draft already went through this — you're posting an approved finding, so
the draft stage is the review's, not a second one.)

## Find the anchor — it must be a line in the diff

```bash
gh pr view <n> --json headRefOid --jq .headRefOid
gh api --paginate 'repos/doshii-io/<repo>/pulls/<n>/files?per_page=100' \
  --jq '.[] | select(.filename=="<file>") | .patch'
```

Count from the hunk header `@@ -a,b +c,d @@`, then confirm against the head commit — local
annotations, stashes, and uncommitted edits all shift line numbers:

```bash
git show <headRefOid>:<file> | sed -n '<start>,<end>p'
```

| Anchoring to | `side` | Line number |
|---|---|---|
| Added or context line | `RIGHT` | Post-change |
| **Deleted** line | `LEFT` | Pre-change — read it from `git show <base>:<file>` |
| A range | both `RIGHT` | `start_line` + `line`, both post-change |

A deleted line is the correct anchor when the deletion is the cause. `RIGHT` with a
pre-change number either 422s or lands on an unrelated line. `gh pr diff` returns HTTP 406
above 20,000 lines — use the files API.

## Never build the body inside a shell command

A quoted heredoc has silently eaten characters mid-body, invisible until the comment was
read back. Write the approved body with the `Write` tool, then pass it **by path**:

```bash
jq -n --rawfile b /path/to/body.md \
  '{body:$b, commit_id:"<headRefOid>", path:"<file>", line:<n>, side:"RIGHT"}' > "$TMPDIR/c.json"
gh api --method POST 'repos/doshii-io/<repo>/pulls/<n>/comments' --input "$TMPDIR/c.json" \
  --jq '{html_url, path, line, side}'
```

No heredoc. No `jq -Rs` from stdin. No `--arg body "…"`. The body never appears in a
command string. Scratchpad only, never the repo.

`gh pr review` and `--comment` create a review *verdict* — don't use them for a finding.

## Read it back — posting is not delivering

```bash
gh api repos/doshii-io/<repo>/pulls/comments/<id> --jq .body | tr -d '\r' > "$TMPDIR/posted.txt"
diff /path/to/body.md "$TMPDIR/posted.txt"
```

One trailing-newline difference is expected; GitHub appends one. Anything else is
corruption — `PATCH` the same endpoint with a corrected `{body}` and diff again.

Never report a comment as posted on the strength of the POST returning 201. Report it on
the strength of the diff.

## Posting multiple findings

One inline comment per finding, each anchored to its own line and leading with its own file
name. Post the most severe first. If scattering comments across the PR would be noisier than
one place — a handful of findings all in one file, say — offer the user a single summary
comment instead, but inline-anchored is the default.
