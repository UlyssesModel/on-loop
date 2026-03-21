---
name: on-doc
description: Generate documentation for the specified target using the documentation agent
user_invocable: true
argument: target file, directory, or description
---

# /on-doc

Standalone documentation generation using the documentation agent.

## Usage

```
/on-doc <file or directory to document>
/on-doc <description of what to document>
/on-doc                                  # document entire project
```

## Instructions

1. Read the user's target argument. If no argument, document the entire project.

2. Create a minimal `.on-loop/` workspace if one doesn't exist:
   - `state.json` with phase `"DOC"` and the target as prompt
   - Empty `agent-notes/` directory

3. Survey the project:
   - Read existing documentation
   - Read source code structure
   - Identify what documentation exists and what's missing

4. Dispatch the **documentation agent** (`agents/documentation.md`):
   - Provide the target scope
   - Provide project context

5. When complete, display:
   - Summary of documentation created/updated
   - List of files modified

## Notes

- This command runs only the documentation agent
- It creates/updates README.md, CLAUDE.md, API docs, and guides
- Existing documentation is updated in place, not overwritten
- Mermaid diagrams are included where appropriate
