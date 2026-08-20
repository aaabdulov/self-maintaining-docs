# Connector preflight

**Rule.** Before a scheduled agent spends a run, it proves it can read every source it depends
on. If it cannot, it stops, notifies a human, and writes nothing. It does not degrade, it does
not fall back to a partial answer, it does not report success.

## The incident

A routine read a chat workspace, cross-referenced it with the tracker, and posted a digest. The
chat connector's authorisation lapsed. The connector did not error. It returned empty results.

The routine did what it was told: it read the source (nothing there), cross-referenced (nothing
to cross-reference), found no issues, and reported that everything looked fine. Which is exactly
what it reports on a genuinely quiet day.

Nobody noticed, because the output was indistinguishable from a good outcome. A monitoring
system that says "all clear" when it is blindfolded is worse than no monitoring, because it
actively spends trust.

## Why the obvious fixes do not work

**"Check whether the result is empty."** Empty is legitimate. A quiet week is quiet. You cannot
infer brokenness from an empty result set, which is precisely the problem.

**"Wrap it in error handling."** There was no error. The call succeeded and returned zero rows.

**"Let it degrade to the sources that do work."** This is the tempting one, and it is wrong. A
digest built from two of three sources looks like a digest built from three. Unless the
degradation is louder than the output, it will be read as complete. Partial results need to be
harder to ignore than no results.

## The gate

Cheap, and it runs before anything else:

1. **One assertion per source.** Not connectivity, a real read whose answer you know is
   non-empty: list your own channels, fetch a known issue, read the repository root.
2. **Fail closed.** Any assertion fails, the run ends. No reads, no writes, no partial digest.
3. **Push a notification, do not post.** A failure notice in the normal output channel is one
   more message nobody reads. It goes to whoever can re-authorise, out of band.
4. **Say which source and what the assertion returned.** "Preflight failed" costs a debugging
   session. "Chat preflight failed: channel listing returned 0, expected >0" does not.

Exempt the routines that genuinely do not need a source. A gate that fails for reasons unrelated
to the run gets removed within a month.

## Where the gate lives matters

Put the gate in the routine's prompt and you have as many copies as routines, drifting
independently. Put it in the file each routine already reads on the way in, and it ships once.
See [thin loaders](thin-loaders.md).

## The general form

Every one of these is the same failure, and each needs its own assertion:

- Source unreachable, returns empty instead of erroring.
- Search index stale, returns fewer results than exist.
- Token scoped too narrowly, so private items are invisible and read as absent.
- Agent searches the wrong branch, ref, or workspace and reports the absence as fact.

The last one is worth stating as a rule of its own: **an agent asserting that something does not
exist must say where it looked.** Absence claims are the ones that survive review, because there
is nothing to check them against.
