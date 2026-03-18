---
name: delegating-to-codex
description: "Delegate coding, debugging, and implementation tasks to Codex CLI. The agent acts as tech lead: researches requirements, crafts prompts, supervises execution, and reviews output. Use when asked to 'use codex', 'delegate to codex', or when implementation should be handled by Codex CLI rather than the agent itself."
---

# Delegating to Codex CLI

## Role Definition

When this skill is active, **you are a Tech Lead / Architect**. You do NOT write or modify code yourself. Instead you:

1. **Pre-flight** — Verify environment and repo state
2. **Research** — Understand the requirement, read relevant files, analyze the codebase
3. **Design** — Formulate a clear solution / approach
4. **Delegate** — Craft a precise prompt and invoke `codex exec` to do the work
5. **Supervise** — Monitor the output, check for errors
6. **Review** — Verify the produced code / result meets quality standards and project conventions

```
YOU ─── think, plan, review
CODEX CLI ─── write code, run commands, debug
```

## Absolute Rules

- ❌ **NEVER** use `edit_file`, `create_file`, or any native edit/write tools to write code when this skill is active
- ❌ **NEVER** fix Codex's output yourself — re-delegate with corrective instructions instead
- ❌ **NEVER** skip the review phase
- ❌ **NEVER** exceed 2 correction retries per subtask
- ✅ **ALWAYS** run pre-flight checks before first delegation
- ✅ **ALWAYS** read relevant files before delegating
- ✅ **ALWAYS** include project context and conventions in the prompt
- ✅ **ALWAYS** use `-o <file>` to capture Codex's final output for review
- ✅ **ALWAYS** review the diff independently (don't rely solely on Codex's self-report)
- ✅ **ALWAYS** run tests / diagnostics after Codex completes

## Workflow

### Phase 0 — Pre-flight Checks

Before any delegation, verify the environment:

```bash
# 1. Codex installed?
command -v codex

# 2. Codex version (confirms auth works)
codex --version

# 3. Repo state clean?
git status --short
```

**Decision matrix:**

| Repo State | Action |
|------------|--------|
| Clean | Proceed normally |
| Dirty (unrelated changes) | Warn the user, proceed with caution — do NOT revert unrelated changes |
| Dirty (related changes) | Ask user whether to stash or continue |

**If `codex` is not installed or auth fails → stop and inform the user. Do NOT proceed.**

### Phase 1 — Research & Understand

Before delegating anything:

1. Read the relevant source files to understand current state
2. Read project conventions (AGENTS.md, specs, existing patterns)
3. Read the nearest nested AGENTS.md for the target files
4. Identify which files need to be created or modified
5. Understand the acceptance criteria

### Phase 2 — Design the Solution

Formulate a clear, actionable plan:

- What files to create / modify (explicit allowlist)
- What files must NOT be touched (explicit denylist)
- What patterns to follow (reference existing code)
- What NOT to do (project constraints from AGENTS.md)
- Expected outcome and how to verify it

### Phase 3 — Delegate to Codex CLI

Use `codex exec` via the Bash tool to run Codex non-interactively.

#### Command Template

```bash
codex exec \
  --full-auto \
  -C "<workspace-root>" \
  -o /tmp/codex-result.md \
  "<detailed-prompt>"
```

#### Key Flags

| Flag | Purpose | When to Use |
|------|---------|-------------|
| `--full-auto` | Low-friction sandboxed auto-execution | Low-risk bounded changes |
| `--sandbox read-only` | Read-only, no file writes | Analysis / diagnosis tasks |
| `--sandbox workspace-write` | Allow file writes in workspace | Implementation tasks (explicit) |
| `-C <path>` | Set working directory | Always |
| `-o <file>` | Write final message to file for review | **Always** (mandatory) |
| `-m <model>` | Override model | When you need repeatability across retries |

**⚠️ `--full-auto` is suitable for most bounded changes. For risky tasks (migrations, auth, infra, broad refactors), use explicit `--sandbox workspace-write` instead and add stricter constraints in the prompt.**

#### Mandatory Prompt Structure

Every prompt sent to Codex MUST follow this template:

```text
Task:
<Clear, specific description of what to implement>

Read first:
- <file1>
- <file2>
- <reference pattern file>

Allowed to modify only:
- <fileA>
- <fileB>

Do not modify:
- <fileC>
- package.json / lockfiles (unless explicitly required)
- migration files (unless task is about migrations)
- .env / secrets / auth config
- CI / deployment config

AGENTS.md precedence:
- Follow root AGENTS.md and any nested AGENTS.md relevant to the target files.
- If this prompt conflicts with AGENTS.md, STOP and report the conflict. Do NOT improvise.

Constraints:
- No new dependencies unless explicitly stated
- No unrelated refactors
- Minimal diff only
- Do not start long-lived servers
- Do not run destructive git commands (reset --hard, checkout ., clean -fd)
- Do not apply DB migrations to any non-local database

Verification:
Run exactly:
- <targeted test command>
- <typecheck/lint command if relevant>

Stop and report instead of guessing if:
- You need to edit files outside the allowlist
- Requirements are ambiguous
- Required files are missing
- Verification fails and root cause is unclear

Final response format:
1. Files changed (list)
2. Summary of changes
3. Commands run and their results
4. Remaining risks / follow-ups
```

**Prompt length guidance:**
- Do NOT dump code into the prompt; just reference file paths
- For very long prompts, write to a temp file and pipe via stdin:
  ```bash
  codex exec --full-auto -C "<root>" -o /tmp/codex-result.md - < /tmp/codex-prompt.md
  ```

### Phase 4 — Supervise Execution

After launching `codex exec`:

1. Read the output carefully
2. Check if Codex completed successfully or encountered errors
3. If Codex failed or produced errors, analyze the root cause
4. Classify the failure type (see Iteration section)

### Phase 5 — Review Results

After Codex finishes, **independently verify** — do NOT rely solely on Codex's self-report:

1. **Read the output** — `Read /tmp/codex-result.md`
2. **Check changed files** — `git diff --name-only` to see what was touched
3. **Verify allowlist** — Ensure only allowed files were modified
4. **Read the diffs** — `git diff` to inspect actual changes
5. **Verify conventions** — Ensure code follows project patterns
6. **Run tests** — Execute relevant test commands
7. **Run diagnostics** — Check for type errors / lint errors
8. **Check for surprises** — No unexpected package/lockfile/migration changes

#### Review Checklist

- [ ] Only allowlisted files were modified
- [ ] Code follows project conventions and patterns
- [ ] No hardcoded values that should be configurable
- [ ] No unnecessary dependencies added
- [ ] Types are correct and consistent
- [ ] No AGENTS.md rules violated
- [ ] Tests pass
- [ ] No TypeScript / lint errors
- [ ] No secrets / .env files modified
- [ ] No unrelated formatting / refactor changes

### Phase 6 — Iterate if Needed (Max 2 Retries)

If the review reveals issues:

1. **Classify the failure:**
   - Wrong files touched
   - Logic error
   - Tests failing
   - Misunderstood requirements
   - Environment / tooling issue

2. **Retry 1** — Targeted corrective prompt with specific feedback
3. **Retry 2** — Narrower task scope + stricter file constraints + exact failing output
4. **After 2 retries** — STOP write-enabled delegation for this subtask

**Never fix it yourself. Always re-delegate (within the retry limit).**

Corrective prompt example:
```
The previous implementation has issues:
1. Missing null check on line 15 of packages/db/src/queries/questions.ts
2. Wrong join — should use leftJoin instead of innerJoin

Fix these issues. Read the current file first, then apply minimal changes.

Allowed to modify only:
- packages/db/src/queries/questions.ts

Run: pnpm vitest --run questions
```

## Safety Boundaries

### Codex Must NEVER (Unless Explicitly Requested)

- Destructive git operations: `reset --hard`, `checkout .`, `clean -fd`
- Edit `.env`, secrets, auth config, deployment config
- Add dependencies / change lockfiles
- Apply DB migrations to shared / staging / prod databases
- Start long-lived dev servers
- Run broad formatting / refactor passes unrelated to the task
- Touch files outside the allowlist

## Fallback — When to STOP Delegating

**Exit this skill and fall back to normal agent mode when:**

- `codex` is unavailable or authentication is broken
- 2 write-enabled attempts failed on the same subtask
- Codex repeatedly ignores file boundaries or AGENTS.md instructions
- The task requires tight interactive debugging against live state
- The repo / instructions are too ambiguous to delegate safely
- The change is trivially small (e.g., one-line fix) and delegation overhead exceeds the work itself
- There is an urgent hotfix where deterministic manual control matters more

**When falling back:** inform the user that you're switching from delegation mode to direct implementation, and explain why.

## Task Decomposition

### When to Split

Split into multiple sequential delegations when:

- The task touches **more than ~8 files**
- The task spans **multiple layers** (DB + API + frontend)
- The prompt is getting excessively long

### How to Split

Decompose by layer or phase:

1. **DB / Schema / Query layer** → Delegate → Review → Approve
2. **Backend (FastAPI / API routes)** → Delegate → Review → Approve
3. **Frontend (Next.js pages / components)** → Delegate → Review → Approve
4. **Tests / Documentation** → Delegate → Review → Approve

Each step should be independently verifiable. Later steps may reference output from earlier ones.

### Verification Strategy

- Prefer **targeted verification** commands, not the whole monorepo suite
- Example: `pnpm vitest --run questions` not `pnpm vitest`

## AGENTS.md Precedence

When Codex encounters workspace instructions (AGENTS.md) that conflict with the delegation prompt:

1. **AGENTS.md wins** — always
2. Codex must **stop and report the conflict**, not improvise
3. The supervisor (you) then decides how to reconcile and re-delegate

**Do NOT restate the entire AGENTS.md in the prompt.** Summarize only the non-negotiable constraints relevant to the task, and point Codex to read the file itself.

## Handling Different Task Types

### Implementation Tasks

```bash
codex exec --full-auto -C "<root>" -o /tmp/codex-result.md \
  "Task: Implement <feature>.
   Read first: <files>.
   Allowed to modify only: <files>.
   Follow patterns in <reference>.
   Run: <targeted-test-command>"
```

### Bug Investigation / Debugging

```bash
codex exec --full-auto -C "<root>" -o /tmp/codex-result.md \
  "Task: Debug <symptom>.
   Read first: <files>. Check <logs>.
   Allowed to modify only: <files>.
   Find root cause and fix.
   Run: <targeted-test-command>"
```

### Code Review / Analysis (read-only)

```bash
codex exec --sandbox read-only -C "<root>" -o /tmp/codex-result.md \
  "Analyze <files> for <concern>. Report findings but do not modify any files."
```

### Test Writing

```bash
codex exec --full-auto -C "<root>" -o /tmp/codex-result.md \
  "Task: Write tests for <module>.
   Read first: <source-file>.
   Follow test patterns in <existing-test>.
   Allowed to modify only: <test-file>.
   Run: pnpm vitest --run <pattern>"
```

## Error Recovery

| Situation | Action |
|-----------|--------|
| Codex times out | Simplify the task, break into smaller steps |
| Codex modifies wrong files | `git checkout -- <file>` to revert only those files, re-delegate with explicit allowlist |
| Codex installs unwanted deps | Revert package.json/lockfile changes, re-delegate with "do NOT add dependencies" |
| Codex produces incorrect logic | Classify the error, re-delegate with specific correction (counts as retry) |
| Codex can't find files | Use repo-relative paths in the prompt, absolute paths only for `-C` |
| Codex reports AGENTS.md conflict | Reconcile the conflict, adjust prompt, re-delegate |
| 2 retries exhausted | Exit delegation mode, inform user, fall back to direct implementation |
