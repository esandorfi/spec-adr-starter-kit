# Starter kit: `/spec-adr`

> **Generate an Architecture Decision Record from a git diff, versioned alongside the merge request, using a coding agent (Claude Code, OpenCode, Codex, etc.).**
>
> Minimal starting point. Adapt it to your stack.
> Field report from LightOn — DevQuest 2026 talk.

---

## Why

Architecture docs drift from the code because they live somewhere else: a wiki, Confluence, Notion. The `/spec-adr` skill brings them **back into the repo**, next to the code, at the moment the decision is made — not three months later in a retro.

The flow: a dev wraps up a branch, runs the skill, the agent reads the diff and **proposes** one or more ADRs. The dev arbitrates. The chosen ADR is committed with the merge request.

---

## Prerequisites

- A git repo
- A coding agent that supports *slash commands* (e.g. Claude Code reads `.claude/commands/*.md`)
- One `<app>/CLAUDE.md` per app you want to document
  *(or your agent's equivalent: `.cursorrules`, `AGENTS.md`, etc.)*

---

## Quickstart (5 minutes)

```bash
# 1. Create the scaffolding in your repo
mkdir -p .claude/commands
mkdir -p <your-app>/_specs/adr
touch <your-app>/CLAUDE.md

# 2. Copy the contents of "The Skill" below
#    into .claude/commands/spec-adr.md

# 3. Test on a branch with a real architectural commit
git checkout -b try/spec-adr
# ... make an architectural change ...
/spec-adr <your-app>

# 4. The agent proposes ADR candidates. You arbitrate.
#    The .md file is written to <your-app>/_specs/adr/
```

If it works on one app, generalize: one `CLAUDE.md` and one `_specs/adr/` per app.

---

## The skill (`.claude/commands/spec-adr.md`)

````markdown
---
description: Generate an Architecture Decision Record for an app
---

# ADR Generator

Create an Architecture Decision Record for the **$ARGUMENTS** app.

## When To Use This

Use `/spec-adr` when a branch contains a real architectural decision
worth preserving — not for routine implementation work.

Typical timing: near the end of a branch, before the merge request
is finalized.

Goal: capture the **why** behind a meaningful code decision.

## Step 1: Validate

1. Parse $ARGUMENTS: first word = app name, remaining = optional topic.
2. If $ARGUMENTS is empty, ask the user which app to document. Do
   not guess from the current branch — the user should be explicit.
3. <!-- ADAPT: which apps are eligible? -->
   Require <app>/CLAUDE.md to exist. If missing, stop and inform the
   user: "Create <app>/CLAUDE.md and <app>/_specs/adr/ to opt in."

## Step 2: Gather

1. Determine ADR directory: <app>/_specs/adr/
2. Determine prefix:
   - Default: first 3 uppercase letters of the app name
     (e.g. AUTH → AUT, BILLING → BIL).
   - <!-- ADAPT: if app names collide on the first 3 letters
        (auth vs authz both → AUT), maintain an explicit mapping
        below and use it instead of the default rule. -->
3. List existing ADRs to find the next sequence number (NNNN).
   Empty directory? Start at 0001.
4. <!-- ADAPT: your main branch name -->
   git fetch origin main:refs/remotes/origin/main
   git merge-base origin/main HEAD     # → BASE
   git diff <BASE>..HEAD -- <app>/     # → the diff to analyze
   git log <BASE>..HEAD -- <app>/      # → the commits scoped to this app

   Note: if the branch was rebased or stacked on a non-main branch,
   `merge-base` returns a confusing BASE. If the resulting diff
   exceeds ~2000 changed lines or covers >20 files, ask the user
   to confirm the comparison range before proceeding.

## Step 3: Identify and Propose

If a topic was provided in $ARGUMENTS, use it as focus.
If no topic, analyze the diff and commits. Look for:
  - New models or services introduced
  - Pattern choices (e.g., EAV vs JSON, sync vs async)
  - Trade-off decisions (performance vs simplicity)

Present each candidate ADR to the user with:
  - Title       — noun phrase
  - One-liner   — what decision it captures
  - Key evidence — 2-3 bullets from the diff

If the diff contains multiple distinct decisions, list them all
and let the user pick which to write (or write multiple).

If no clear decision is found, ask: "What decision should this
ADR capture?" Do NOT force an ADR for pure implementation work.

This is the only user-approval gate. Once the user confirms which
ADR(s) to write, proceed through Steps 4–7 without asking again.

## Step 4: Generate

Write the ADR in lightweight MADR format:

    # <PREFIX>NNNN: <Noun Phrase Title>

    **Date:** YYYY-MM-DD
    **Status:** Accepted
    **Branch:** `<branch-name>`

    ## Context

    2-4 sentences. What problem prompted this decision?
    What constraints existed?

    ### Evidence

    Concrete proof: counts, tracebacks, tables. Concise.

    ## Decision

    2-4 sentences. What was decided, how does it work?

    ### Metrics (optional)

    | | Before | After |
    |--|--------|-------|
    | Metric 1 | value | value |

    ## Consequences

    **Positive:**
    - Benefit 1

    **Negative:**
    - Drawback 1. Mitigation: how it is addressed.

    ## Examples

    BAD vs GOOD code patterns. 1-2 pairs max.

    ## Rules

    Codified in `<rules-file>` (section "<Section Name>").

Default Status: **Accepted**. Use `Proposal` only when the decision
is genuinely up for debate. Use `Superseded by <PREFIX>NNNN` when
an older ADR is retired — and link the new one in the old one.

## Step 5: Place Rules

<!-- ADAPT: rules organization for your team -->

Rules derived from the ADR go to ONE file:
  → Specific to one app:   <app>/CLAUDE.md
  → Cross-app:             a shared rules file (e.g. .claude/rules/<domain>.md)

Rule format = terse constraint for code generation:
  - DO/DON'T:  "Use X. Never Y."
  - WHEN/THEN: "When X, do Y."
  - WHERE:     "Put X in Y. Never in Z."

NOT rules (stays in the ADR itself):
  - Prose explaining WHY
  - Process instructions
  - ADR links

Append to an existing section that matches the rule's coding context
(e.g. "Imports", "Models", "API Views"). Don't create a new section
per ADR.

## Step 6: Verify Placement (MANDATORY)

Before writing ANY file:

1. ADR location → rule location matches.
2. No code blocks in the rules file (code → ADR, terse rules → CLAUDE.md).
3. No duplicate constraint already present in the target rules file.
4. <!-- ADAPT: your promotion criterion -->
   A rule moves from app-level to platform-level ONLY when proven
   across 2+ apps. (This is a social check, not enforced by the
   skill — the user is responsible for honoring it.)

If any check fails, fix before writing. Do NOT skip.

## Step 7: Write and Confirm

1. Save the ADR to <app>/_specs/adr/<PREFIX>NNNN-kebab-title.md
2. Add the terse rules to the target rules file.
3. Do NOT duplicate the rule text in the ADR — reference only.
4. Show the user the file paths created and a one-line summary
   of what was added to the rules file.
````

---

## What to adapt for your stack

| Block | What depends on your team |
|---|---|
| **Step 1 — eligibility** | Which apps are eligible? Which rules file do you expect? |
| **Step 2 — target branch** | `main`? `dev`? `develop`? |
| **Step 2 — prefix** | The first 3 letters of the app, or an explicit mapping if app names collide |
| **Step 4 — template** | Adjust the MADR format for your internal sections (Compliance, RACI, etc.) |
| **Step 5 — rules location** | `<app>/CLAUDE.md` for Claude Code, `.cursorrules` for Cursor, `AGENTS.md` for other agents |
| **Step 6 — promotion gate** | Promotion threshold = 2 apps? 3? Validated in weekly review? |

---

## Known limits of this v1

Worth saying out loud so you can decide whether to extend:

- **One app per invocation.** A branch that touches three apps needs three calls. Intentional — keeps each ADR scoped — but it means cross-cutting decisions split awkwardly. Consider a `--multi` mode if you hit this often.
- **No supersede flow built in.** The template supports `Status: Superseded` but the skill won't proactively detect that a new ADR contradicts an old one. That's a separate audit job (see `/spec-readme` below).
- **No enforcement of the 2-app promotion gate.** The skill *says* the rule, but nothing checks it. Treat it as a team norm, not tooling.

---

## Next step: hand off to 3 skills

Once `/spec-adr` is adopted by the team, two logical companions:

### `/mr-review` *(upstream)*
- **When**: before each `git push` of the local branch.
- **What**: challenges the diff against the `CLAUDE.md` rules. Score, blocking points, non-blocking points.
- **Effect**: less noise in peer review; juniors progress by seeing why a comment fires.

### `/spec-readme` *(downstream)*
- **When**: once a week, by a person responsible for the ritual.
- **What**: walks **all** ADRs in the codebase, generates a single markdown report (stats, ID duplicates, Proposals open for too long, ADRs without enforcement, candidate supersessions).
- **Effect**: the audit becomes a team conversation. You no longer meet to decide — you meet to review what you decided.

The three skills relay because they share **the same git context**. `/spec-adr` can explicitly delegate to `/spec-readme` — just write it in the file:

> *« This command does not maintain the README. A later `/spec-readme` audit may decide whether the app README should reflect the new ADR. »*

---

## Bootstrap your own (the prompt attitude)

If your stack differs enough from this one that line-by-line adaptation feels tedious, hand this whole gist to your coding agent with the prompt below. The agent will read the gist, ask you the right questions, and produce a `.claude/commands/spec-adr.md` (or `.cursorrules` equivalent) tailored to your setup.

> **Prompt:**
>
> I'm reading the gist `spec-adr-starter-kit.md` (attached). I want a `/spec-adr` slash command for my own coding agent, adapted to my stack. Don't write anything yet — first ask me, one question at a time:
>
> 1. What's the agent I use? (Claude Code, Cursor, Aider, other) — this determines the file location and frontmatter.
> 2. What's my main branch called? (`main`, `master`, `dev`, `develop`)
> 3. What's the directory layout? Single-app repo, monorepo with `<app>/` folders, or something else? Show me an example path.
> 4. Where do my rules live? (`CLAUDE.md` per app, a single `.cursorrules`, `AGENTS.md`, other) — and do you want app-level and shared rules separated?
> 5. ADR prefix convention — first 3 letters of app name, full app name, generic `ADR-`, or explicit mapping?
> 6. Default `Status` — `Accepted` (recommended), `Proposal`, or something else?
> 7. Anything specific your ADRs must contain that's not in the MADR template? (Compliance section, RACI, JIRA link, etc.)
> 8. Are there apps that should be excluded? (legacy, experiments, etc.)
>
> Once you have my answers, write the skill file. Show me the file in a code block; do not save it yet. After I confirm, save it to the right path and tell me which file you wrote.
>
> Ground rule: every `<!-- ADAPT: ... -->` comment in the original must be either resolved (replaced with my actual value) or kept (because it's still a per-team choice). Don't silently drop them.

This is the meta-skill: a prompt that produces your skill. If your team rewrites it twice in a row, that's the signal to fork the prompt itself — the skill that writes the skill becomes the artifact.

---

## Three tips before you start

1. **One skill at a time.** Don't try to deploy all three in parallel. `/spec-adr` is the best entry point — it's the one that produces the material to audit later.
2. **Rewrite it, don't copy it.** A copied skill doesn't live. A skill rewritten for your stack does. The `<!-- ADAPT: ... -->` comments are there for that reason.
3. **`Status: Accepted` by default.** Otherwise you accumulate Proposals that linger. Accept first, contest later if needed. This governance detail separates a system that lives from one that bogs down.

---

## Credits

Field report from LightOn — RAG platform team.

DevQuest 2026 talk — *The code changes, the docs follow: 3 skills in relay* — Emmanuel Sandorfi.
