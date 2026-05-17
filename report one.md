# Bug Report: Hardcoded Repository in Duplicate Comment Script

## Summary

`/home/runner/work/claude-code/claude-code/scripts/comment-on-duplicates.sh` uses a hardcoded repository value:

```bash
REPO="anthropics/claude-code"
```

This causes duplicate-check commenting logic to target the wrong repository when run in `onlybugs05/claude-code`.

## Location

- File: `/home/runner/work/claude-code/claude-code/scripts/comment-on-duplicates.sh`
- Line: 11

## Impact

- The dedupe workflow can fail to find issues in the current repository.
- Comments may be attempted against issues in `anthropics/claude-code` instead of `onlybugs05/claude-code`.
- Automation behavior becomes incorrect for forks or renamed repositories.

## Reproduction Steps

1. Open an issue in `onlybugs05/claude-code`.
2. Trigger the dedupe workflow (`.github/workflows/claude-dedupe-issues.yml`).
3. Observe that script commands use `--repo anthropics/claude-code`.
4. Validation/commenting fails or targets the wrong repository.

## Expected Behavior

The script should use the active repository context (for example, `GITHUB_REPOSITORY`) so it always comments on issues in the repository where the workflow is running.
