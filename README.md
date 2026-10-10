# caloo skills

The `caloo` [Agent Skill](https://agentskills.io): teaches any agent to set up and maintain a
[Caloo](https://caloo.ai) booking page through the Caloo MCP server, and to book a time on someone's Caloo page
through the public API. One skill, the same text for every agent.

```
skills/caloo/
  SKILL.md                         the map: which job, which reference, the owner workflow, the hard rules
  references/setup.md              owner onboarding and later changes, tool by tool
  references/stdlib.md             caloo.d.ts explained, every helper with one example
  references/patterns.md           worked rules.js cases with the slots they produce (engine-tested)
  references/booking.md            booking on someone's behalf through the public API
  references/troubleshooting.md    tool errors, explain, get_errors, sync problems
.claude-plugin/                    Claude Code plugin + marketplace manifests (skill + MCP server)
```

## Install

| Route                | Command or step                                                                                                                                                                   |
| -------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Claude Code          | `/plugin marketplace add amlalabs/caloo-skills` then `/plugin install caloo@caloo` (skill + MCP server) |
| Any agent with `npx` | `npx skills add amlalabs/caloo-skills` (skill only) |
| Manual               | Download `https://caloo.ai/skills/caloo/SKILL.md` and the `references/` files (listed at `https://caloo.ai/skills`) into a `caloo` folder in the agent's skills directory, e.g. `~/.claude/skills/` |
| No file system       | Paste the SKILL.md URL, or call the `get_skill` tool, which returns SKILL.md and, with a `reference` argument, any references file                                                |

The Claude Code plugin also registers the MCP server; with the other routes, add `https://caloo.ai/mcp` to the agent
yourself.

## Development

The public `amlalabs/caloo-skills` repository is generated from the Caloo monorepo by `tools/publish-skills.sh`;
changes made directly there are overwritten by the next publish. In the monorepo:

- `pnpm validate` checks the frontmatter, that SKILL.md stays under 300 lines and links every reference,
  that every MCP tool name matches `TOOL_NAMES` in `@caloo/types`, that every stdlib identifier exists in
  `caloo.d.ts`, and that `.claude-plugin/` carries SKILL.md's name and version.
- `pnpm test` runs every case in `references/patterns.md` through `@caloo/engine` and asserts its slots.
- Bump `version` in SKILL.md together with `SKILL_VERSION` in `packages/types` and both `.claude-plugin/`
  manifests; the server tells agents with an older skill to update.
