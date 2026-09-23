# Halfsies

Halfsies is an iOS and Android app backed by the Halfsies API; `REQUIREMENTS.md` section 1 describes the product. The project is specified but not built yet, so there is no application code.

## Where things are defined

Each topic lives in exactly one file. Read the owning file before working in its domain. Cite its rule IDs; don't restate its content. When content fits more than one file, security content goes in `SECURITY.md`.

| File | Owns | Read before |
|---|---|---|
| `REQUIREMENTS.md` | What the system does: scope, functional and non-functional requirements, API contract, privacy, quality gates | Changing observable behavior |
| `DESIGN.md` | The design language, including accessibility targets. `style-guide.html` is its rendered reference | UI work |
| `ARCHITECTURE.md` | Components, boundaries, data flow, dependency rules, technology choices | Structural changes |
| `SECURITY.md` | Security requirements, threat model, controls, trust-boundary enforcement, secure coding rules | Touching auth, input handling, data protection, or any trust boundary |
| `REQUIREMENT_TEMPLATE.md` | The required structure for new GitHub issues | Opening an issue |

## Rules

- Every new GitHub issue MUST follow `REQUIREMENT_TEMPLATE.md`, so each issue is a structured, testable requirement.
- When a change contradicts a spec file, update that file in the same PR.
- When specs conflict or a decision is missing, don't choose silently. Add it to the owning file's open-questions list and mark it `TO BE DECIDED`.

## Commands

Build, test, and run commands: TO BE DECIDED. None exist yet.
