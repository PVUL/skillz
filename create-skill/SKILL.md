---
name: create-skill
description: A skill to create new AI agent skills and SKILL.md files.
---

# create-skill

This skill provides instructions for creating new agent skills, which are markdown files (typically named `SKILL.md`) containing guidelines and instructions for AI agents.

## When to use

- The user asks you to "create a new skill", "write a skill", or "make a skill".
- You are tasked with codifying a workflow, a set of project-specific rules, or a standard operating procedure into an AI-executable format.

## Instructions

When tasked with creating a new skill, follow these steps to generate a high-quality, actionable skill file:

1. **Understand the Goal:**
   - Clarify the exact purpose of the skill. What should the agent be able to do autonomously after reading it?
   - Identify the triggers (when should the agent use this?).

2. **Initialize the Skill:**
   - ALWAYS create new skills in the `~/repos/skillz` directory so they are saved in the repository and can be referenced elsewhere.
   - Navigate to `~/repos/skillz` and use the command `npx skills init <skill-name>` to generate a boilerplate `SKILL.md` file.
   - Alternatively, you can manually create a directory named `<skill-name>` inside `~/repos/skillz` and write the `SKILL.md` file inside it.
   - If the user just wants the content without saving, output the markdown directly or write it to an artifact.

3. **Write the Frontmatter:**
   - Every skill MUST begin with a YAML frontmatter.
   - `name`: A concise, kebab-case name (e.g., `git-workflow`, `add-new-route`).
   - `description`: A brief, 1-2 sentence summary of what the skill does.

4. **Structure the Markdown Content:**
   - **H1 Header:** `# <skill-name>` matching the frontmatter name.
   - **Introduction:** A brief description summarizing the skill.
   - **`## When to use`:** Provide a bulleted list of clear, unambiguous triggers. Think about what user prompts, file types, or repository states should cause the agent to activate this skill.
   - **`## Instructions`:** Provide the core logic. Use numbered steps for linear workflows, or bullet points for general guidelines.

5. **Best Practices for Writing AI Instructions:**
   - **Use Imperative Voice:** Use clear action verbs ("Run this command", "Read this file", "Create a function").
   - **Be Specific & Concrete:** Provide exact commands to run, specific file paths to check, or code snippets to use as templates. Avoid vague instructions.
   - **Prioritize Context Gathering:** Tell the agent what to check *before* acting (e.g., "First, read `package.json` to verify dependencies").
   - **Define Edge Cases & Error Handling:** Instruct the agent on what to do if a step fails or if an expected file is missing.

### Example Skill Template

```markdown
---
name: example-skill
description: Briefly describe what this skill does and the problem it solves.
---

# example-skill

A brief overview of the skill's purpose and any necessary background information.

## When to use
- When the user asks to [action].
- When modifying files in the `[directory-name]/` directory.
- When you encounter [specific error or situation].

## Instructions
1. **Analyze Context:** First, check [file/state] to understand the current configuration.
2. **Execute Task:** Run the command `[command]` or apply the following pattern:
   ...code pattern here...
3. **Verify:** Ensure that the task succeeded by running `[test-command]` or checking `[output-file]`.
```
