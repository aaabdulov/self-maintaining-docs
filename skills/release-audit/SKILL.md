---
name: release-audit
description: Reconcile a release version in the tracker against what actually merged to the release branch in GitHub, in both directions. Use before cutting a release, or when asked "what is really in 2.2.6", "did anything leak to prod untested", "check the release", or when a ticket's version field looks wrong.
---

# Release audit: the version field against merged code

Answers two questions, and they are not symmetric.

1. **Merged to the release branch, missing from the tracker version.** The dangerous
   direction. The code is on the staging environment and goes to production with the release,
   while the ticket sits in a pre-QA status, has no version, or names a different one. Nobody
   tests it and nobody knows it shipped.
2. **In the tracker version, not merged.** The version claims it ships and no code is on the
   release branch. Either the PR is still open, the fix rode along inside another ticket's PR,
   or the ticket should be pulled from the release.

## The premise

**Release scope is what merged, not what a tracker query returns.** This skill exists because a
query-based answer produced false positives often enough to be untrustworthy. Version fields are
set by hand, late, by whoever remembers. Merge records are a side effect of the work itself, so
they are the ones that are true.

## How the branch topology is assumed to work

Adapt to yours. The method survives, the branch names do not.

- Feature PRs target `stage/vX.Y.Z`. Pushing that branch deploys to the staging environment. So
  "merged to staging" is exactly "merged PRs whose base ref is `stage/vX.Y.Z`".
- For production, `stage/vX.Y.Z` merges into `main` and is tagged `vX.Y.Z`.
- The release branch is deleted after release, but closed PRs stay queryable by base ref, so
  old releases remain auditable.
- Repos that version independently are out of scope for a given release. Say so rather than
  reporting them as empty.

## Configure

```
GitHub org:        ORG
Repos:             api, web
Tracker projects:  PROJ, OPS
Tokens:            ~/.github-token, ~/.atlassian-token
```

## Run it

```bash
python scripts/audit.py --version 2.2.6
python scripts/audit.py --version 2.2.6 --json report.json
python scripts/audit.py --version 2.2.6 --repos api,web --projects PROJ,OPS
```

Pass the version **name**, not an internal id. One to four minutes for a full release, because
it fetches every PR's commits and body.

## Reading the output

Direction 1, merged but not correctly versioned:

| Verdict | Meaning | Usual action |
|---|---|---|
| `NO FIX VERSION` | merged to the release branch, tracker has no version | real gap unless the code is inert, set the version |
| `WRONG FIX VERSION` | version names an earlier or unrelated release | correct it |
| `TAGGED FOR LATER` | version names a later release, no later-branch PR found | it ships now, correct it |
| `SPLIT ACROSS RELEASES` | also merged to a later release branch | usually fine, the later release delivers the visible part |
| `NON-RELEASE VERSION` | only a non-numeric version, e.g. a workstream bucket | check whether a real ticket covers the work |
| `NOT IN TRACKER` | key in Git, no such issue | typo in a branch or commit message |

Direction 2: `PR STILL OPEN`, `MERGED ELSEWHERE`, `NO CODE TRACE`.

## Benign patterns, triage these fast

- **Old key in the title, real key in the body.** Branches get named after an ancestor ticket.
  If your PR template has a links section, that is the authoritative list. Read it before
  chasing a flag.
- **Merged but inert.** Code shipped behind configuration, a flag, or routing not yet enabled.
  Git and the tracker cannot tell this from an untested leak, so it needs a human. This is the
  single most common false alarm.
- **Back-merge PRs** drag already-released keys onto the release branch. Filter them and report
  them as skipped rather than hiding them.
- **A stray key in a commit inside another ticket's PR** gets attributed to that PR. Read the PR
  title before concluding anything.

## Method notes that matter

**Collect keys from everywhere, not just the title.** PR title, head branch name, PR body as
bare keys **and** as keys inside link URLs, and every commit message in the PR.

**Never use code search to decide whether a ticket has code.** A search for a key does not match
a key that appears only inside a markdown link URL in a PR body, which is exactly how bundled
tickets get referenced. Observed: two tickets with zero search hits, both shipped in the same
PR. Use search only as a secondary trace for keys with no PR on the release branch, where a miss
means "no trace found" and never "no code".

**Do not diff tag ranges to attribute tickets to releases.** `compare/vA...vB` produces false
positives for tickets split across release branches and for stray key mentions. The PR base ref
is the reliable signal.

## After the audit

Report the rows and their verdicts and let the user triage. Do not edit version fields or
statuses without explicit approval.

**Statuses go stale within minutes.** A run takes minutes, and on release day the team
transitions tickets throughout. Any status in the output is already history by the time it is
read. Stamp the output with the read time, re-read specific keys before quoting a status, and
never quote a status from an earlier run in the same conversation. Version assignments are far
more stable than statuses, but the same rule applies while a version is being assembled.
