---
name: jira
description: Create or edit tickets (epics, stories, bugs, tasks) in the tracker. Use when asked to create, update or manage issues, or given a ticket key or URL. Handles formatting, transitions, project routing and field quirks by delegating to its modules.
---

# Ticket authoring

A router. It holds the rules about what a good ticket is, and delegates mechanics to two
modules. Invoke it as `/jira`; it invokes the modules itself.

> **This is a trimmed version.** The internal original is roughly four times this size, most of
> the difference being organisation-specific routing, field ids, workflow states and house
> style. What is kept here is the part that transfers.

## Why it is split

The original was a single file of about 127KB. Past a certain size, rules inside a long
instruction file start getting missed: not ignored on purpose, just not applied consistently,
and the ones in the middle fare worst. Splitting it into a router plus conditionally loaded
modules fixed that.

Two constraints made the split work:

- **The router invokes the modules itself.** The human-facing entry point stays one command.
  Anything that requires the user to remember which of four skills to call will be called wrong.
- **A module is loaded only when its trigger fires.** Markup mechanics load when a ticket
  actually needs images or a raw edit. Research loads when there is an upstream source to read.
  Most tickets need neither.

## Companion modules: invoke these yourself, do not wait to be asked

| Module | Load when |
|---|---|
| `jira-adf` | The ticket needs images, layouts, tables, a raw markup edit, issue linking, a transition, or a move between projects. |
| `jira-research` | There is an upstream source: a support thread, a chat thread, existing attachments on the ticket, a linked external page, or a root cause that needs the codebase read. |

## Decision tree

1. **Create or edit?** An edit to a ticket where work has started also needs a comment saying
   what changed and why. A silent description edit is invisible to whoever is building it.
2. **Issue type.** Delivery work, defect, or investigation. Post-incident and prevention work is
   a task, not a story: there is no user in the story.
3. **Does it need research?** Load `jira-research`.
4. **Does it need images or raw markup?** Load `jira-adf`.
5. **Write it.** Rules below.
6. **After creating:** check the post-creation list at the end.

## Length budget

The shortest ticket that is still buildable.

| Type | Budget |
|---|---|
| Configuration or mechanical change | 60 to 100 words |
| Story | 120 to 200 words |
| Epic | 200 to 300 words |
| Comment | about 80 words |

Length is not thoroughness. A long ticket gets skimmed, and the acceptance criteria at the
bottom are what gets skipped. If the work genuinely needs more than the budget, that is usually
a sign it is more than one ticket, or that a decision is being deferred into the ticket rather
than made before it.

## Write for the person who has to build it

- **Plain language.** No abstraction the reader has to unpack. Developers reading a ticket are
  context-switching into it; every unfamiliar construction costs them a re-read.
- **No internal shorthand a reader cannot decode.** If a term only makes sense to whoever was in
  the meeting, expand it or drop it.
- **No phase jargon.** Words like "discovery", "hardening" and "phase two" describe a process,
  not work. Say what changes.
- **Never write a ticket key as plain text.** Make it a link. A key that cannot be clicked gets
  copy-pasted into a search box, or ignored.
- **Never let a numbered list collide with numbering from an external document.** If a spec has
  its own numbering, quote it rather than renumbering it. Two competing numbering schemes in one
  ticket cause the wrong thing to get built.

## Titles

**Delivery tickets lead with a verb. Epics and above are named as things.**

- Story, bug, task: "Add X", "Fix Y", "Remove Z". A verb states what will be different.
- Epic, initiative, idea: a noun phrase. It names an area, not an action, because it will
  outlive several actions.

Improve a title whenever you touch a ticket for another reason. A bad title is read hundreds of
times in a board view and costs more, cumulatively, than a bad description.

## Acceptance criteria

- **Do not invent criteria nobody asked for.** An invented criterion is either scope somebody
  now has to build, or a line that gets ignored, and both are worse than its absence. Performance
  budgets nobody agreed to fund are the classic case.
- **When a decision lands, dead criteria die.** If a decision makes a criterion obsolete, delete
  it in the same pass, across the whole ticket tree. Stale criteria that contradict a decision
  are how the wrong thing gets built by someone who read the ticket and not the thread.
- **A bug report contains only what was observed.** Steps, expected, actual. Not a theory of the
  cause presented as fact. If there is a hypothesis, label it as one.

## Testing sections

Written for someone with no context on the tool and no special access.

The failure mode: a testing note that assumes the reader can open a browser console, query a
database, or read a log aggregator. QA often cannot, so the note is unusable and the ticket goes
back and forth twice before anyone says why.

Describe what to click and what should happen on screen.

## Marking AI provenance

Any section generated from a code investigation goes under a heading that names its origin, for
example **Suggested by Claude**. Comments written by an agent say so.

This is not modesty. A reader weighs a claim differently depending on where it came from, and an
agent's read of a codebase is a strong hypothesis rather than a verified fact. Hiding the origin
removes information the reader needs. It also makes the failure recoverable: when one of these
turns out to be wrong, everything under that heading gets re-checked, and nothing else does.

## Notifying people

**Tag by involvement, not by appearance of a name.** Notify only the people who must act. A
mention that requires no action trains the recipient to ignore the next one.

Mention the assignee only when the edit changes what they have to build.

Do not use a ticket to police status. If a ticket is in the wrong state, that is a conversation,
not a comment thread.

## Structure

- **Epic:** the outcome, why now, what is in scope, what is explicitly not, and the child
  tickets.
- **Story:** context in one or two sentences, the change, acceptance criteria, testing, links.
- **Bug:** steps, expected, actual, environment, evidence.

Documentation links get their own section, never mixed into dependencies. This section is also
what makes automated documentation maintenance possible later: it records the human-confirmed
mapping from this work to the pages that describe it, at the one moment a human is definitely
present. See the `release-docs` skill, which reads it as its highest-confidence mapping route.

## Post-creation checklist

- Correct project and issue type
- Title follows the verb or noun-phrase rule
- Within the length budget
- No invented acceptance criteria, no dead ones left behind
- Testing section usable by someone without special access
- Every key and URL is a link
- Documentation section present when the work changes documented behaviour
- Generated sections marked
- Only the people who must act were notified
