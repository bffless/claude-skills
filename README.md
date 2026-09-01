# @bffless/skills

Agent skills for [BFFless](https://bffless.app) — a self-hosted static asset hosting platform with AI-powered pipelines.

These skills give your AI coding agent domain knowledge about BFFless features so it can help you build, deploy, and configure your projects.

Originally built as a Claude Code plugin, the skills are plain markdown and also work with the open-source [`skills`](https://www.npmjs.com/package/skills) CLI — so you can install them into any agent that reads skill files from your project.

## Skills Included

| Skill                 | Description                                              |
| --------------------- | -------------------------------------------------------- |
| **authentication**    | Cross-domain auth, login relay, cookie sessions          |
| **authorization**     | Global and project roles, API keys, permission model     |
| **bffless**           | Platform overview, key concepts, and feature summary     |
| **cache-and-storage** | Cache rules, storage backends, API keys for CI/CD        |
| **chat**              | AI chat widget/full-page, skills, streaming, persistence |
| **pipeline-to-skill** | Generate a project-scoped skill from a BFFless proxy rule set |
| **pipelines**         | Backend automation with handler chains and DB Records    |
| **proxy-rules**       | Forward requests to backend APIs, eliminate CORS         |
| **repository**        | Deployments, aliases, content browser, rollback          |
| **rules-as-code**     | Manage proxy rule sets as code in git with the bffless CLI and CI sync |
| **share-links**       | Token-based sharing for private deployments              |
| **traffic-splitting** | A/B testing, canary deployments, weighted routing        |
| **upload-artifact**   | GitHub Action for uploading builds to BFFless            |
| **use-bff-state**     | React hook for server-side state with Data Tables        |
| **workflow**          | Author and deploy Workflow harness implementations with the @bffless/workflow CLI |

## Install

### Claude Code (plugin marketplace)

```bash
# Add the marketplace
/plugin marketplace add bffless/skills

# Install the plugin
/plugin install bffless
```

Or via CLI:

```bash
claude plugin install bffless --scope user
```

### Any agent (via `npx skills`)

The same skills repo works with the open-source [`skills`](https://www.npmjs.com/package/skills) CLI:

```bash
# List available skills
npx skills add bffless/skills --list

# Install all skills into your project
npx skills add bffless/skills

# Install a specific skill
npx skills add bffless/skills --skill chat
```

This clones the repo and copies the selected skill files into your project so any compatible agent can read them.

### Local Development

To use a local copy with Claude Code (e.g. when contributing):

```bash
git clone https://github.com/bffless/skills.git
claude --plugin-dir ./skills
```

## Usage

Once installed, your agent automatically uses these skills when you ask about BFFless features. For example:

- "Set up a proxy rule to forward /api requests to my backend"
- "Add AI chat to my site with streaming"
- "Configure traffic splitting for a canary deployment"
- "Set up the upload-artifact GitHub Action for my repo"

In Claude Code, skills are invoked by name with the `bffless` namespace:

```
/bffless:chat
/bffless:proxy-rules
/bffless:pipelines
```

## Developer Skills vs BFFless Pipeline Skills

This package contains **developer-facing agent skills** — they teach your coding agent about the BFFless platform so it can assist you while you build.

These are different from **BFFless pipeline skills**, which are markdown files you create in your own project's `.bffless/skills/` directory. Pipeline skills are deployed with your site and loaded by the AI chat handler at runtime to give your chatbot domain-specific knowledge.

|              | Developer Skills (this package)         | Pipeline Skills (your project)          |
| ------------ | --------------------------------------- | --------------------------------------- |
| **Purpose**  | Help your agent help _you_ build on BFFless | Give _your chatbot_ domain knowledge |
| **Location** | Installed via Claude Code plugin or `npx skills` | `.bffless/skills/` in your project repo |
| **Used by**  | Your coding agent (your dev tool)       | BFFless AI chat handler (your users)    |
| **Deployed** | npm / `skills` CLI                      | `bffless/upload-artifact` GitHub Action |

## Documentation

- [BFFless Docs](https://docs.bffless.app)
- [Getting Started](https://docs.bffless.app/getting-started/quickstart)
- [Skills](https://docs.bffless.app/features/claude-code-plugin)
- [Chat Feature](https://docs.bffless.app/features/chat/)
- [Pipelines](https://docs.bffless.app/features/pipelines/)

## Contributing

1. Fork this repo
2. Add or update skills in `plugins/bffless/skills/<skill-name>/SKILL.md`
3. Each skill needs YAML frontmatter with `name` and `description`
4. Open a PR — CI validates all skills automatically

## License

[O'Saasy](./LICENSE.md)
