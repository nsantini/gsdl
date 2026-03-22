# Create PRD Examples

## Example 1: CLI Tool PRD

**Seed File** (`task-manager/seed.md`):
```markdown
# Task Manager CLI - Seed

## Initial Idea
Create a simple command-line task manager.

## Rough Features
- Add tasks
- List tasks
- Mark complete
- Maybe priorities?
```

**Agent Clarifying Questions**:
1. "Who is the primary user — developers, personal use, or teams?"
2. "Should tasks persist between sessions? If so, what storage format?"
3. "What priority levels would be useful? (e.g., high/medium/low, or 1-5 scale)"
4. "Should tasks have due dates?"
5. "Any specific CLI framework preference?"

**User Answers**:
- Personal use for developers
- Yes, persist to JSON file in home directory
- Simple high/medium/low priorities
- Yes, due dates would be helpful
- No preference, keep it simple

**Resulting PRD** (`task-manager/prd-task-manager-cli.md`):

```markdown
# Task Manager CLI - Product Requirements Document

## 1. Introduction/Overview
A lightweight command-line task manager for developers to quickly track personal todos without leaving the terminal. Stores tasks locally in JSON format for simplicity and portability.

## 2. Goals
- Provide fast task creation and retrieval from the command line
- Persist tasks between sessions
- Support basic organization via priorities and due dates
- Keep the interface minimal and intuitive

## 3. User Stories
- As a developer, I want to quickly add a task so that I don't forget it while coding
- As a developer, I want to list my tasks so that I can see what needs to be done
- As a developer, I want to mark tasks complete so that I can track my progress
- As a developer, I want to set priorities so that I focus on important tasks first
- As a developer, I want to set due dates so that I don't miss deadlines

## 4. Functional Requirements
1. The system must allow users to add a task with a title
2. The system must allow users to optionally set priority (high/medium/low, default: medium)
3. The system must allow users to optionally set a due date (YYYY-MM-DD format)
4. The system must list all tasks, showing title, priority, due date, and status
5. The system must support filtering tasks by status (pending/completed)
6. The system must support filtering tasks by priority
7. The system must allow marking a task as complete by ID
8. The system must allow deleting a task by ID
9. The system must persist tasks to ~/.tasks.json
10. The system must display tasks sorted by due date, then priority

## 5. Non-Goals (Out of Scope)
- Multi-user or team features
- Cloud sync or remote storage
- GUI or web interface
- Recurring tasks
- Task categories or projects (future consideration)

## 6. Design Considerations
CLI commands should follow common conventions:
- `task add "Task title" --priority high --due 2024-09-01`
- `task list [--status pending|completed] [--priority high|medium|low]`
- `task done <id>`
- `task delete <id>`

## 7. Technical Considerations
- Single-file script preferred for easy installation
- JSON storage for human-readable data
- Consider using a CLI parsing library for better argument handling

## 8. Success Metrics
- User can add, list, complete, and delete tasks in under 2 seconds each
- Data persists correctly across terminal sessions
- Zero learning curve for basic operations

## 9. Open Questions
- [ ] Should completed tasks be archived or deleted after a period?
- [ ] Should there be a `task edit` command for modifying existing tasks?
```

---

## Example 2: Web Feature PRD

**Seed File** (`saas-app/seed.md`):
```markdown
# User Settings Page - Seed

## Problem
Users can't update their profile or preferences in the app.

## Ideas
- Profile editing (name, avatar)
- Email preferences
- Theme selection (dark/light)
```

**Agent Clarifying Questions**:
1. "What profile fields should be editable beyond name and avatar?"
2. "What email preferences need to be configurable?"
3. "Are there other theme options beyond dark/light?"
4. "Should changes require email confirmation or password verification?"
5. "Any existing design system or components to use?"

**User Answers**:
- Name, avatar, and optional bio
- Marketing emails on/off, weekly digest on/off
- Just dark and light for now
- Password change should require current password
- Yes, using existing Tailwind component library

**Resulting PRD** (`saas-app/prd-user-settings-page.md`):

```markdown
# User Settings Page - Product Requirements Document

## 1. Introduction/Overview
A centralized settings page where users can manage their profile information, email preferences, and display theme. This improves user autonomy and reduces support requests for profile changes.

## 2. Goals
- Allow users to self-service profile updates
- Provide control over email communication preferences
- Support dark/light theme preference
- Maintain security for sensitive changes

## 3. User Stories
- As a user, I want to update my display name so that my profile reflects my current name
- As a user, I want to upload an avatar so that I'm recognizable to teammates
- As a user, I want to add a bio so that others know about me
- As a user, I want to control marketing emails so that I only receive emails I want
- As a user, I want to choose dark or light theme so that the app matches my preference
- As a user, I want to change my password securely so that I can maintain account security

## 4. Functional Requirements
1. The system must display a settings page accessible from the user menu
2. The system must allow users to edit their display name (max 100 characters)
3. The system must allow users to upload an avatar image (max 2MB, jpg/png)
4. The system must allow users to add an optional bio (max 500 characters)
5. The system must provide toggles for marketing emails and weekly digest
6. The system must provide a theme selector (dark/light) with immediate preview
7. The system must persist theme preference and apply on page load
8. The system must require current password to set a new password
9. The system must validate new password meets security requirements
10. The system must show success/error feedback for all save actions

## 5. Non-Goals (Out of Scope)
- Account deletion (separate feature)
- Two-factor authentication setup (separate feature)
- Notification preferences beyond email
- Custom theme colors

## 6. Design Considerations
- Use existing Tailwind component library
- Settings organized in sections: Profile, Email, Appearance, Security
- Responsive design for mobile
- Accessible form labels and error states

## 7. Technical Considerations
- Avatar uploads to existing S3 bucket
- Theme preference stored in localStorage and user profile
- Email preferences update via existing notification service API
- Password change uses existing auth module

## 8. Success Metrics
- 50% reduction in "change my profile" support tickets
- 80% of users who visit settings complete at least one update
- Theme preference used by 30% of users within first month

## 9. Open Questions
- [ ] Should avatar have cropping tool or accept as-is?
- [ ] Should bio support markdown formatting?
```

---

## Example 3: Handling Vague Requests

**User Request**: "I want to build something for managing customers"

**Agent Response**:

"I'd like to create a PRD for your customer management feature. To make sure I capture the right requirements, I have a few questions:

1. **What's the main problem you're solving?** (e.g., tracking customer info, managing sales pipeline, handling support)
2. **Who will use this?** (e.g., sales team, support agents, account managers)
3. **What information do you need to track per customer?** (e.g., contact info, purchase history, notes)
4. **Is this standalone or part of an existing app?**
5. **Any existing tools you're replacing or integrating with?**

Once I understand these, I'll create a seed file and then generate a detailed PRD for you."

---

## Question Patterns by Feature Type

### API/Backend Feature
- What endpoints are needed?
- What data structures are involved?
- Authentication/authorization requirements?
- Rate limiting or performance considerations?
- Error handling expectations?

### Frontend/UI Feature
- What pages/components are needed?
- What user interactions are required?
- Responsive design requirements?
- Accessibility considerations?
- Loading/error states?

### Integration Feature
- What external services are involved?
- What data needs to sync?
- How should failures be handled?
- Real-time or batch processing?
- Security/credential management?

### CLI Tool
- What commands are needed?
- What arguments/flags for each?
- Output format (text, JSON, table)?
- Configuration file support?
- Installation/distribution method?
