---
name: implement-feature
description: "Generic feature implementation session. Accepts a feature specification in English or Portuguese, routes work to the appropriate agent(s), and applies the relevant rules and skills."
argument-hint: "Paste the feature specification. Optionally specify: agent (programmer | reviewer | tester | all), rules, and skills to apply."
user-invocable: true
---

<!--
-------------------- How to Use This Skill --------------------
Provide a feature specification and optionally configure which agent(s) and additional rules/skills to activate. The session will route work to the correct agent based on the requested task type.
-->

# Feature Implementation Session

### Feature Specification
> Describe the feature in English or Portuguese. Include: module, business rules, inputs, outputs, dependencies, and acceptance criteria.

### Agent
`<programmer | reviewer | tester | all>`  
**Default**: all (runs programmer → reviewer → tester in sequence)

### Rules (optional)
> List any additional rule files to apply, e.g. architecture-flow-and-code-style, create-usecase-with-tests

### Skills (optional)
> List any skills to activate, e.g. find-edge-cases, staged-security-review

---

## Agent Routing

### `programmer`

Activates the [Programmer Agent](../../agents/programmer.agent.md).

- Reads [`CONTEXT.md`](../../CONTEXT.md) for project context before implementing.
- Follows [`architecture-flow-and-code-style.rule.md`](../../rules/architecture-flow-and-code-style.rule.md).
- Creates files in order: entity/DTO → data_provider → logic_provider → trigger.
- Does NOT create tests.

### `reviewer`

Activates the [Reviewer Agent](../../agents/reviewer.agent.md).

- Applies [`find-edge-cases`](../find-edge-cases/SKILL.md) on the staged diff.
- Applies [`staged-security-review`](../staged-security-review/SKILL.md) on the staged diff.
- Proposes 3 commit message options using [`commit-message.rule.md`](../../rules/commit-message.rule.md).
- Requires staged changes (`git add` before running).

### `tester`

Activates the [Tester Agent](../../agents/tester.agent.md).

- Creates unit, integration, and E2E tests under `tests/`.
- Follows [`create-usecase-with-tests.rule.md`](../../rules/create-usecase-with-tests.rule.md).
- Covers all scenarios including wrong-type inputs and edge cases.
- Does NOT modify source code.

### `all` (default)

Runs all three agents in sequence:
1. **Programmer** — implements the feature.
2. **Reviewer** — reviews the staged changes for edge cases and security.
3. **Tester** — creates tests covering all scenarios.

---

## Examples

### Example — Full Feature Session

```
### Feature Specification
Implement a usecase to deactivate a user account.
Module: users
Input: userId (string, required)
Business rules:
  - The user must exist; otherwise throw notFound.
  - If the user is already inactive, throw a conflict error.
  - Set the user's status to INACTIVE and record the deactivation timestamp.
Output: updated User entity

### Agent
all

### Rules
architecture-flow-and-code-style

### Skills
find-edge-cases
```

---

### Example — Programmer Only

```
### Feature Specification
Add a GET /users/:id endpoint that returns the user profile.
Module: users
Dependencies: UserRepository, LoggerService

### Agent
programmer
```

---

### Example — Reviewer Only (after staging changes)

```
### Feature Specification
Review the changes I just staged for the payment module.

### Agent
reviewer

### Skills
staged-security-review
find-edge-cases
```

---

### Example — Tester Only

```
### Feature Specification
Create tests for the DeactivateUserUseCase in the users module.
The usecase calls UserRepository.findById and UserRepository.save.
Success: returns the updated user.
Failure: throws notFound when user does not exist; throws conflict when already inactive.

### Agent
tester
```
