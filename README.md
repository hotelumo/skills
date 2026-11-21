# Hotelumo Agent Skills

[Agent Skills](https://agentskills.io) for
[Hotelumo](https://www.hotelumo.com/) — hotel property management:
reservations and guests, rooms and room types, rate plans and per-day prices,
live availability and booking requests.

## Install

```bash
npx skills add hotelumo/skills
```

Works with any agent that supports the Skills standard — Claude Code, Cursor,
Codex, Gemini CLI, GitHub Copilot, OpenCode, and
[others](https://agentskills.io/clients). Update later with
`npx skills update`.

To install a single skill:

```bash
npx skills add hotelumo/skills@hotelumo
npx skills add hotelumo/skills@hotel-operations
```

## Skills

| Skill                                                  | Backed by                                                   | Use it for                                                                                               |
| ------------------------------------------------------ | ----------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| [`hotelumo`](skills/hotelumo/SKILL.md)                 | the [`hotelumo` CLI](https://github.com/hotelumo/cli) (npm) | Reading and changing reservations, guests, rooms, rates and availability from the shell — `npx hotelumo` |
| [`hotel-operations`](skills/hotel-operations/SKILL.md) | the Hotelumo MCP connector (`https://mcp.hotelumo.com/mcp`) | Hotel operations and stock counts through MCP tools in hosts like Claude, ChatGPT, Cursor, and Codex     |

The two are not interchangeable. The CLI skill drives the full REST API with
an account or an API key, including writes. The MCP skill drives the hosted
OAuth connector, which reads operations and changes nothing but a stock count,
and deliberately exposes no guest identities, individual reservations,
payments or credentials.

Both need a Hotelumo account with at least one hotel —
[sign up](https://app.hotelumo.com), then `hotelumo login` (CLI) or connect
the MCP server (OAuth).

## Other entrypoints

- **CLI** — `npm install -g hotelumo` or `npx hotelumo`, source at
  [hotelumo/cli](https://github.com/hotelumo/cli).
- **MCP connector** — `https://mcp.hotelumo.com/mcp` (Streamable HTTP,
  OAuth). Setup for every MCP client at
  [hotelumo.com/developers](https://www.hotelumo.com/developers/#mcp).
- **Claude Code plugin** — the connector bundled with the `hotel-operations`
  skill: [hotelumo/claude-plugin](https://github.com/hotelumo/claude-plugin).
- **REST API** — API keys and reference at
  [hotelumo.com/developers](https://www.hotelumo.com/developers/).

## License

[Apache-2.0](LICENSE)
