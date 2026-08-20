# Propose, never publish

**Rule.** An agent that maintains documentation drafts the change and stops. A human accepts
each change individually, against a diff, before anything is written.

## Why, specifically

The pipeline has several steps and they do not fail equally.

| Step | How reliable | Consequence of an error |
|---|---|---|
| Collect what shipped | High. It is an API call against merge records. | Miss a ticket, nothing gets updated. Recoverable. |
| Decide it changed documented behaviour | Medium. Judgement, but bounded by a diff. | A false negative is silence. A false positive wastes a review. |
| Pick the page that is now stale | **Low.** Semantic matching over hundreds of pages. | Edits a page that was correct. **Not recoverable by the next run.** |
| Write the edit | High, once the target is known. | Visible in the diff. |

The unreliable step is the one that decides where the write lands, and its failure is the
only one that destroys existing correctness. Autonomy at any earlier step is cheap; autonomy
at that step is not.

Note the asymmetry against "the model is getting better". Improving the drafting step improves
the output. Improving the mapping step from 90% to 95% still means one page in twenty gets
damaged, silently, forever. The gate is not there because the model is weak. It is there
because the cost is asymmetric.

## What a proposal must carry

Not just the diff. Enough for a reviewer to reject it without opening four tabs.

- **The target,** with its id and title, so a wrong page is obvious on sight.
- **Why this page,** which of the routes below found it, in one sentence.
- **Confidence,** and the reason for it. "Ticket links to this page" and "closest text match
  of nine candidates" are not the same claim and must not read the same.
- **The exact change,** rendered readably, not as raw markup.
- **Open questions.** If a change might be scoped to one surface, or one plan, or one
  platform, say so rather than resolving it silently.

## Route the mapping, in order of trust

1. **An explicit link on the ticket.** If the ticket names its documentation page, that is
   authoritative. Worth creating that link upstream at ticket-writing time, when a human is
   present anyway, so the mapping problem is solved before it exists.
2. **A confirmed mapping from a previous run.** See [the mapping table](mapping-memory.md).
3. **Semantic search.** Lowest confidence. Read the candidate pages and choose on content,
   never on title. State that this is what happened.

## Reviewer experience is the real bottleneck

Approval is not free, and its cost sets the ceiling on the whole system.

- **Precision, not volume, is the metric.** Proposals generated is a vanity number. Measure
  accepted, edited-then-accepted, dismissed, expired, split by source and by reviewer.
- **A dismissal is a label.** Capture the reason in one click. It is the only honest training
  signal available for tuning the mapping step.
- **Queue age is a defect class.** A proposal nobody decided on is a routing failure. It means
  it reached someone with no standing to judge it. Fix routing, do not send reminders.
- **Low precision is self-reinforcing.** Reviewers who have dismissed six bad proposals stop
  reading the seventh carefully. The queue then launders bad edits into approved ones, and the
  human gate becomes theatre while still being claimed as a safeguard.

## "Checked, still correct" is a result

A run that finds nothing to change currently looks identical to a run that did not happen.
Publish the no-op: page X was verified against release Y on date Z. Freshness that can be
pointed at is most of what the reader, and any downstream agent, actually wants. It is also
the only way to distinguish a quiet system from a broken one, which is the same problem
[connector preflight](connector-preflight.md) solves at the other end of the pipe.
