---
name: council
description: Convene a council of model delegates (Claude Opus 5.5 and Claude Fable 5.1) to independently answer, review, or critique the same request, then synthesize their findings, disagreements, and next steps. For code reviews, score each round out of 10 and iterate fixes until every delegate is satisfied, without scope creep. Use when the user says "council", "ask the council", "get a second opinion", "council review", or wants multiple models to weigh in.
---

# council

Send the same request to several model delegates in parallel, collect their independent answers, and act as chair. The chair gathers context, synthesizes what the delegates said, decides which recommendations to adopt, and keeps scope tight.

## Delegates

| Delegate | Model ID |
| --- | --- |
| Claude Opus 5.5 | `claude-opus-5-5` |
| Claude Fable 5.1 | `claude-fable-5-1` |

To add or remove a delegate, edit this table. Everything below applies to every delegate listed here.

## When to use

- The user says "council", "ask the council", "convene the council", or "council review".
- The user wants a second opinion or multiple models to weigh in on a plan, design, decision, or piece of code.
- The user asks for a multi-model code review before or after opening a pull request.

## Instructions

### 1. Gather context (chair)

- Restate the request in one or two sentences, and decide whether it is a **question/plan** (answer and recommend) or a **review** (critique existing work).
- Collect the relevant context so delegates don't start from zero: the user's request, relevant file paths, the diff (`but diff`, or `gh pr diff <n>` for an open PR), constraints, and anything already decided.
- **Write down the scope** explicitly, e.g. "Scope: the retry logic added in `src/client.ts`. Out of scope: the logging refactor." This line goes into every delegate prompt and every later round.

### 2. Dispatch delegates in parallel

Give every delegate the **same prompt** and dispatch them all at once so they can't see each other's answers.

- **Preferred:** use your harness's subagent/Task tool with each delegate's model set to the model ID from the table.
- **Fallback:** if the harness can't pick a model per subagent, shell out to the Claude CLI in read-only mode, one background process per delegate:

  ```bash
  claude -p --model claude-opus-5-5  --permission-mode plan \
    --disallowedTools "Edit" "Write" "NotebookEdit" < /tmp/council-prompt.md > /tmp/council-opus.md &
  claude -p --model claude-fable-5-1 --permission-mode plan \
    --disallowedTools "Edit" "Write" "NotebookEdit" < /tmp/council-prompt.md > /tmp/council-fable.md &
  wait
  ```

- **Read-only for reviews and feedback:** delegates must not edit files, commit, push, or comment on PRs. Say this in the prompt and enforce it with tool restrictions where possible. Only the chair makes changes.

Delegate prompt template:

```markdown
You are a delegate on a review council. Work independently.

## Request
<the user's request, restated>

## Scope
In scope: <...>
Out of scope: <...>  Do not recommend changes outside the scope.

## Context
<file paths, diff, constraints, prior decisions>

## Rules
- READ-ONLY. Do not edit files, run git write commands, or post anything.
- Be concrete: cite file:line, and separate blocking issues from nice-to-haves.

## Output format
1. Summary (2-3 sentences)
2. Blocking issues (must fix before you'd approve)
3. Non-blocking suggestions
4. Score: N/10   <- code reviews only; final line, exactly this format
5. Verdict: SATISFIED or NOT SATISFIED   <- code reviews only
```

If a delegate fails or times out, retry once. If it fails again, record it as unavailable for that round and continue with the rest. Never fabricate its response.

### 3. Synthesize (chair)

Report back to the user in this shape:

- **Consensus findings:** points raised by more than one delegate.
- **Unique findings:** useful points only one delegate raised, credited to that delegate.
- **Disagreements:** where delegates conflict, what each said, and your call with a one-line reason.
- **Recommended next steps:** what you plan to do, in order.
- **Scores** (code reviews only): one line per delegate.

Judge findings on their merits, not by vote count. A single correct blocking finding outweighs two delegates who missed it.

### 4. Code review rounds

- **Round:** one full review pass in which every delegate reviews the current state of the work. A fresh review after fixes starts a new round. Number rounds from 1.
- **Score:** ask each delegate for a score out of 10 every round. Record exactly what it gave. If it gave no score, record `no score`. Never infer, estimate, or invent one.
- **Loop:**
  1. Run a round (steps 2–3).
  2. For each recommendation, decide whether to **adopt** it (you agree and it's in scope), **reject** it (you disagree, with a reason), or **defer** it (valid but out of scope, so list it as a follow-up and don't do it).
  3. Make the adopted changes yourself. Run relevant checks, and commit per the repo's VCS rules.
  4. Start a fresh round. Tell each delegate what changed since the last round and why you rejected or deferred anything it raised. It can push back if it thinks a rejection is wrong.
  5. Stop when every delegate returns `SATISFIED` with no remaining blocking issues.
- **Deadlock:** if a delegate is still not satisfied after 3 rounds because of a disagreement you've already reasoned through (not a real unfixed bug), stop and ask the user to decide. Don't loop forever.

### 5. Fight scope creep

You are responsible for keeping scope small. Delegates will suggest more each round, so hold the line:

- Every round uses the **same scope statement** from step 1. Don't widen it unless the user approves.
- Adopt only changes that fix real problems in the in-scope work: bugs, regressions, missing tests for the new behavior, or clear violations of project conventions.
- Defer refactors of untouched code, new features, speculative abstractions, stylistic preferences, and "while you're here" ideas. List them under **Follow-ups**. Don't implement them.
- A later round may only raise *new* blocking issues if they come from the fixes themselves or were genuinely missed. Treat nitpicks that keep coming back as non-blocking.
- If the scope really does need to grow, ask the user first.

### 6. Report results

For code reviews, build the results table. Include one column per round and one row per delegate:

```markdown
## Council review
Rounds: 2

| Delegate | Round 1 | Round 2 |
| --- | --- | --- |
| Claude Opus 5.5 | 7/10 | 9/10 |
| Claude Fable 5.1 | 8/10 | 10/10 |
```

Use `no score` in a cell when a delegate didn't give one, and `unavailable` when it failed to respond.

Where the table goes:

- **Open PR exists:** edit the PR description. Don't post a comment. Add the `## Council review` section **just before the model and harness line at the end** of the description (for example, `🤖 Generated with Claude Code`), and replace any existing `## Council review` section rather than adding a second one:

  ```bash
  gh pr view <n> --json body -q .body > /tmp/pr-body.md
  # edit /tmp/pr-body.md: insert/replace the section above the trailing model/harness line
  gh pr edit <n> --body-file /tmp/pr-body.md
  ```

  If the description has no model/harness line, put the section at the end.
- **Work will end in a PR you're creating:** include the section in the description when you create the PR, using the same placement rule.
- **No PR:** put the same section in your final reply only.

Always finish with a short final reply covering the outcome, the scores table, deferred follow-ups, and anything left for the user to decide.
