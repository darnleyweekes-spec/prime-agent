---
name: skillopt-sleep
description: Optimize ALPHA and Prime Agent skills from past Prime Agent sessions using the darnleyweekes-spec/SkillOpt fork. Use when the user asks ALPHA or a specialist agent to learn from prior work, improve a SKILL.md, run a validated sleep/offline optimization cycle, inspect staged SkillOpt changes, or adopt an approved skill improvement.
compatibility: Requires git, Python 3.10+, and Prime Agent. Uses the SkillOpt source checkout at SKILLOPT_SLEEP_REPO or ~/.prime/agent/vendor/SkillOpt.
---

# SkillOpt-Sleep for ALPHA

Use the user's `darnleyweekes-spec/SkillOpt` fork as an offline optimization layer for ALPHA and Prime Agent skills. SkillOpt may harvest prior Prime Agent sessions, mine recurring tasks, replay them, propose bounded edits, validate those edits on held-out tasks, and stage accepted proposals. It does not replace ALPHA's routing, safety rules, human approval boundaries, or the underlying model.

## Safety boundary

ALPHA mode is **stage-first and approval-gated**:

- Never pass `--auto-adopt`.
- Keep SkillOpt's validation gate enabled.
- Set `evolve_memory` to `false`; do not let SkillOpt rewrite `CLAUDE.md`, `AGENTS.md`, ALPHA architecture rules, provider settings, credentials, or system prompts.
- Optimize one target `SKILL.md` at a time.
- Prefer a project skill under `.prime/agent/skills/<name>/SKILL.md` or `.agents/skills/<name>/SKILL.md`.
- Do not adopt directly into the package-shipped `packages/coding-agent/skills/` tree or an installed package. If a built-in skill needs tuning, create a project-level override with `skill-creator`, then optimize that override.
- Treat every generated edit as a proposal. Show validation evidence and the exact staged diff before asking for approval to adopt.
- Consequential actions remain human-gated: external messages, spending, credentials, security testing, production changes, destructive actions, and irreversible operations.

If the requested ALPHA specialist does not yet have its own project `SKILL.md`, use `skill-creator` first to create a bounded specialist skill, then target that file with SkillOpt.

## Prime Agent compatibility

Prime Agent is built on the Pi agent stack and stores default session history under `~/.prime/agent/sessions`. SkillOpt's Pi transcript source can read that history when pointed at `~/.prime`:

```bash
--source pi --pi-home "$HOME/.prime"
```

SkillOpt's Pi backend can call Prime Agent because Prime Agent supports the noninteractive flags the backend requires. Point it at the current Prime Agent executable:

```bash
--backend pi --pi-path "$(command -v prime-agent)"
```

If `PRIME_AGENT_CODING_AGENT_DIR` or the session directory is customized, do not guess the transcript location. Resolve the active Prime Agent session directory before harvesting and only use the Pi source when its expected `<pi-home>/agent/sessions` layout matches. Otherwise export reviewed tasks and use `--tasks-file` instead of harvesting the wrong directory.

## Source setup

Use the user's fork, not an unrelated checkout. For an explicit install or enable request, install it once under Prime Agent's data directory unless `SKILLOPT_SLEEP_REPO` already points to a reviewed checkout:

```bash
REPO="${SKILLOPT_SLEEP_REPO:-$HOME/.prime/agent/vendor/SkillOpt}"
if [ ! -d "$REPO/.git" ]; then
  mkdir -p "$(dirname "$REPO")"
  git clone --depth 1 https://github.com/darnleyweekes-spec/SkillOpt.git "$REPO"
fi
export SKILLOPT_SLEEP_REPO="$REPO"
python3 --version
bash "$REPO/plugins/run-sleep.sh" status --project "$(pwd)"
```

Do not silently pull or replace an existing checkout. Updating the fork is a separate, explicit action so a previously reviewed integration does not change underneath ALPHA.

## ALPHA configuration

Before a real optimization run, make sure `~/.skillopt-sleep/config.json` keeps the ALPHA boundary. Preserve unrelated existing keys when editing this file.

Required ALPHA settings:

```json
{
  "evolve_memory": false,
  "evolve_skill": true,
  "gate_mode": "on",
  "gate_no_regression": true,
  "transcript_source": "pi",
  "pi_home": "~/.prime",
  "pi_path": "prime-agent",
  "preferences": "Preserve ALPHA controller authority, specialist scope, human approval gates, factual claims, source grounding, concise business communication, and existing safety/reliability rules. Do not broaden permissions or automate send, spend, credential, security-testing, production, destructive, or irreversible actions."
}
```

A project may use a stricter `preferences` value, but never weaken these boundaries while operating as ALPHA.

## Workflow

Use this sequence unless the user explicitly requests only one read-only action.

### 1. Resolve the target

Identify exactly one project-level specialist skill:

```bash
TARGET_SKILL=".prime/agent/skills/<specialist>/SKILL.md"
test -f "$TARGET_SKILL"
```

Do not infer a nonexistent target. If it is missing, create the specialist skill first with `skill-creator`.

### 2. Inspect status

```bash
REPO="${SKILLOPT_SLEEP_REPO:-$HOME/.prime/agent/vendor/SkillOpt}"
bash "$REPO/plugins/run-sleep.sh" status --project "$(pwd)"
```

### 3. Harvest read-only history

For the default Prime Agent layout:

```bash
bash "$REPO/plugins/run-sleep.sh" harvest --project "$(pwd)" \
  --source pi --pi-home "$HOME/.prime" \
  --target-skill-path "$TARGET_SKILL"
```

When transcripts may contain sensitive client, credential, security, or production context, export tasks first, review/redact them, mark the task file reviewed, and use that file for the real backend. Pattern redaction is defense in depth, not a guarantee that transcript-derived prompts are secret-free.

### 4. Smoke test with zero provider spend

Always run the mock dry-run first for a new target or after changing the integration:

```bash
bash "$REPO/plugins/run-sleep.sh" dry-run --project "$(pwd)" \
  --source pi --pi-home "$HOME/.prime" \
  --target-skill-path "$TARGET_SKILL" \
  --backend mock --json
```

A plumbing smoke test does not prove the skill improved; it only verifies the cycle can load the source and target safely.

### 5. Run validated optimization

Only after the mock path works and the transcript/task boundary is acceptable:

```bash
PRIME_AGENT_BIN="$(command -v prime-agent)"
bash "$REPO/plugins/run-sleep.sh" run --project "$(pwd)" \
  --source pi --pi-home "$HOME/.prime" \
  --target-skill-path "$TARGET_SKILL" \
  --backend pi --pi-path "$PRIME_AGENT_BIN" \
  --max-sessions 5 --max-tasks 3 --edit-budget 4 --progress
```

Do not add `--auto-adopt`.

### 6. Review staged evidence

A successful `run` stages a proposal; it does not make the proposal authoritative. Read the generated report and staged skill. Report at least:

- target skill path;
- sessions and tasks used;
- baseline and candidate held-out scores;
- gate result;
- exact proposed add/delete/replace edits;
- any regressions or missing evidence;
- staging directory.

Reject or rerun when there is no held-out improvement, a regression, broadened permissions, weakened safety language, unsupported claims, or cross-specialist scope creep.

### 7. Adopt only after approval

After the user reviews the staged evidence and explicitly approves that proposal:

```bash
bash "$REPO/plugins/run-sleep.sh" adopt --project "$(pwd)"
```

Use SkillOpt's adoption path so backups and staged-state checks are preserved. Do not hand-copy staged text over the live skill as a shortcut.

## Multi-agent ALPHA policy

For multiple specialists, optimize serially by skill even if evaluation work is parallelized. A specialist's learned rule must stay within that specialist's charter. Do not consolidate several agents into one shared skill merely because similar instructions appear in their histories.

ALPHA remains the controller:

1. ALPHA selects the specialist and target skill.
2. SkillOpt evaluates and stages a bounded proposal.
3. ALPHA checks scope, safety, evidence, and regressions.
4. The user approves or rejects the staged proposal.
5. SkillOpt adopts the approved proposal with backups.

This separation prevents self-improvement from becoming self-expansion of permissions.

## Scheduling

Do not schedule SkillOpt just because the skill is installed. Scheduling is a separate user decision. If the user later requests a nightly cycle, configure it to **stage only** and keep `--auto-adopt` disabled. Validate the target path, transcript source, provider data boundary, and scheduled account authentication before enabling it.

## Hard rules

- Never expose or commit raw transcripts, secrets, credentials, private client data, or provider tokens.
- Never claim a staged proposal is installed until `adopt` succeeds.
- Never claim a held-out gain guarantees broader production performance.
- Never skip the mock smoke test for a new target.
- Never use a real backend on unreviewed sensitive tasks.
- Never weaken ALPHA approval, safety, reliability, or specialist boundaries to improve a benchmark score.
