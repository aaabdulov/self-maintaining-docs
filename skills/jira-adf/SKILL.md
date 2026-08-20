---
name: jira-adf
description: Atlassian Document Format and REST mechanics for tickets: authentication, a Windows-safe scripting pattern, Unicode safety, embedding and sizing images, comment formatting, issue linking, moving tickets between projects, transition ids and custom field quirks. Invoked by the jira router when a ticket is actually being written. Do not invoke directly, start from /jira.
---

# Ticket markup and REST mechanics

Module of the `jira` router. Loaded when a ticket needs images, layouts, a raw markup edit,
linking, a transition, or a move.

> Trimmed: organisation-specific field ids, account ids, workflow states and project constants
> are removed. The mechanics and the traps are what transfer.

## Authentication

```bash
TOKEN=$(tr -d '\r\n' < ~/.atlassian-token)
BASE=https://example.atlassian.net
AUTH="you@example.com:$TOKEN"
```

Basic auth with an API token. Note that the same token can behave differently across endpoint
families on the same tenant, so a 401 on one path does not mean the token is wrong.

## Read and write raw markup

```bash
# read
curl -s -u "$AUTH" "$BASE/rest/api/3/issue/<KEY>?fields=description"
# write
curl -s -u "$AUTH" -X PUT -H "Content-Type: application/json" \
  --data @payload.json "$BASE/rest/api/3/issue/<KEY>"
```

Descriptions are a structured document tree, not markdown. Anything non-trivial means building
that tree, which means a script.

## The Windows-safe scripting pattern

Three rules, each learned by losing content.

**Write the payload to a file, never inline it.** Shell quoting and JSON quoting fight, and the
loser is a description with a mangled code block in it.

**Put temp files under the home directory, not `/tmp`.** On Windows, a script run from a POSIX
shell and the language runtime it invokes resolve `/tmp` to different places. The write appears
to succeed and the read finds nothing, or worse, finds a stale file.

**Set the encoding explicitly on both ends.** Read and write UTF-8 by name. Do not rely on the
platform default, which on Windows is not UTF-8, and which fails silently by substituting
characters rather than raising.

```python
import io, json
payload = {...}
path = os.path.expanduser("~/payload.json")
io.open(path, "w", encoding="utf-8").write(json.dumps(payload, ensure_ascii=False))
```

## Unicode safety

Non-ASCII in a description is the most common way a scripted edit corrupts a ticket, because
nothing errors. Symptoms: an em dash becomes three characters, a name loses its accent, a
non-Latin string becomes question marks.

Round-trip before publishing: read the payload back, decode, and compare against what you
intended to write. Cheap, and it catches every variant of this.

## Embedding images

1. Upload the file as an attachment to the issue, with the no-check header the API requires.
2. Take the attachment id from the response.
3. Reference that id from a media node in the document tree.

**Size deliberately.** An image at full width dominates the ticket and pushes the acceptance
criteria below the fold. In a comment, go one step smaller than you would in a description: a
comment is a remark, and a full-width image in one reads as the main event.

**Exporting a design frame adds canvas padding.** A frame export includes the surrounding
canvas, so the design sits centred in a much larger image. Embedded as-is, it letterboxes and
takes a wrong aspect ratio. Crop to content first: build a solid image from the corner pixel,
which is the canvas background and is usually not white, difference against it, threshold, take
the bounding box, crop with a small margin. Then pick the display width from the cropped ratio.

**Inspect images already on a ticket before editing it.** A raw edit that rebuilds the document
tree will drop media nodes it does not know about. Read them, keep them, and look at them: an
existing screenshot often contradicts the change being described.

## Comments

Same document format, different conventions. Shorter, one idea, images a notch smaller. Verify
after posting: comment rendering diverges from description rendering for some node types, and
the only reliable check is to read it back.

## Issue linking

Links are their own endpoint, not a field on the issue. Fetch the available link types first
rather than guessing a name; they are configurable per tenant and the inward and outward names
differ.

Comment threading is usually a UI-only construct with no API equivalent. A reply posted through
the API becomes a top-level comment. Do not promise a threaded reply you cannot produce.

## Moving tickets between projects

Use the tracker's own move function rather than create-and-close. Move preserves history,
comments, attachments and links. Recreating loses all of it and leaves a confusing pair of
tickets, one of which will be found first by search.

## Field and workflow quirks

- **Custom fields are per-tenant.** The same field has a different id on a different site. Never
  copy an id between tenants. Fetch the field list and match on name.
- **Custom fields are often omitted from default responses.** Ask for them explicitly, or
  conclude wrongly that a field is empty.
- **Transition ids are per-workflow, not per-status name.** The same status name reached from
  two issue types can need two different ids. Fetch the available transitions for the issue
  rather than hardcoding.
- **A query that excludes a value with a not-equals operator also drops issues where the field
  is empty.** An exclusion can silently zero a result set. Add an explicit is-empty clause.
- **A search endpoint can be deprecated under you.** When a query that used to work starts
  returning nothing rather than erroring, suspect the endpoint before suspecting the query.
- **MCP search results can be truncated while reporting that more pages exist.** For anything
  where completeness matters, use REST and page it yourself.
