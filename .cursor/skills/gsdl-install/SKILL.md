---
name: gsdl-install
description: Install GSDL Cursor rules into the current project. Creates .cursor/rules/*.mdc files that enable /gsdl, /gsdl-setup-project, /gsdl-create-prd, /gsdl-create-plan, and /gsdl-execute-plan slash commands. Use when the user says "gsdl install", "install gsdl rules", or wants to set up the GSDL slash commands.
---

# GSDL Install

Creates the `.cursor/rules/` files that expose each GSDL skill as a Cursor `/` slash command.

## Steps

1. Create `.cursor/rules/` in the project root if it doesn't exist.

2. Write each file below **only if it does not already exist**. Never overwrite existing files.

3. Confirm to the user which files were created and which were skipped (already existed).

---

## Files to create

### `.cursor/rules/gsdl.mdc`

```
---
description: GSD — Run the full project pipeline from idea to implementation (setup → PRD → task list → implement). Use when building something end-to-end.
alwaysApply: false
---

Read the skill file at `.cursor/skills/gsdl/SKILL.md` using the Read tool, then follow all instructions within it.
```

### `.cursor/rules/gsdl-setup-project.mdc`

```
---
description: GSD Setup — Initialize a new project structure with tasks directory and seed file.
alwaysApply: false
---

Read the skill file at `.cursor/skills/gsdl-setup-project/SKILL.md` using the Read tool, then follow all instructions within it.
```

### `.cursor/rules/gsdl-create-prd.mdc`

```
---
description: GSD PRD — Generate a Product Requirements Document from a seed file or user input.
alwaysApply: false
---

Read the skill file at `.cursor/skills/gsdl-create-prd/SKILL.md` using the Read tool, then follow all instructions within it.
```

### `.cursor/rules/gsdl-create-plan.mdc`

```
---
description: GSD Plan — Break down a PRD into a detailed implementation task list.
alwaysApply: false
---

Read the skill file at `.cursor/skills/gsdl-create-plan/SKILL.md` using the Read tool, then follow all instructions within it.
```

### `.cursor/rules/gsdl-execute-plan.mdc`

```
---
description: GSD Execute — Work through an implementation plan step-by-step with progress tracking.
alwaysApply: false
---

Read the skill file at `.cursor/skills/gsdl-execute-plan/SKILL.md` using the Read tool, then follow all instructions within it.
```

---

## After creating files

Tell the user:

> GSDL rules installed. The following slash commands are now available in Cursor chat:
> - `/gsdl` — full pipeline orchestrator
> - `/gsdl-setup-project` — initialize project structure
> - `/gsdl-create-prd` — generate a PRD
> - `/gsdl-create-plan` — break a PRD into tasks
> - `/gsdl-execute-plan` — implement the plan
>
> You may need to reload Cursor for the rules to appear in the `/` picker.
