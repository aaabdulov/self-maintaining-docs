# Thin loaders

**Rule.** A scheduled agent's prompt contains a pointer, not a spec. The spec lives in a
version-controlled repository the agent mounts and reads. A missing repository is a hard stop,
never a fallback.

## The problem it solves

Cloud-scheduled agents cannot see your local machine. The obvious move is to paste the
instructions into the prompt. That works until there are eight routines and the same rule
appears in five of them.

What follows is predictable:

- A rule gets fixed in one routine and stays broken in four.
- Nobody can tell which copy is current.
- Editing a routine means re-sending tens of kilobytes of prompt, so edits get batched, so they
  get deferred, so the copies drift further.
- Two routines disagree, and their outputs disagree, and the disagreement looks like a finding.

A prompt is a copy. Copies drift. The fix is the same as anywhere else in software: one source,
many readers.

## The shape

The prompt shrinks to roughly this:

```
Read `.claude/skills/<name>/routines/<routine>-prompt.md` from the mounted repository
and follow it exactly. If the file is not there, stop, push a notification, print a
directory listing of the working directory, and end the run.
```

Everything else lives in the repo. A fix is a commit and a push. The next run has it. No prompt
is touched, so the change is reviewable as a diff instead of arriving as a wall of pasted text.

## The trap: conditional fallback

The tempting version reads:

```
IF the working directory contains the repository, read the spec from it.
OTHERWISE follow the instructions below.
```

This is worse than either option alone. When the mount fails, the routine runs the inline copy,
which is by construction the version that existed when the loader was written, missing every fix
since. The run succeeds. The output looks normal. The regression is silent and dated.

**A missing spec is a hard stop.** No inline fallback, ever. The loader has nothing to read, so
it says so and ends. Same principle as [connector preflight](connector-preflight.md): make the
failure louder than the success, because only one of the two gets read carefully.

## Migrating an existing prompt without retyping it

Moving a large prompt into a repo by hand is where content gets silently mangled. Do it
mechanically:

1. Pull the live prompt out of the scheduler's API as data.
2. Write it to the repo path unmodified.
3. Assert the written file is byte-identical to the live prompt. Fail the migration if not.
4. Commit, push, then replace the live prompt with the loader.

Nothing is retyped, so the paste that makes a migration risky never happens.

## Verify the mirror, do not infer it

If a hook syncs your working copy into the repo, do not trust it because the tree is clean. A
clean tree means both "already mirrored" and "never noticed the edit". Observed both, in the
same week, from the same hook.

Two cheap checks, both unambiguous:

- Hash the source file against its mirrored copy.
- Compare local `HEAD` against the remote ref.

Run them after editing anything a scheduled agent reads. "It looked fine" is not a check.
