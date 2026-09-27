# Paperclip + OpenCode Local-Agent Runbook

## Purpose

This runbook explains how to operate and troubleshoot a Paperclip agent that uses OpenCode with a local Olares-hosted model. It is sanitized: it excludes hostnames, container names, internal URLs, endpoint paths, API keys, tokens, passwords, and other access details.

Target workflow:

```text
Paperclip task -> CEO agent -> OpenCode local adapter -> local Qwen model -> Paperclip action
```

A successful task is more than a text response. It creates concrete Paperclip evidence: a comment, status update, subtask, artifact, or another recorded action.

## Verified Baseline

- Agent: CEO
- Adapter: `opencode_local`
- Verified OpenCode version: `1.18.32`
- Default model:

  ```text
  olares-qwen38/unsloth/Qwen3.8-27B-GGUF:UD-Q4_K_XL
  ```

- During compatibility testing, both the primary and cheap-model profiles used the same local Qwen model.
- The end-to-end test succeeded: the agent posted a completion comment and changed a Paperclip task from `in_progress` to `done`.

## Configuration File

OpenCode configuration location:

```text
~/.config/opencode/opencode.jsonc
```

Keep exactly one `model` property and exactly one `small_model` property:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",

  "model": "olares-qwen38/unsloth/Qwen3.8-27B-GGUF:UD-Q4_K_XL",
  "small_model": "olares-qwen38/unsloth/Qwen3.8-27B-GGUF:UD-Q4_K_XL",

  "provider": {
    // Existing provider configuration
  }
}
```

Duplicate JSON keys are a configuration hazard. Some parsers use the final duplicate, others may behave differently; retain one copy only.

## Safe Change Procedure

### Back up first

```sh
cp ~/.config/opencode/opencode.jsonc \
  ~/.config/opencode/opencode.jsonc.before-change-$(date +%Y%m%d-%H%M%S)
```

### Inspect after editing

```sh
head -8 ~/.config/opencode/opencode.jsonc
```

### Validate OpenCode directly

Run a no-override test so the configuration, not a command-line model flag, is being tested:

```sh
opencode run \
  --format json \
  "Respond with exactly: default Qwen works"
```

Success signals:

- A JSON event with `"type":"text"`.
- The expected response text.
- A final `"type":"step_finish"` event.
- `"reason":"stop"`.

Do not troubleshoot Paperclip until this direct OpenCode test succeeds.

## Paperclip End-to-End Test

Create a fresh, low-risk task assigned to CEO:

```text
Confirm that the local OpenCode model is available. Then mark this task done and leave a brief completion comment stating that the local model test passed.
```

A successful test has all three outcomes:

1. The run is `Succeeded` or `Completed`.
2. CEO leaves a completion comment.
3. The task moves from `in_progress` to `done`.

This validates the entire path: model response, OpenCode adapter, tool use, Paperclip comment, and task state update.

## How to Write Reliable Tasks

Text-only prompts verify generation but may leave no workflow evidence. Avoid prompts such as:

```text
Respond with exactly: Paperclip local model works
```

Prefer tasks that specify scope, deliverable, concrete Paperclip action, and end state:

```text
[Bounded objective].

Use only [allowed sources]. Do not inspect [out-of-scope sources].

Add one comment containing:
1. [Required finding]
2. [Required recommendation]
3. [Required next action]

Then mark this task done.
```

## Control Runtime and Churn

Open-ended research can exceed Paperclip's runtime limit. Automatic retries may then multiply activity without producing a conclusion or comment.

Use bounded prompts:

```text
Review the issue title, description, and five most recent run summaries only.
Do not inspect full logs or use extra skills.
Add a short comment with the likely cause, one recommended fix, and one next action.
Then mark the task done.
```

Break larger work into smaller subtasks. Limit source material, tool calls, and expected output for every task that must finish quickly.

## Common Outcomes

### Useful output but no concrete action

**Meaning:** The model answered, but CEO did not create a Paperclip comment, state update, artifact, or other action.

**Response:** Rewrite the task to require a specific Paperclip action and a final status change.

### `timed_out`

**Meaning:** The run exceeded the Paperclip runtime limit.

**Response:** Stop retry churn, resolve or cancel the stranded run intentionally, then create a narrower successor task.

### `adapter_failed`

**Meaning:** Paperclip could not invoke or communicate successfully with the OpenCode adapter.

**Response:** Confirm the direct OpenCode test; verify CEO's adapter and model selections; inspect the earliest error in the failed run transcript; redact sensitive details before sharing diagnostics.

### Recovery needed or stranded task

**Meaning:** A retry was unable to restore a live execution path.

**Response:** Record an explicit decision—cancel, block, defer, or replace—and prevent unlimited retries.

## Model Selection

Using the same model for primary and cheap work is acceptable while proving compatibility. After stability is established:

- Retain the stronger model for CEO's main reasoning and tool use.
- Consider a smaller, faster model for summaries or lightweight jobs.
- Change one setting at a time.
- Repeat the direct OpenCode test and the Paperclip end-to-end test after every meaningful configuration change.

## Security and Documentation Rules

Do not put the following in Paperclip tasks, comments, screenshots, chat transcripts, or shared runbooks:

- Internal hostnames, IP addresses, container names, or service identifiers.
- Model-service URLs, endpoint paths, or network topology details.
- API keys, tokens, passwords, cookies, or provider credentials.
- Unredacted configuration files or environment-variable output.
- Private links unless their sharing is intentionally approved.

Use neutral placeholders:

```text
LOCAL_MODEL_PROVIDER=<configured locally>
LOCAL_MODEL_NAME=<configured model>
PAPERCLIP_ADAPTER=opencode_local
```

## Routine Operations Checklist

- [ ] Direct OpenCode test passes without a `--model` override.
- [ ] CEO is assigned to the intended task.
- [ ] Task has a bounded scope.
- [ ] Task requires a comment, status update, artifact, or other action.
- [ ] Task has a clear terminal state: done, blocked, cancelled, or review.
- [ ] No secrets or local infrastructure details are included.
- [ ] The run ledger is checked for retries or duplicate work.

## Incident-Response Checklist

Use this one-page checklist for timeouts, retries, adapter failures, and stranded tasks.

### 1. Stabilize

- [ ] Do not create another duplicate task while one is running or retrying.
- [ ] Note the task ID, run ID, current status, elapsed time, and agent.
- [ ] Capture the first visible error only; redact infrastructure details and secrets.
- [ ] Check whether a live run is still making progress before cancelling it.

### 2. Classify

- [ ] **Text succeeded, no action:** Useful output but no Paperclip comment/status/artifact.
- [ ] **Timed out:** Run exceeded the runtime limit.
- [ ] **Adapter failed:** Paperclip could not complete the OpenCode adapter invocation.
- [ ] **Stranded/recovery needed:** Retry cannot establish a live execution path.
- [ ] **High churn:** Many runs occur without meaningful comments or state progress.

### 3. Contain retries

- [ ] Stop or resolve automatic recovery if it is repeatedly retrying.
- [ ] Use the task recovery controls to record an explicit decision.
- [ ] Select one status: `done`, `blocked`, `cancelled`, or `deferred`.
- [ ] Leave a concise human comment stating why the run was stopped.

Suggested comment:

```text
Paused after repeated [timeout/adapter failure/retry] behavior. No further automatic retries should run until the cause is confirmed. Next step: create a bounded follow-up task after reviewing the sanitized run error.
```

### 4. Verify the execution path

- [ ] Run the direct no-override OpenCode test.
- [ ] Confirm the intended primary model is selected for CEO.
- [ ] Confirm the `opencode_local` adapter remains selected.
- [ ] Verify the local model is healthy using only approved, non-sensitive checks.
- [ ] Inspect the first failed run transcript for the earliest actionable error.

### 5. Recover safely

- [ ] Replace an open-ended task with a narrow, time-bounded task.
- [ ] Limit inputs: issue text plus a small number of recent summaries.
- [ ] Prohibit expensive or broad log exploration unless explicitly required.
- [ ] Require a comment and final status update.
- [ ] Run only one recovery task before allowing further retries.

Recovery-task template:

```text
Review only [limited sources]. Do not inspect full logs or use optional skills.
Provide one short comment with the cause, one fix, and one next action.
Then set the task status to [done or blocked].
```

### 6. Close the incident

- [ ] Confirm the replacement task produced a concrete Paperclip action.
- [ ] Confirm the original runaway task is no longer retrying.
- [ ] Add a short post-incident note describing the cause and prevention step.
- [ ] Update this runbook if a new failure pattern or fix was learned.

## Change Log Template

```md
## YYYY-MM-DD — Change title

- Changed:
- Reason:
- Validation command:
- Result:
- Rollback location:
- Follow-up:
```
