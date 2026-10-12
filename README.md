# daycal skills

[Daycal](https://daycal.io) gives you a page where other people can book time with you, and it gets your real
situation right (trips in other time zones, holidays, Friday cutoffs, focus blocks, daily limits, several calendars)
because you just tell your AI assistant. Any schedule. Just say it.

The `daycal` [Agent Skill](https://agentskills.io) teaches that assistant both jobs:

- **For the page's owner:** set up the page and keep it right. Under the hood, availability is a short `rules.js`
  written against the daycal standard library; the assistant writes and maintains it through Daycal's tools (the
  MCP server, or the same tools over plain REST with an API token), always previewing and getting the owner's go-ahead before pushing. Already on Calendly or Cal.com? It imports
  the meeting types and hours from there.
- **For someone booking:** find a time on a person's Daycal page over the public API, confirm it with them, and
  book it over the API.

One skill, the same text for every agent.

```
skills/daycal/
  SKILL.md                         the map: which job, which reference, the owner workflow, the hard rules
  references/setup.md              owner onboarding and later changes, tool by tool
  references/stdlib.md             daycal.d.ts explained, every helper with one example
  references/patterns.md           worked rules.js cases with the slots they produce (engine-tested)
  references/booking.md            booking on someone's behalf: slots and booking over the API
  references/troubleshooting.md    tool errors, explain, get_errors, sync problems
.claude-plugin/                    Claude Code plugin + marketplace manifests (skill + MCP server)
```

## Install

| Route                | Command or step                                                                                                                                                                   |
| -------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Claude Code          | `/plugin marketplace add amlalabs/daycal-skills` then `/plugin install daycal@daycal` (skill + MCP server) |
| Any agent with `npx` | `npx skills add amlalabs/daycal-skills` (skill only) |
| Manual               | Download `https://daycal.io/skills/daycal/SKILL.md` and the `references/` files (listed at `https://daycal.io/skills`) into a `daycal` folder in the agent's skills directory, e.g. `~/.claude/skills/` |
| No file system       | Paste the SKILL.md URL, or call the `get_skill` tool, which returns SKILL.md and, with a `reference` argument, any references file                                                |

The Claude Code plugin also registers the MCP server; with the other routes, add `https://daycal.io/mcp` to the agent
yourself. An agent without an MCP client calls the same tools over REST with an API token from
`https://daycal.io/token` (`POST https://daycal.io/api/me/tools/<tool name>`; see `references/setup.md`, step 0).

## Development

The public `amlalabs/daycal-skills` repository is generated from the Daycal monorepo by `tools/publish-skills.sh`,
which CI runs after each production deploy, tagging each version `skill-v<version>` (one content per version); changes
made directly there are overwritten by the next publish. In the monorepo:

- `pnpm validate` checks the frontmatter, that SKILL.md stays under 300 lines and links every reference,
  that every MCP tool name matches `TOOL_NAMES` in `@daycal/types`, that every stdlib identifier exists in
  `daycal.d.ts`, and that `.claude-plugin/` carries SKILL.md's name and version.
- `pnpm test` runs every case in `references/patterns.md` through `@daycal/engine` and asserts its slots.
- Bump `version` in SKILL.md together with `SKILL_VERSION` in `packages/types` and both `.claude-plugin/`
  manifests; the server tells agents with an older skill to update.
