# Netrevenue Skills Repository

Agent skills for Netrevenue's workflows and operations.

## About

This repository contains modular skills that enable AI agents to perform operational tasks across Netrevenue's tech stack. Each skill provides API documentation, verified endpoints, and workflow patterns for specific platforms or systems.

## Skills

### [GHL Operations](ghl-operations/)
Manage GoHighLevel sub-accounts via REST API for client account operations.

**Capabilities:** User management, calendar operations, phone number provisioning, pipeline auditing, account configuration.

---

## Repository Structure

```
skills/
├── README.md
├── {skill-name}/
│   ├── SKILL.md              # Main skill definition (routing, fundamentals)
│   └── references/           # Detailed API endpoint documentation
│       ├── {category}.md
│       └── ...
```

## Usage

Skills are designed for OpenClaw and LLM agents. Each skill directory contains:
- **SKILL.md**: Main skill file with routing logic, fundamentals, and workflow patterns
- **references/**: Detailed API endpoint documentation organized by category

To use a skill, provide the agent with the skill directory. The agent will reference the documented endpoints and patterns to perform requested operations.

## Contributing

When adding new skills:
1. Create a new directory: `{skill-name}/`
2. Add `SKILL.md` (routing layer, under 150 lines)
3. Add detailed reference docs in `references/` (under 350 lines each)
4. Verify all API endpoints with real API calls
5. Use realistic examples from actual API responses
6. Document edge cases, error conditions, and API quirks

