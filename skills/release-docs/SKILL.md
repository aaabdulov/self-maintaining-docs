---
name: release-docs
description: Update existing documentation to reflect what shipped in a software release. Use when the user says "we released X", "update docs for the release", "sync docs", names a release tag or version, or asks which tickets are in a release. Collects the tickets in a release, classifies which ones changed documented behaviour, maps each to the right existing page, and PROPOSES edits for approval (never publishes unattended).
---

# Release-driven documentation sync

When a release ships, find the existing pages whose content is now wrong and propose precise
edits.

This is **documentation maintenance**, not release-notes generation. Do not create a "release
notes" page unless explicitly asked. The goal is that pages describing how the product behaves
keep describing how it behaves.

## Hard rules

- **Propose first, always.** Draft the exact change per page, with reasoning and a confidence
  level. Publish only after the user approves that specific page. Semantic mapping is fuzzy and
  a confident edit to the wrong page is the failure mode that matters, because it damages a page
  that was previously correct. See `patterns/propose-never-publish.md`.
- **Use the REST APIs directly for reads.** Where an MCP server exists it is often fine, but its
  token can go stale independently of the one on disk, and a stale token surfaces as a 404 that
  reads like a wrong path. When a private read 404s, verify the token authenticates at all
  before assuming the path is wrong.
- **Delegate publishing to the documentation skill.** Reading, building markup, preserving
  embedded media, encoding safety and the post-update checklist belong in one place. Pass it a
  target page id and an approved change. Do not reimplement markup building here.

## Configure

Replace with your own. Everything else in this skill is generic.

```
GitHub org:        ORG
Repos in scope:    api (frequent), web (frequent), admin (rare), infra (rare)
Ticket projects:   PROJ, OPS
Doc spaces:        PS  product and user-facing behaviour
                   DS  technical mechanics
Tenant:            https://example.atlassian.net
Tokens:            ~/.github-token, ~/.atlassian-token
```

Infrastructure repos are mostly default-skip, but read the diff before skipping. Mail-sending
configuration, ingress-level redirects, DNS and rate-limit infrastructure are documented
mechanics that live in DS and change in the infra repo.

## Confirmed mappings (maintain this)

Check this table **before** semantic search, and update it whenever a run confirms or a
reviewer corrects a mapping. This is what makes the next run cheaper than this one. See
`patterns/mapping-memory.md`.

| Feature area | Page | Space | id | Notes |
|---|---|---|---|---|
| _example_ Sign-in, all methods, lockout | Sign in | PS | `<id>` | Lockout mechanics detailed in DS `<id>`. |
| _example_ Registration, standalone | Sign up | PS | `<id>` | Distinct from the in-flow variant. Check which flow the diff touches. |
| _example_ Mail templates and identifiers | Mail register | PS | `<id>` | Child of a children-listing index. Target the child, never the index. |

## Pipeline

### 1. Collect the tickets in the release

Read the release body, which usually lists the merged pull requests, and the compare between
the previous tag and this one. The compare catches keys that never made it into a PR title.

```bash
TOKEN=$(tr -d '\n\r' < ~/.github-token)
curl -s -H "Authorization: token $TOKEN" \
  "https://api.github.com/repos/ORG/<repo>/releases/tags/<tag>"
curl -s -H "Authorization: token $TOKEN" \
  "https://api.github.com/repos/ORG/<repo>/compare/<prevtag>...<tag>"
```

Extract ticket keys with `(?:PROJ|OPS)-\d+`, case-insensitive.

**If your projects have ever been renamed, normalise old keys to new ones.** Git history keeps
the old key in branch names, PR titles and PR bodies forever, while the tracker answers only to
the new one. Without normalisation every pre-rename ticket reads as "no trace found".

The version field on the ticket is a **weak cross-check only**. It is often missing or wrong.
What merged is the source of truth. See the `release-audit` skill for why.

### 2. Classify: documented-behaviour change, or skip

Most tickets in a release do not touch documentation. Default-skip:

- Infrastructure, CI, observability
- Translations
- Internal refactors, chores, tests, lint
- Copy tweaks that do not change documented behaviour

Keep tickets that change **user-facing behaviour** (PS) or **documented technical mechanics**
(DS). Realistically three to six survive out of fifteen to twenty.

Read the ticket and the diff. Do not judge from the title. A title says what someone intended
when they opened the branch.

### 3. Map each surviving ticket to a page

In this order, because they are in descending order of trust:

1. **A documentation link on the ticket.** If tickets are created with a documentation section
   naming their pages, that mapping was confirmed by a human at creation time. Treat as high
   confidence. Also check the ticket's remote links.
2. **The confirmed-mappings table above.** High confidence. Update it after the run.
3. **Semantic search across both spaces.** Lowest confidence. Read the candidates and choose on
   content, never on title.

Always state the reasoning and a confidence level, so a bad match can be caught before publish.

### 4. Work out what is actually stale

Read the ticket, the diff and the candidate page together. Identify the specific sentence,
section or table row that is now wrong or missing. Precision here is what keeps the proposal
reviewable in seconds instead of minutes.

Watch for scope: a change may be gated to one surface, one plan, one platform or one brand. If
so, say it, and frame the edit as a scoped mechanism rather than a blanket behaviour change.

**Inspect the page's images.** On a behaviour change, screenshots go stale in the same release
as the text, and nobody notices because prose review does not look at pictures. If the release
changes a documented screen, check what the page shows.

### 5. Propose

Per surviving ticket:

- ticket key, one line on what changed
- target page: id, title, space, why this page, confidence
- the exact proposed change, rendered readably
- any open question, including scope questions from step 4

### 6. Publish only what is approved

For each approved page, hand off to the documentation skill with the page id and the change.

## Learned rules

Each of these came from a real correction. Add to it.

- **Roughly one ticket in five touches documentation.** If a run proposes an edit for most of
  the release, the classifier is too loose and the reviewer will stop reading carefully.
- **Target the child, not the index.** A parent page whose body is only a children-listing macro
  has no content to edit. Fetch its children and pick the right one.
- **Match the page's own conventions.** If a page starts its sections at heading level two, use
  heading level two. The page's local style beats generic house style.
- **Two pages with near-identical titles usually cover different flows.** Check which flow the
  diff touches. Record the distinction in the mappings table the first time it bites.
- **Document the stable part, hold the unfinished part visibly.** When a release ships most of a
  behaviour and one sub-behaviour is still in flight, document what is stable and add a visible
  note for the rest. Do not silently document the intended end state.
- **Include concrete identifiers unless told otherwise.** Environment variable names, field
  names and exact values are what makes a page usable. Confirm the audience first: some product
  spaces want them, some do not.
- **Confirm scope when a diff carries more than the headline change.** A release that touches
  three things while the user asked about one is a question, not an invitation.
