# self-maintaining-docs

Claude Code skills that keep a team's documentation and delivery records true to what
actually shipped, and a set of patterns for running agents against real systems without
letting them lie to you.

This is an extracted, genericised subset of a larger internal skill library that runs
against live Jira, Confluence, GitHub and Slack workspaces. Company names, tenant URLs,
page ids, channel ids, colleague names and product detail have been removed, and the
skills that were too tenant-bound to genericise are not here. What remains is the
architecture and the rules, which are the transferable part.

Every identifier is a placeholder: `ORG`, `PROJ`, `OPS`, space keys `PS` and `DS`,
`example.atlassian.net`. Substitute your own.

## The problem

Documentation drifts the moment a release ships. Nobody notices, because nothing breaks.
It surfaces later, when someone acts on a page that stopped being true four releases ago.
That someone is increasingly an agent, at which point stale docs stop being an annoyance
and start being a correctness problem with no human in the loop to smell that the answer
is wrong.

The naive fix is to have an LLM rewrite the docs. That fails for a reason worth naming:
the hard part is not writing the update, it is knowing *which page* is now wrong. Semantic
mapping from a code change to a documentation page is fuzzy, and a confident edit to the
wrong page is worse than no edit at all, because it damages a page that was previously
correct.

So the loop here is deliberately narrow:

    what shipped  ->  which of it changed documented behaviour  ->  which page says so
                  ->  the exact proposed edit  ->  a human approves  ->  publish

Only the last arrow writes anything.

## Skills

| Skill | What it does |
|---|---|
| [`release-docs`](skills/release-docs/SKILL.md) | Takes a release, resolves it to the tickets it contains, classifies which ones changed documented behaviour, maps each to the existing page that is now stale, and proposes the exact edit. Never publishes unattended. |
| [`release-audit`](skills/release-audit/SKILL.md) | Reconciles what a release *claims* to contain against what actually merged to the release branch, in both directions. Answers "did anything ship untested" and "does this version list work that never merged". |
| [`create-release`](skills/create-release/SKILL.md) | Cuts the next version and tags the code-complete tickets, with a scope description derived from the tickets rather than a placeholder. |
| [`jira`](skills/jira/SKILL.md) | Router for ticket authoring. Holds the quality rules, delegates mechanics to its modules. Trimmed here: the internal version is roughly four times this size. |
| [`jira-adf`](skills/jira-adf/SKILL.md) | Module. Atlassian Document Format and REST mechanics: raw ADF edits, image embedding, encoding safety, transitions, linking. |
| [`jira-research`](skills/jira-research/SKILL.md) | Module. Upstream research before writing a ticket: support threads, chat threads, existing attachments, and the codebase. |

## Patterns

The skills are the worked examples. These are the four ideas underneath them, each written
after something went wrong.

| Pattern | One line |
|---|---|
| [Propose, never publish](patterns/propose-never-publish.md) | The mapping step is fuzzy, so the write step needs a human. Confidence must be stated, not implied. |
| [Connector preflight](patterns/connector-preflight.md) | An agent that cannot read its sources must stop loudly, not produce a confident empty answer. |
| [Thin loaders](patterns/thin-loaders.md) | A prompt is a copy. Keep the spec in a repo the agent reads, so a fix ships once and reaches every routine. |
| [The mapping table is the memory](patterns/mapping-memory.md) | A correction that does not survive the run is not a fix. Write confirmed mappings back into the skill. |

## Why the failure modes are the interesting part

Three incidents shaped most of what is here.

**A connector deauthorised silently.** A scheduled routine kept running, read nothing, found
nothing wrong, and reported success. An empty result and a broken integration were
indistinguishable from the outside. Now every routine that depends on an external source has
to prove the source is live before it spends the run, and stop with a notification if it
cannot. A green run that read nothing is the most dangerous output an agent can produce.

**An agent searched the wrong branch.** It concluded that a capability did not exist, because
it searched the default branch while the work sat on an open one. The conclusion was
confident, well-argued and wrong. "State which branch you searched" became a required output
field. The general rule: when an agent asserts an absence, it has to say where it looked.

**A prompt fell back to a stale copy.** Routines read their spec from a mounted repo, with an
inline fallback for when the repo was missing. The repo failed to mount, the fallback ran, and
the fallback predated several fixes. Conditional fallbacks turn a loud failure into a quiet
regression. Now a missing repo is a hard stop.

The common shape: in all three the system reported success. Anything downstream that trusted
the output had no way to tell. Most of the rules in these skills exist to make failures
visible rather than to make successes prettier.

## Using them

Drop a skill directory into `~/.claude/skills/`. Claude Code discovers it from the
frontmatter `description`, or invoke it directly as `/release-docs`. The modules under
`jira-adf` and `jira-research` are invoked by the `jira` router, not directly.

They assume tokens at `~/.github-token` and `~/.atlassian-token`, and they use the REST APIs
rather than MCP servers in the places where the REST behaviour is more predictable. Each
skill says where and why.

## Licence

MIT. See [LICENSE](LICENSE).
