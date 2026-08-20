---
name: create-release
description: Create the next release version in the tracker and tag every current-sprint ticket that is ready to ship. Reads the active sprint, finds the highest released version, increments the patch, creates the version in each project, and appends it to all code-complete tickets. Use when asked to "create a release", "cut a release", "new release version", or "tag the sprint for release".
argument-hint: "[optional: explicit version, e.g. 2.2.5]"
---

# Create release

Cuts the next version and tags the sprint work that is ready for production.

## Configure

```
Projects:   PROJ (id <id>), OPS (id <id>)   both on board <BOARD_ID>
Tenant:     https://example.atlassian.net
Token:      ~/.atlassian-token
```

## Access

Use the REST API, not an MCP server. Version management is usually absent from MCP tool sets,
and a full-sprint query blows the response limit.

```bash
TOKEN=$(tr -d '\r\n' < ~/.atlassian-token)
BASE=https://example.atlassian.net
AUTH="you@example.com:$TOKEN"
```

## Steps

### 1. Find the active sprint

```bash
curl -s -u "$AUTH" "$BASE/rest/agile/1.0/board/<BOARD_ID>/sprint?state=active"
```

One active sprint covers all projects on the board. Note its numeric id.

### 2. Derive the version number

If the user gave one, use it. Otherwise:

```bash
for P in PROJ OPS; do
  curl -s -u "$AUTH" "$BASE/rest/api/3/project/$P/versions"
done
```

Take the highest **released** version across **all** projects, sorted by release date, and bump
the patch. Projects drift, because one of them skips versions, so the maximum across all of them
is the reference.

Ignore noise when picking the maximum: bucket versions, anything with a `beta` or `hotfix`
suffix, anything not matching `^\d+\.\d+\.\d+$`.

State the derived number before creating anything.

### 3. Pull the tickets in scope

```bash
curl -s -u "$AUTH" -G "$BASE/rest/api/3/search/jql" \
  --data-urlencode 'jql=sprint = <SPRINT_ID> AND status in ("PENDING PROD","QA","PENDING QA") ORDER BY project, status, key' \
  --data-urlencode "fields=summary,status,project,issuetype,fixVersions" \
  --data-urlencode "maxResults=200"
```

Those statuses are the release scope: code-complete, not yet in production. Anything earlier in
the workflow is not code-complete. Anything already done shipped under an earlier version.

Show the list grouped by project and status **before** writing anything.

### 4. Write a real description, then create the version

**The description must describe the actual scope.** Never ship a placeholder like "Release
2.2.5" or "Sprint 15 release". The description is what a reader sees on the releases page
without opening twenty tickets, and a placeholder wastes the one field that could have told
them something.

Build it from the step 3 summaries, grouped into themes. Each project gets its own description
covering only its own tickets. Semicolon-separated scope, nothing else:

> Builder step two and three state preservation and layout fixes; checkout defects (wallet
> refresh, annual plan flash, mobile header divider); passwordless sign-up and a friendly
> rate-limit message; account spacing; FAQ tabs removed; analytics tracking swap.

**Exclude as noise:** the sprint name, the ticket count, status breakdowns, dates, the version
number. The releases page already shows all of that. Keep it under about 280 characters. Plain
text, no markup, no em dashes. Name concrete surfaces and defects.

```bash
curl -s -u "$AUTH" -X POST -H "Content-Type: application/json" \
  --data "{\"name\":\"<VERSION>\",\"projectId\":<PID>,\"description\":\"<SCOPE>\",\"released\":false,\"archived\":false}" \
  "$BASE/rest/api/3/version"
```

Create it in every project even if one has no tickets in scope, and say so in its description.
Versions stay aligned. Leave `released: false`: this skill prepares a release, it does not ship
one.

### 5. Tag the tickets

Append, never replace. Tickets may already carry a bucket version, and overwriting loses it.

```bash
for K in <KEYS>; do
  CODE=$(curl -s -o ~/resp.json -w "%{http_code}" -u "$AUTH" -X PUT \
    -H "Content-Type: application/json" \
    --data '{"update":{"fixVersions":[{"add":{"name":"<VERSION>"}}]}}' \
    "$BASE/rest/api/3/issue/$K")
  if [ "$CODE" = "204" ]; then echo "OK   $K"; else echo "FAIL $K ($CODE)"; fi
done
```

`204` is success. Report every failure explicitly rather than in a summary count.

### 6. Verify

Re-query by version and check the count matches step 3. Report the per-project split.

## Rules

- **Never mark the version released.** That is a separate manual action after the production
  deploy.
- **Never leave a placeholder description.**
- **Append versions, never overwrite.**
- Create the version in every project, including empty ones.
- Do not create tickets, transition statuses or edit summaries. This skill touches versions and
  the version field, nothing else.
- If the derived version already exists anywhere, stop and ask. Do not silently reuse it or bump
  again.

## Output

- sprint name and date range
- version created, with each version id
- ticket count per project and the full key list grouped by status
- any ticket that already had a version, naming the old value
- any failures
