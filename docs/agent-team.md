# Agent team

This project uses GitHub Copilot CLI in a Codespace to orchestrate a team of prebuilt custom agents, defined in `.github/agents/`, to plan, design, build, and validate Mona's Project Pulse dashboard.

## Orchestrator

- **Model**: Claude Opus 4.7 (copilot)
- **Responsibility**: Coordinates the Planner, Coder, and Designer agents. Breaks the request into phases, assigns explicit file scopes, decides what can run in parallel vs. sequentially, verifies the integrated result, and reports the final outcome. Does not implement work itself and does not perform git operations.
- **Definition**: `.github/agents/orchestrator.agent.md`

## Planner

- **Model**: Claude Opus 4.7 (copilot)
- **Responsibility**: Researches the repository and documentation, then produces a practical implementation plan for the Orchestrator, including ordered steps, file assignments, dependencies, parallelizable work, edge cases, and validation expectations. Does not write code.
- **Definition**: `.github/agents/planner.agent.md`

## Coder

- **Model**: GPT-5.5 (copilot)
- **Responsibility**: Implements code within the file scope assigned by the Orchestrator (e.g., `app/index.html`, `app/project-data.json`, and support configuration like `.vscode/launch.json`). Keeps behavior explicit, deterministic, and testable, and validates changes before reporting completion.
- **Definition**: `.github/agents/coder.agent.md`

## Designer

- **Model**: Gemini 3.1 Pro (copilot)
- **Responsibility**: Owns UI/UX, accessibility, information architecture, and visual design within the scope assigned by the Orchestrator, such as `app/styles.css`. Delivers a polished dashboard with project cards, status badges, priority treatment, and responsive layout.
- **Definition**: `.github/agents/designer.agent.md`

## How the team builds Project Pulse

Using GitHub Copilot CLI in a Codespace, the learner works through the Orchestrator to coordinate the rest of the team:

1. The **Orchestrator** asks the **Planner** for an implementation plan covering `app/index.html`, `app/styles.css`, `app/project-data.json`, and `.vscode/launch.json`.
2. The **Orchestrator** parses the plan into phases and delegates design decisions to the **Designer** and implementation to the **Coder**, keeping overlapping file scopes in separate phases.
3. The **Designer** and **Coder** produce the static dashboard (HTML, CSS, and JSON) and the launch configuration for previewing it.
4. The **Orchestrator** verifies the integrated result and reports the final outcome; the learner remains in control of all git operations (stage, commit, push) through Copilot CLI prompts.
