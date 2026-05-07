# Agent Skills

A collection of agent skills providing tailored instructions for common development workflows. By giving the agent actual domain expertise and repeatable workflows, you drastically reduce mistakes and ensure agents reliably complete tasks following best practices.

Skills are essentially simple folders of files that can be seen as complementary to MCP, where MCP gives an agent access to specialized tools and a skill teaches the agent how to use those tools for a specific task.

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

## Contributing

Please see [CONTRIBUTING.md](CONTRIBUTING.md) for information on how to contribute to this repository.

## Code of Conduct

Please see [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md) for our code of conduct.

## License

This project is licensed under the BSD License - see the [LICENSE](LICENSE) file for details.

## Available Skills

| Skill | Description | Example prompt |
| --- | --- | --- |
| *Add your skills here* | *Description of what the skill does* | *Example of how to trigger the skill* |

## Getting Started

1. Browse the available skills in the table above
2. Install the skills you need using the installation command
3. The skills will be available in your `.agents/skills` directory
4. Your AI agent will automatically discover and use them when appropriate

## Repository Structure

```
skills/
├── SKILL_NAME_1/
│   └── SKILL.md
├── SKILL_NAME_2/
│   └── SKILL.md
├── README.md
├── LICENSE
├── CODE_OF_CONDUCT.md
└── CONTRIBUTING.md
```

Each skill is a folder containing a `SKILL.md` file with detailed instructions for the AI agent on how to perform specific tasks.
