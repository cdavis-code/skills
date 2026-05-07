# Contributing

Thank you for your interest in contributing to this project! This document provides guidelines and information for contributors.

## Getting Started

1. Fork the repository
2. Create a new branch for your feature or bug fix
3. Make your changes
4. Submit a pull request

## Providing Feedback on Existing Skills

Your feedback helps us improve the quality of our skills. When providing feedback:

1. **What agent are you using?** (e.g., Claude Code, Cursor, GitHub Copilot, etc.)
2. **What skill did you use?**
3. **What was your expected outcome?**
4. **What actually happened?**
5. **Include logs** showing the prompt you used and the steps the agent took (such as which skills it chose to use, what tools it invoked, etc.)

File an issue with this information so we can investigate and improve the skill.

## Requesting a New Skill

Before requesting a new skill:

1. Check the existing skills in the README to see if something similar already exists
2. Check open issues to see if someone has already requested it

If the skill doesn't exist, please file an issue with:

1. **Skill name** - A clear, descriptive name for the skill
2. **Use case** - What workflows or tasks would this skill help with?
3. **Example prompts** - How would users trigger this skill?
4. **Priority** - How important is this for your workflow?

## Pull Request Guidelines

### Before Submitting

- Ensure your code follows the existing style and conventions
- Test your changes thoroughly
- Update documentation if needed
- Make sure all tests pass (if applicable)

### Pull Request Process

1. Update the README.md if you're adding or modifying skills
2. Ensure your PR description clearly describes the changes
3. Link any related issues
4. Be responsive to review comments and make requested changes

### Skill Structure

Each skill should follow this structure:

```
skill-name/
└── SKILL.md
```

The `SKILL.md` file should contain:
- Clear description of when to use the skill
- Step-by-step instructions for the AI agent
- Example prompts
- Any relevant code snippets or templates

## Code of Conduct

Please note that this project is released with a [Code of Conduct](CODE_OF_CONDUCT.md). By participating in this project you agree to abide by its terms.

## Questions?

If you have questions about contributing, please:

1. Check existing issues for similar questions
2. File a new issue with your question
3. Be as specific as possible in your description

Thank you for contributing!
