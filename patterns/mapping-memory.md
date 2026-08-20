# The mapping table is the memory

**Rule.** When a human corrects an agent's mapping, the correction is written back into the
skill file. A correction that lives only in the conversation is not a fix, it is a favour you
will repeat.

## The problem

Deciding which documentation page a change belongs to is the expensive, unreliable step. It
costs a semantic search over hundreds of pages, several page reads, and a judgement call that
is wrong often enough to need review.

Then the reviewer corrects it: "not that page, this one." Run again next month, and the agent
pays the full cost again and can easily make the same mistake again. The human is not reviewing
a system that learns, they are a permanent component of it.

## The fix

Keep a table in the skill file itself, and require every run to update it.

```markdown
## Confirmed mappings (maintain this)

Check here FIRST, before semantic search. Whenever a run confirms or a reviewer
corrects a mapping, add or fix the row.

| Feature area | Page | Space | id | Notes |
|---|---|---|---|---|
| Sign-in, all methods, lockout | Sign in | PS | `<id>` | Rate-limit detail lives in DS `<id>`. |
| Registration, standalone form | Sign up | PS | `<id>` | Distinct from the in-flow variant below. |
| Registration, inside purchase flow | Sign up (purchase flow) | PS | `<id>` | A change to one of these rarely affects the other. Check which flow the diff touches. |
```

The lookup order becomes: explicit link on the ticket, then this table, then semantic search.
Each run is cheaper and more accurate than the last, in the areas the team actually touches,
which are the areas that matter.

## Why in the skill file rather than a database

Because the skill file is what the agent reads, what a human edits, and what version control
tracks. A correction becomes a commit with an author and a date. You can see the table grow, and
you can see which areas the team keeps working in, which turns out to be useful on its own.

A separate store would need loading, schema, and its own drift problem, to hold thirty rows.

## What belongs in the notes column

The notes are where the near-misses go, and they are worth more than the ids.

- **Pairs that are easy to confuse.** Two pages with near-identical titles covering different
  flows. Record the distinction the first time someone explains it.
- **Index pages.** A parent page whose body is only a children-listing macro has no content to
  edit. Record that the real target is a specific child, or the agent will confidently edit the
  index.
- **Cross-links.** Page A is the user-facing description, page B the technical mechanics, and a
  change often touches both. One row should tell you that.
- **Conventions local to a page.** If a page starts its sections at heading level two, match it.
  Generic house style loses to the page's own style.

## The general principle

This is the same idea as [propose, never publish](propose-never-publish.md), one step further
on. Human review is only worth its cost if the review is capital rather than maintenance. If
the tenth review is as expensive as the first, the system is not learning, it is just being
supervised, and it will be switched off as soon as somebody prices the supervision.
