# Skill Execution Logic

This repository's skill system is prompt-driven. A skill is not mainly a TypeScript plugin with custom business logic. Instead, it is usually a `SKILL.md` file plus frontmatter, loaded into a `Command` with `type: 'prompt'`, then expanded into the conversation or executed in a forked sub-agent.

## 1. What a skill is

- A skill becomes a `PromptCommand` / `Command` at runtime.
- The key type definitions are in [src/types/command.ts](./src/types/command.ts#L25) and [src/types/command.ts](./src/types/command.ts#L175).
Important frontmatter fields:

- `description`
- `when_to_use`
- `allowed-tools`
- `arguments`
- `shell`
- `context`
- `agent`
- `model`
- `effort`
- `hooks`
- `paths`

## 2. How skills are loaded

- Local skills are mainly loaded from `.claude/skills/<skill-name>/SKILL.md`.
- The loader parses YAML frontmatter with `parseFrontmatter()`, then normalizes shared fields with `parseSkillFrontmatterFields()`.
- The parsed result is converted into a runtime command by `createSkillCommand()`.

Key code:

- Frontmatter parsing: [src/utils/frontmatterParser.ts](./src/utils/frontmatterParser.ts)
- Shared skill field parsing: [src/skills/loadSkillsDir.ts:185](./src/skills/loadSkillsDir.ts#L185)
- Command creation: [src/skills/loadSkillsDir.ts:270](./src/skills/loadSkillsDir.ts#L270)
- Directory scanning: [src/skills/loadSkillsDir.ts:638](./src/skills/loadSkillsDir.ts#L638)

## 3. How a skill becomes visible to the model

- Skills are collected into the command registry.
- Model-invocable prompt skills are filtered by `getSkillToolCommands()`.
- Their names and descriptions are injected into the system prompt so the model can decide when to use them.

Key code:

- Skill filtering for model use: [src/commands.ts:563](./src/commands.ts#L563)
- Slash-command skill filtering: [src/commands.ts:586](./src/commands.ts#L586)
- System prompt assembly: [src/constants/prompts.ts:444](./src/constants/prompts.ts#L444)
- Skill tool prompt: [src/tools/SkillTool/prompt.ts](./src/tools/SkillTool/prompt.ts)

## 4. How a skill is executed

There are two main entry points:

- User types `/skill-name`
- The model invokes `SkillTool`

Key code:

- Slash-command expansion path: [src/utils/processUserInput/processSlashCommand.tsx:827](./src/utils/processUserInput/processSlashCommand.tsx#L827)
- Tool-based invocation path: [src/tools/SkillTool/SkillTool.ts:332](./src/tools/SkillTool/SkillTool.ts#L332)

Both paths eventually call the skill's `getPromptForCommand()` and turn the markdown content into actual model-visible prompt text.

## 5. What happens inside `getPromptForCommand()`

When a file-based skill is executed, `createSkillCommand()` defines a `getPromptForCommand()` implementation that:

1. Adds a base-directory header when the skill has a directory.
2. Substitutes declared arguments into the markdown body.
3. Replaces `${CLAUDE_SKILL_DIR}` with the skill directory.
4. Replaces `${CLAUDE_SESSION_ID}` with the current session ID.
5. Expands embedded shell snippets in the markdown body.
6. Returns the final prompt content as text blocks.

Key code:

- Expansion pipeline: [src/skills/loadSkillsDir.ts:344](./src/skills/loadSkillsDir.ts#L344)
- `${CLAUDE_SKILL_DIR}` substitution: [src/skills/loadSkillsDir.ts:356](./src/skills/loadSkillsDir.ts#L356)
- `${CLAUDE_SESSION_ID}` substitution: [src/skills/loadSkillsDir.ts:365](./src/skills/loadSkillsDir.ts#L365)
- Shell expansion call: [src/skills/loadSkillsDir.ts:375](./src/skills/loadSkillsDir.ts#L375)

## 6. How scripts inside a skill are executed

Scripts under a skill directory are not auto-discovered or auto-run.

They run only when `SKILL.md` explicitly references them through shell expansion syntax:

- Inline form: ``!`command` ``
- Block form:

````md
```!
some command
```
````

Typical usage:

```md
---
shell: bash
---

Collect environment details:

!`${CLAUDE_SKILL_DIR}/scripts/collect.sh`
```

Execution flow:

1. `shell:` frontmatter is parsed.
2. The skill body is scanned for `!` shell snippets.
3. Each snippet goes through tool permission checks.
4. The shell command is executed with `BashTool` or `PowerShellTool`.
5. The command output replaces the original `!` snippet in the prompt text.
6. The final expanded prompt is sent to the model.

Key code:

- `shell:` parsing: [src/utils/frontmatterParser.ts:337](./src/utils/frontmatterParser.ts#L337)
- `shell:` validation: [src/utils/frontmatterParser.ts:351](./src/utils/frontmatterParser.ts#L351)
- Shell expansion patterns: [src/utils/promptShellExecution.ts:48](./src/utils/promptShellExecution.ts#L48)
- Shell expansion function: [src/utils/promptShellExecution.ts:69](./src/utils/promptShellExecution.ts#L69)
- Permission check before execution: [src/utils/promptShellExecution.ts:97](./src/utils/promptShellExecution.ts#L97)
- Actual tool call: [src/utils/promptShellExecution.ts:115](./src/utils/promptShellExecution.ts#L115)

Important boundary:

- MCP skills are treated as untrusted remote content, so their markdown body does not execute `!` shell snippets. See [src/skills/loadSkillsDir.ts:371](./src/skills/loadSkillsDir.ts#L371).

## 7. Inline vs fork execution

Skills can run in two modes:

- `inline`: the expanded prompt is injected into the current conversation.
- `fork`: the skill runs in a separate sub-agent with its own context and token budget.

Key code:

- Forked execution entry: [src/tools/SkillTool/SkillTool.ts:122](./src/tools/SkillTool/SkillTool.ts#L122)
- Skill tool main entry: [src/tools/SkillTool/SkillTool.ts:332](./src/tools/SkillTool/SkillTool.ts#L332)

## 8. Permission model

- Invoking a skill itself goes through `SkillTool.checkPermissions()`.
- Executing `!` shell snippets inside a skill goes through tool permission checks again.
- A skill that only uses "safe properties" may be auto-allowed.

Key code:

- Skill permission checks: [src/tools/SkillTool/SkillTool.ts:453](./src/tools/SkillTool/SkillTool.ts#L453)
- Safe-property allowlist: [src/tools/SkillTool/SkillTool.ts:876](./src/tools/SkillTool/SkillTool.ts#L876)

## 9. Bundled skills and extracted files

Bundled skills may ship with embedded reference files. On first invocation, those files are extracted to disk and the prompt is prefixed with the extracted base directory, so the skill can reference local files just like a disk-based skill.

Key code:

- Bundled skill definition: [src/skills/bundledSkills.ts:15](./src/skills/bundledSkills.ts#L15)
- Lazy extraction wrapper: [src/skills/bundledSkills.ts:59](./src/skills/bundledSkills.ts#L59)
- File extraction: [src/skills/bundledSkills.ts:131](./src/skills/bundledSkills.ts#L131)

## 10. End-to-end summary

The complete flow is:

1. Load `SKILL.md`
2. Parse frontmatter
3. Build a prompt-based `Command`
4. Expose skill metadata to the model through the system prompt
5. Invoke the skill through slash command or `SkillTool`
6. Expand arguments, directory/session placeholders, and optional `!` shell snippets
7. Apply permissions, hooks, model/effort overrides, and execution context
8. Run inline or in a forked sub-agent
9. Continue the conversation using the expanded skill prompt
