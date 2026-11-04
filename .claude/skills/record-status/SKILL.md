---
name: record-status
description: Record a TEETHi plan item as built, fixed or merged in the root docs repo. Ticks the item in docs/plans, adds the Verified note, and commits with the sub-repo commit hashes in the message.
argument-hint: <what> as <built|fixed|merged|audited> [repos...]
disable-model-invocation: true
---

# Record status in the root docs repo

Record "$ARGUMENTS" in the shared docs. The root repo (`D:\Projects\TEETHi Project`) tracks only `docs/`, `CLAUDE.md`, `.mcp.json` and `.claude/`. The four project folders are separate repos. Never commit in them from this skill.

## 1. Collect the facts from each involved repo

Work out which repos the change touched from the arguments and the conversation. If that's unclear, check each repo's recent commits. The repos are `TEETHi-Backend`, `TEETHi-Frontend`, `TEETHi-Patient-App` and `TEETHI-Dentist-App` (note the capital `I` in the Dentist folder name).

For each one:

```bash
git -C <repo> branch --show-current
git -C <repo> log --oneline -5
git -C <repo> status --short
```

- Use the short hash (7 chars) of the commit or commits that did the work. When recording a merge, use the hash that is now on `master`, the default branch in all four project repos. Only the root docs repo uses `main`.
- If the work is still on a feature branch, note `on <branch>`.
- If a repo has uncommitted changes that belong to this work, stop and tell the user. Don't record work that isn't committed.

Don't invent test counts or verification results. Use only numbers this session actually saw, such as test output or analyzer counts. If something wasn't checked (browser, device), say so in the note, the way earlier notes do.

## 2. Update the docs

- **Plans** (`docs/plans/*.md`): find the item. Change `- [ ]` to `- [x]` only once the work is built and verified. Add or extend the `**Verified (YYYY-MM-DD):**` paragraph under the phase. Match the existing notes: repo name, hash, branch or merge, test files and counts, what was and wasn't checked live, and anything still open. When recording a merge, update the existing Verified note (e.g. "merged to `master` 2026-10-30") rather than adding a second one.
- **Issue IDs** (e.g. `R8-08`, `H5`, `M9`): update their entries in `docs/CONSISTENCY_REPORT.md` or the latest `docs/E2E_REPORT_*.md` too.
- **Contract changes:** if a backend change alters an endpoint, update every client plan that documents it in the same commit (docs rule 3). Then suggest running `/sync-api-contract`.
- Use today's date (absolute, never "today").

## 3. Commit in the root repo only

Stage only the docs files you changed. `.claude/launch.json` often has unrelated local edits, so leave it alone.

Message format, following the root history:

```
Record <what> as <state> (<Repo> <hash>[ on <branch>], <Repo> <hash>, ...)
```

Examples:
- `Record R8-08 payments polish as fixed (TEETHi-Backend 2afd4b6, TEETHi-Patient-App 65d3e1b on fix/r8-08-payments)`
- `Record account deletion as merged (TEETHi-Backend 91a92ce, TEETHi-Patient-App 37028d4, TEETHI-Dentist-App c183d57)`

End the message with the attribution trailer the session requires. Commit straight to `main`, the way the rest of this repo's history does. There is no remote, so don't push.

Finish by stating the root commit hash and the docs sections you changed.
