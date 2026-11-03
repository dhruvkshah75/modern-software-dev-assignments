# Modern Software Development — My Learning Journey

My personal learning fork of [mihail911/modern-software-dev-assignments](https://github.com/mihail911/modern-software-dev-assignments), the assignment repository for Stanford's [CS146S: The Modern Software Developer](https://themodernsoftware.dev).

I'm working through these assignments independently to understand modern software development with AI coding agents, MCP servers, and reusable skills. This repository will collect my implementations, completed assignments, experiments, and notes as I progress. I may adapt the original requirements to explore ideas that are useful to me.

## Assignment progress

| Week | Topic | Status | Materials |
|---|---|---|---|
| 1 | Trace dissection of a real Claude Code session | Not started | [Assignment](week1/assignment.md) · [Writeup](week1/writeup.md) |
| 2 | Build an MCP server for a third-party API | Planning complete; implementation next | [Assignment](week2/assignment.md) · [Project plan](week2/plan.md) · [Writeup](week2/writeup.md) |
| 3 | Author a skill | Not started | [Assignment](week3/assignment.md) · [Writeup](week3/writeup.md) |
| 4 | Make a messy repository agent-ready | Not started | [Assignment](week4/assignment.md) · [Writeup](week4/writeup.md) |
| 5 | Agent-readiness report | Not started | [Assignment](week5/assignment.md) · [Writeup](week5/writeup.md) |

The Python environment and initial dependencies are set up. Assignment implementations and completed writeups will be added as I work through them.

## Starting with Week 2: GitHub Backlog Triage MCP Server

The first project will connect **Codex to GitHub through a custom Python MCP server**, helping investigate and organize issues across selected repositories.

Example workflow:

> Find open authentication issues across my selected repositories. Recommend existing labels, assignees, and milestones, show me a preview, and apply the changes I approve.

Planned capabilities:

- Search issues and read their descriptions and discussions.
- Discover existing labels, assignable users, and open milestones for each repository.
- Preview changes before adding labels or assignees and setting a milestone.
- Authenticate through GitHub OAuth, cache tokens, and refresh them automatically.
- Restrict operations to an explicit repository allowlist and return actionable errors.
- Test through the MCP protocol, integrate with Codex, and later add a reusable triage skill.

The project uses **Python 3.12, FastMCP, and local stdio transport**. Codex makes recommendations; the server retrieves data, validates inputs, and performs GitHub operations.

The [detailed Week 2 plan](week2/plan.md) pairs each implementation phase with concepts to learn, exercises, and working checkpoints. **The server is planned and has not been implemented yet.**

## Local setup

Install Python 3.12 and [Poetry](https://python-poetry.org/docs/#installation), then run:

```bash
git clone https://github.com/dhruvkshah75/modern-software-dev-assignments.git
cd modern-software-dev-assignments

python3.12 -m venv .venv
source .venv/bin/activate
poetry install --no-interaction
```

For an existing checkout with its environment already created:

```bash
source .venv/bin/activate
poetry install --no-interaction
```

Verify the environment:

```bash
python --version
poetry check --lock
python -c "import fastmcp; print(fastmcp.__version__)"
```

The original course setup uses a Conda environment. Conda with Python 3.12 can also be used in place of `.venv`; run the same Poetry dependency installation inside the activated environment.

Week 2's GitHub App registration, OAuth configuration, and server commands will be documented as those phases are implemented. Keep credentials and cached tokens out of version control.

## Learning approach

For each phase, I aim to understand the concept, attempt a small exercise, build the feature, and verify it before moving on. Writeups will record design decisions, failures, observations of agent behavior, and what I learned from them.

## Credits

The original assignment materials come from [mihail911/modern-software-dev-assignments](https://github.com/mihail911/modern-software-dev-assignments) and the [CS146S course](https://themodernsoftware.dev). My additions are the project plans, implementations, experiments, and learning notes developed in this fork.
