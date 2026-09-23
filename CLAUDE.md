# Halfsies

Halfsies is an iOS and Android app backed by the Halfsies API. Two people each enter a starting point. Halfsies returns places where both travel times are roughly equal, ranked by fairness and total travel time. One person proposes a place, the other accepts, and both get directions. The project is specified but not built yet, so there is no application code.

## Where things are defined

Each topic lives in exactly one file. Read the owning file before working in its domain. Cite its rule IDs; don't restate its content.

| File | Owns | Read before |
|---|---|---|
| `REQUIREMENTS.md` | What the system does: scope, API contract, privacy, security requirements, quality gates | Changing observable behavior |
| `DESIGN.md` | The design language. `style-guide.html` is its rendered reference | UI work |
| `ARCHITECTURE.md` | Components, boundaries, data flow, dependency rules, technology choices | Structural changes |
| `SECURITY.md` | Threat model, controls, secure coding rules | Touching auth, input handling, data protection, or any trust boundary |
| `REQUIREMENT_TEMPLATE.md` | The required structure for new GitHub issues | Opening an issue |

## Rules

- Every new GitHub issue MUST follow `REQUIREMENT_TEMPLATE.md`, so each issue is a structured, testable requirement.
- When a change contradicts a spec file, update that file in the same PR.
- When specs conflict or a decision is missing, don't choose silently. Add it to the owning file's open-questions list and mark it `TO BE DECIDED`.

## Commands

Build, test, and run commands: TO BE DECIDED. None exist yet.
