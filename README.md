# Agent Skills

A collection of agent skills providing tailored instructions for common development workflows. By giving the agent actual domain expertise and repeatable workflows, you drastically reduce mistakes and ensure agents reliably complete tasks following best practices.

Skills are essentially simple folders of files that can be seen as complementary to MCP, where MCP gives an agent access to specialized tools and a skill teaches the agent how to use those tools for a specific task.

## Table of Contents

- [Quick Start](#quick-start)
- [Features](#features)
- [Installation](#installation)
- [Updating Skills](#updating-skills)
- [Available Skills](#available-skills)
  - [easy_mcp_add-server-annotations](#easy_mcp_add-server-annotations)
  - [obs-mcp](#obs-mcp)
  - [opencode_api-api-usage](#opencode_api-api-usage)
- [Repository Structure](#repository-structure)
- [External Resources](#external-resources)
- [Contributing](#contributing)
- [Code of Conduct](#code-of-conduct)
- [License](#license)

## Quick Start

Install a skill and let your AI agent do the rest:

```bash
# Install the OBS skill
npx skills add cdavis-code/skills --skill obs-mcp --agent universal

# Now ask your AI: "Start recording in OBS"
```

Your AI agent will automatically discover the skill and follow its instructions.

## Features

- **AI-ready instructions** - Skills teach your AI agent how to use tools correctly
- **Best practices built-in** - Reduce mistakes with proven workflows
- **Easy installation** - One command to add skills to any project
- **IDE agnostic** - Works with Cursor, Claude Code, GitHub Copilot, and more
- **Complements MCP** - Skills teach the "how", MCP provides the "what"

## Installation

To install all skills into your project, run the following command. The `--agent universal` flag puts them in the standard `.agents/skills` folder that most agents use.

```bash
npx skills add cdavis-code/skills --skill '*' --agent universal
```

To install a specific skill:

```bash
npx skills add cdavis-code/skills --skill <skill-name> --agent universal
```

## Updating Skills

To update all installed skills, run:

```bash
npx skills update
```

## Available Skills

### easy_mcp_add-server-annotations

Add `@Server`, `@Tool`, and `@Parameter` annotations to existing Dart code to expose methods as MCP tools or REST API endpoints.

**When to use:** Converting Dart libraries to MCP/REST servers, adding tool exposure to existing functions, or making Dart code callable via the Model Context Protocol or HTTP APIs.

**Installation:**
```bash
npx skills add cdavis-code/skills --skill easy_mcp_add-server-annotations --agent universal
```

### obs-mcp

Control a running OBS Studio instance through the obs-mcp-stdio MCP server. Exposes 60+ tools for scenes, sources, audio, streaming, recording, and more.

**When to use:** Connecting to OBS, listing/switching scenes, transforming sources, controlling audio inputs, starting/stopping streaming or recording, triggering hotkeys, or running any OBS WebSocket operation.

**Installation:**
```bash
npx skills add cdavis-code/skills --skill obs-mcp --agent universal
```

### opencode_api-api-usage

Work with the opencode_api package to send prompts to opencode server and interpret the message response structure.

**When to use:** Integrating with opencode.ai server API, sending prompts to sessions, processing responses with text/reasoning/tool parts, or building tools that interact with opencode programmatically.

**Installation:**
```bash
npx skills add cdavis-code/skills --skill opencode_api-api-usage --agent universal
```

## Repository Structure

```
skills/
├── easy_mcp_add-server-annotations/
│   └── SKILL.md
├── obs-mcp/
│   ├── SKILL.md
│   └── scripts/
├── opencode_api-api-usage/
│   └── SKILL.md
├── README.md
├── LICENSE
├── CODE_OF_CONDUCT.md
└── CONTRIBUTING.md
```

Each skill is a folder containing a `SKILL.md` file with detailed instructions for the AI agent on how to perform specific tasks. Some skills also include helper scripts, references, or assets.

## External Resources

- [Agent Skills Specification](https://agentskills.io/specification) - The standard for AI agent skills
- [Model Context Protocol (MCP)](https://modelcontextprotocol.io/) - Learn about MCP servers and tools

## Contributing

Please see [CONTRIBUTING.md](CONTRIBUTING.md) for information on how to contribute to this repository.

## Code of Conduct

Please see [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md) for our code of conduct.

## License

This project is licensed under the BSD License - see the [LICENSE](LICENSE) file for details.
