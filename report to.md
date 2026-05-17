# Bug Report 2: Hardcoded “New Issue” URL points to wrong repository

## Summary

`/home/runner/work/claude-code/claude-code/scripts/sweep.ts` hardcodes:

```ts
const NEW_ISSUE = "https://github.com/anthropics/claude-code/issues/new/choose";
```

This sends users to the upstream repo instead of the current repository when running in forks (like `onlybugs05/claude-code`).

## Location

- File: `/home/runner/work/claude-code/claude-code/scripts/sweep.ts`
- Line: 7

## Impact

- Sweep output can direct users to file issues in the wrong repository.
- Fork maintainers lose issue intake in their own repo.
- Creates confusion and misrouted bug reports.

## Reproduction Steps

1. Run sweep logic in `onlybugs05/claude-code`.
2. Trigger path/message that includes `NEW_ISSUE`.
3. Observe generated link points to `anthropics/claude-code`.

## Expected Behavior

The issue creation URL should be built dynamically from repository context (current owner/repo), not hardcoded to `anthropics/claude-code`.
