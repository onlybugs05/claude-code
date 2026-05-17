# Security Vulnerability Report

## Scope
Repository: `onlybugs05/claude-code`  
Assessment focus: GitHub Actions workflows and automation scripts that process untrusted issue/comment input.

---

## Finding 1: Untrusted Users Can Trigger Costly Claude Workflows (Abuse/DoS Risk)

### Severity
**Medium** (abuse of paid automation and CI capacity; may escalate to High depending on billing/limits)

### Affected Files
- `/home/runner/work/claude-code/claude-code/.github/workflows/claude-issue-triage.yml`
- `/home/runner/work/claude-code/claude-code/.github/workflows/claude-dedupe-issues.yml`

### Evidence
- `allowed_non_write_users: "*"` in:
  - `claude-issue-triage.yml` (line 35)
  - `claude-dedupe-issues.yml` (line 32)
- `claude-issue-triage.yml` triggers on:
  - `issues: opened`
  - `issue_comment: created`
- The triage job condition allows non-bot issue comments, enabling repeated triggering by any GitHub account.

### Description
The workflows allow users without write access to invoke Claude-based automation. Because issue comments are untrusted and can be posted repeatedly, this creates a resource abuse path where an attacker can generate many runs and consume paid API/workflow resources.

### Reproduction Steps
1. Use a GitHub account without write access to the repository.
2. Open an issue in the target repository.
3. Post many comments on the issue (normal text, no special permissions required).
4. Observe that each comment triggers the `Claude Issue Triage` workflow.
5. Verify that workflow run count and Claude action usage increase with each comment.

### Expected vs Actual
- **Expected:** Only trusted users or tightly constrained events should trigger expensive Claude actions.
- **Actual:** Any non-write user can repeatedly trigger the automation.

### Security Impact
- Workflow/API cost amplification.
- CI queue saturation and degraded automation reliability.
- Increased noise/spam from automated triage actions.

### Recommended Fixes
1. Replace wildcard trust with explicit allowlists:
   - Avoid `allowed_non_write_users: "*"` unless strictly necessary.
2. Add abuse controls:
   - Per-user cooldown/rate limiting logic.
   - Label or maintainer-gated execution.
3. Tighten trigger conditions:
   - Limit comment-triggered runs to trusted actors or explicit slash-command patterns.
4. Add monitoring/alerting:
   - Alert on unusual workflow run spikes.

---

## Finding 2: Unsafe JSON Construction with Untrusted Issue Title in Statsig Logger

### Severity
**Low to Medium** (data integrity and telemetry reliability impact; may become higher if downstream parsing trust assumptions exist)

### Affected File
- `/home/runner/work/claude-code/claude-code/.github/workflows/log-issue-events.yml`

### Evidence
- Untrusted issue title sourced from event:
  - `ISSUE_TITLE: ${{ github.event.issue.title }}` (line 18)
- JSON payload is manually constructed in shell using string interpolation and partial escaping:
  - `sed "s/\"/\\\\\"/g"` (line 34)
  - JSON body built inline in `curl -d '...'` block.

### Description
The workflow uses manual shell string interpolation to assemble JSON, escaping only double quotes in the issue title. This is not full JSON-safe encoding and can fail or produce malformed payloads with crafted characters (e.g., backslashes/control characters/newlines), affecting integrity of logged events.

### Reproduction Steps
1. Create an issue with a crafted title containing escape-sensitive characters (e.g., backslashes, mixed quotes, newline/control-like input).
2. Trigger the `Log Issue Events to Statsig` workflow via issue creation.
3. Observe request behavior:
   - malformed JSON payload or altered metadata values,
   - non-200 response from endpoint,
   - telemetry inconsistencies.

### Expected vs Actual
- **Expected:** Event metadata should be JSON-encoded safely for all user-controlled inputs.
- **Actual:** Manual interpolation with partial escaping risks malformed payloads and integrity issues.

### Security Impact
- Telemetry poisoning or loss.
- Inaccurate analytics/alerts based on corrupted metadata.
- Potential downstream parsing failures in consuming systems.

### Recommended Fixes
1. Build JSON with `jq -n --arg ...` for all untrusted fields.
2. Avoid manual `sed`-based escaping for JSON construction.
3. Add error handling/validation for payload generation before sending.
4. Add tests or dry-run validation for edge-case titles.

---

## Notes for Triage
- No direct remote code execution path was identified in reviewed scripts.
- Main risk centers are workflow abuse controls and safe handling of untrusted text in telemetry payload generation.

## Suggested Priority
1. **First:** Fix Finding 1 (abuse/cost amplification risk).
2. **Second:** Fix Finding 2 (telemetry integrity and robustness).
