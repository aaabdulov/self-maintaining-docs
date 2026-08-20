---
name: jira-research
description: Upstream research before writing a ticket: parsing a support conversation, parsing a chat thread, inspecting images already attached to a ticket, reading referenced external pages, and investigating the codebase. Invoked by the jira router when a ticket has a support report, a chat link, existing attachments or a non-obvious root cause. Do not invoke directly, start from /jira.
---

# Upstream research

Module of the `jira` router. Loaded when there is a source to read before writing.

The purpose is narrow: a ticket written from a one-line summary of a source is a ticket written
from someone's memory of it. Read the source.

## Support conversations

**Read the original description, not just the thread.** The first message is what the customer
actually reported. By reply four the conversation is about a workaround, and a ticket written
from the tail end describes the workaround failing rather than the original problem.

Extract and keep separate:

- What the customer did, in their words
- What they expected
- What happened
- Environment: platform, browser, plan, locale, anything that scopes the report
- Attachments, which usually carry the actual evidence

**Separate report from diagnosis.** Support agents theorise, helpfully, and their theory arrives
in the same paragraph as the observation. In the ticket, the observation is the bug and the
theory is at most a labelled hypothesis.

Note that support search may need REST rather than an MCP server, which sometimes exposes read
tools without a search tool.

## Chat threads

**Fetch and parse the thread. Do not paste a link and move on.** A link is not context: it is a
promise that someone will go and get the context, which they will not.

Walk the whole thread. Decisions get reversed three messages later, and a ticket written from
the first half of a thread implements a decision the team already abandoned.

**Never screenshot colleagues.** Paraphrase what was decided. A screenshot puts a person's
wording, name and picture into a permanent artefact they did not agree to, to convey information
a sentence conveys better. If a screenshot was attached before this rule, remove it and
paraphrase.

## Images already on the ticket

**Always look at them before editing.** Two reasons, and they compound.

A raw markup edit that rebuilds the document tree will silently drop media nodes it did not
account for. And an existing screenshot frequently contradicts the change being described,
because it was taken before the last two releases.

If more than one image is attached and it is not obvious what each shows, ask. An image captioned
by assumption is worse than an uncaptioned one.

## Referenced external pages

If the ticket points at a document, a design file or a specification, read it before commenting
on the ticket. A comment that talks past the linked artefact is how a thread ends up with two
people confidently discussing different things.

Verify the reference still resolves. Links to moved or deleted pages are common, and a dead link
in a ticket is a small, permanent tax on everyone who opens it.

## Codebase investigation

For a non-obvious root cause, read the code and report what is actually there.

**State which branch you searched.** This is the rule that matters most in this file. An agent
once concluded a capability did not exist, because it searched the default branch while the work
sat on an open one. The conclusion was confident, well argued and wrong, and it was corrected
only because a human happened to remember the branch.

Generalised: **an absence claim must say where it looked.** "No handler exists for X" is
unverifiable. "No handler for X on `main` as of <sha>" can be checked, and can be wrong in a way
someone notices.

Other rules for this section:

- **Report what is there, not what should be.** Redesign suggestions belong in a separate,
  clearly labelled part of the ticket, or in a conversation.
- **Point at current code.** If the repository contains a legacy copy of the application, exclude
  it explicitly and name the paths you did search.
- **Everything produced here goes under a heading that marks its origin.** A code read by an
  agent is a strong hypothesis. Label it so that when one turns out to be wrong, the reader knows
  what else to re-check.
- **A private-repository 404 can be an authorisation failure, not a wrong path.** Before
  concluding a repository or file does not exist, confirm the token still authenticates at all.
