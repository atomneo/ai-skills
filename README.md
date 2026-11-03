# AI Skills

A collection of reusable agent skills for financial APIs. Each skill includes
agent instructions, practical workflows, optional UI metadata, and a local OpenAPI snapshot.
These are agent skills, rather than SDKs or MCP servers.

The shared `SKILL.md` instructions and references use the
[Agent Skills format](https://agentskills.io/specification) and are intended for
Claude Code, Codex, and other agents that support it. They do not depend on
agent-specific tools. The agent needs access to the installed files and an HTTP
client or command runner to call the APIs; the Python examples require Python 3.
`agents/openai.yaml` is optional OpenAI UI metadata, not a runtime dependency.

## Available skills

| Skill | Purpose | API |
| --- | --- | --- |
| [myfund-api](skills/myfund-api/SKILL.md) | Read portfolios, holdings, and history; import transactions, cash flows, broker files, and PPK files. | myFund.pl v1 (beta) |
| [ynab-api](skills/ynab-api/SKILL.md) | Read plans, accounts, categories, and transactions; synchronize changes and manage plan data. | YNAB v1, release 1.88.0 |

The README, skill instructions, workflows, and UI metadata are in English.
API snapshots, field names, parameter values, and UI labels retain the language
and terminology used by their providers. Agents can still respond in the user's language.

## Installation

Install globally with the [skills CLI](https://github.com/vercel-labs/skills),
choosing your agent interactively:

```sh
npx skills add atomneo/ai-skills --skill myfund-api -g
npx skills add atomneo/ai-skills --skill ynab-api -g
```

To select an agent explicitly, add `-a claude-code` for Claude Code or `-a codex`
for Codex. For example:

```sh
npx skills add atomneo/ai-skills --skill ynab-api -a claude-code -g
npx skills add atomneo/ai-skills --skill ynab-api -a codex -g
```

Omit `-g` to install into the current project. The CLI requires Node.js 22.20 or later.
This repository is private, so Git must be authenticated with an account that
has access. If necessary, use its SSH URL:

```sh
npx skills add git@github.com:atomneo/ai-skills.git --skill ynab-api -g
```

You can also install from a local checkout:

```sh
npx skills add /absolute/path/to/ai-skills --skill ynab-api -g
```

For manual installation, copy the entire skill directory, including `references/`,
into your agent's skills directory. In
[Claude Code](https://code.claude.com/docs/en/skills), use
`.claude/skills/<skill-name>/` for a project or `~/.claude/skills/<skill-name>/`
for a personal installation. Other agents have their own discovery locations.
Cloning this repository alone does not install its skills into an agent.

## Credentials

Provide credentials in the environment of the agent making API requests:

| Skill | Environment variable | Credential source |
| --- | --- | --- |
| myfund-api | `MYFUND_API_KEY` | myFund.pl: account settings, API key |
| ynab-api | `YNAB_ACCESS_TOKEN` | YNAB: Account Settings → Developer Settings → Personal Access Tokens |

Installation does not require API credentials. Skills do not automatically load
`.env` files. Keep credentials and private financial data out of this repository.
For YNAB integrations used by other account owners, use OAuth instead of collecting
personal access tokens.

## Usage

Example requests to your agent after installation:

```text
Use the myfund-api skill to list my portfolios and summarize the selected portfolio.
Use the myfund-api skill to prepare an import of this broker export.
Use the ynab-api skill to list my plans and summarize the selected plan.
Use the ynab-api skill to retrieve transactions for the selected account.
```

Explicit invocation syntax depends on the agent: Claude Code uses `/myfund-api`
or `/ynab-api`; Codex uses `$myfund-api` or `$ynab-api`. Agents may also select a
skill automatically from its description.

Preparation and execution of a write are separate operations. Skills require a
user instruction authorizing the write, resolve ambiguous targets, and avoid
blindly retrying writes whose outcome is uncertain.

## Repository structure

```text
skills/
  myfund-api/
    SKILL.md
    agents/openai.yaml
    references/
      openapi.yaml
      workflows.md
      provenance.json
  ynab-api/
    SKILL.md
    agents/openai.yaml
    references/
      openapi.yaml
      workflows.md
      provenance.json
```

Each directory under `skills/` is independently installable. Add future skills
using the same layout. There is no build step or runtime dependency for the skills.

## Maintenance

API snapshots date from 2026-11-03. Check each skill's metadata and provenance
before relying on a particular API release. Source paths in provenance records
refer to the original documentation workspace, not files in this collection.
When updating a skill, review the API documentation, update affected instructions
and schemas, and record the snapshot date and hashes in `provenance.json`.
Skill versions and API versions are independent.

Validate changed skills against the [Agent Skills specification](https://agentskills.io/specification):
check YAML frontmatter, names, descriptions, and relative links. Keep shared
instructions independent of agent-specific tools, paths, and invocation syntax;
put optional agent integration metadata in separate files such as `agents/openai.yaml`.
Validate API specifications with an OpenAPI-compatible validator and
check internal references. Do not test writes against real accounts without a
user instruction authorizing those operations.

## Attribution

The myFund skill retains its original MIT license in [LICENSE](LICENSE).
API documentation and schema content remain attributable to their providers;
the license does not relicense third-party documentation. YNAB API sources:
[documentation](https://api.ynab.com/) and [endpoint reference](https://api.ynab.com/v1).
