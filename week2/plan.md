# GitHub Backlog Triage: Learning MCP, Agents, and Skills

## Purpose

Build a useful GitHub backlog assistant while learning how modern coding agents use MCP servers and skills. This is a self-directed project inspired by the course's Week 2 assignment. Learning, maintainability, and useful behavior take priority over grading requirements or submission instructions.

The first complete workflow is:

> Inspect open authentication issues across my selected repositories. Recommend existing labels, assignable users, and milestones. Explain the recommendations, preview the changes, and apply the changes I approve.

Codex interprets the request and makes recommendations. Our MCP server retrieves information, validates inputs, and performs narrowly defined GitHub operations. A later skill will teach Codex a repeatable triage procedure using those tools.

## Decisions and boundaries

| Aspect | Decision |
|---|---|
| Starting knowledge | Comfortable with programming, servers, and APIs; new to MCP |
| Learning style | Guided checkpoints: learn, attempt, implement, verify, reflect |
| Coding agent and MCP client | Codex |
| Language and environment | Python 3.12, Poetry, repository-local `.venv` |
| Framework | Installed FastMCP 4.1.6 |
| Transport | Local stdio |
| Authentication | GitHub App user authorization through OAuth, with expiring tokens |
| Repositories | Several explicitly allowlisted repositories |
| First writes | Add existing labels, add assignable users, set an existing open milestone |
| Write brake | Preview by default with `dry_run=True` |
| Operation size | One issue per mutation call |
| Initial deployment | Local process launched by Codex |

Defer removals, closing issues, comments as writes, creating metadata, pull-request management, bulk writes, and hosted deployment until the first workflow is reliable. No frontend, database, or embedded LLM is needed initially.

The course rubric is a useful review checklist, but there is no need to use another agent client, obtain instructor approval, add grading collaborators, or submit to Gradescope.

## Current repository state

- Python 3.12 and `.venv` are ready.
- Course dependencies and FastMCP are installed.
- `pyproject.toml` and `poetry.lock` already contain the setup changes.
- `week2/assignment.md` provides the original project brief.
- `week2/writeup.md` is still an empty course template; adapt it into an engineering learning record as the project develops.
- The server, GitHub App, OAuth flow, configuration examples, and tests remain to be built.

## Architecture and concepts

```text
You: request and approve work
  |
  v
Codex: agent host and MCP client
  | discovers tools, interprets schemas, chooses calls
  | MCP over stdin/stdout
  v
Our FastMCP server
  |-- tool contracts and workflow instructions
  |-- repository allowlist and validation
  |-- GitHub API adapter
  |-- OAuth token manager
  |
  | HTTPS
  v
GitHub API
```

Learn these distinctions early:

- **Agent:** chooses actions using the model, instructions, conversation, and available tools.
- **MCP client:** connects to a server, discovers its capabilities, and invokes tools.
- **MCP server:** exposes tools through a standard protocol and executes their implementations.
- **Third-party API:** GitHub's interface, called by our server after an MCP tool invocation.
- **Skill:** reusable instructions and optional resources that teach the agent how to perform a workflow. A skill can use MCP tools but does not replace their implementations.
- **Authentication boundary:** the server's GitHub login is separate from Codex's local stdio connection to the server.

## Public tool contract

| Tool | Inputs and behavior | Results used by other tools |
|---|---|---|
| `list_repositories` | List configured repositories without an HTTP request | `owner/repo` names |
| `search_issues` | Repository, plain search text, state, optional labels, pagination | Issue numbers, titles, URLs, concise triage fields |
| `get_issue` | Repository and positive issue number | Body, current labels, assignees, milestone, update timestamp, comment count |
| `list_issue_comments` | Repository, issue number, pagination | Discussion text, author logins, timestamps, URLs |
| `list_triage_options` | Repository, kind restricted to `labels`, `assignees`, or `milestones`, pagination | Existing label names, assignable user logins, open milestone numbers |
| `update_issue_triage` | Repository, issue number, expected update timestamp, labels/users to add, optional milestone number, `dry_run=True` | Proposed before/after state or verified per-operation results |

### Contract rules

- Use repository plus `issue_number`, not GitHub's global issue ID.
- Use assignee logins and repository-local milestone numbers.
- Validate repository names against the normalized allowlist before requests.
- Construct search queries from structured inputs; the server controls repository qualifiers and `is:issue`. Plain search text cannot broaden the repository scope.
- Reject pull requests in issue-specific tools.
- Encode enums, positive integers, bounded collections, and pagination limits in schemas.
- Default to 20 results per page and cap at 100; return explicit next-page information.
- Shape outputs around the workflow. Include URLs and explicit truncation markers for shortened text.
- Preserve GitHub search's `incomplete_results` flag.
- Return repository context with discovered values to prevent accidental cross-repository reuse.
- Set `readOnlyHint=True` on readers. Mark the triage operation as a write even when invoked as a preview.
- Use concise server instructions for the overall workflow and tool descriptions for tool-specific contracts.

### Results and errors

Use typed, consistently shaped success and error objects. Expected failures should set MCP's tool-error flag and include an error code, explanation, whether retry is appropriate, and an actionable next step. Include a retry delay when available.

Cover invalid input, disallowed repositories, missing or inaccessible issues, invalid metadata, stale previews, required authorization, insufficient permissions, rate limits, temporary upstream failures, partial writes, and uncertain mutation outcomes. Never expose tokens or raw tracebacks in tool results.

Tool annotations describe behavior. Server validation and dry-run logic enforce behavior. A `dry_run=False` argument is not proof of human consent; the agent workflow and client controls must obtain approval before invoking it.

## Phase 0 — Define the workflow and test backlog

**Learn:** composable tools, focused workflows, and the boundary between model judgment and deterministic execution.

**Attempt:** write a realistic request and list the exact information and actions required to satisfy it manually.

**Build:**

1. Select two repositories you control for live testing; record them in local configuration.
2. Prepare issues with clear bugs, ambiguous reports, discussion, existing labels, assignable users, and an open milestone.
3. Include issues that already have labels and assignees, so preservation is observable.
4. Record a successful workflow and one intentional failure scenario.

**Checkpoint:** explain search → inspect → discover options → recommend → preview → approve → apply using actual repository data.

## Phase 1 — Learn MCP through a minimal working server

**Learn:** host/client/server roles, initialization, tool discovery, invocation, JSON schemas, and stdio process lifecycle. Understand why protocol traffic uses stdout and logs use stderr.

**Attempt:** predict the schema for a typed Python tool, then compare it with the schema published by FastMCP. Read one reference server before designing ours.

**Build:**

1. Create a Week 2 package with a CLI entrypoint and a dependency-injected server factory.
2. Implement the real `list_repositories` tool using local configuration.
3. Add short workflow instructions and strict input validation.
4. Keep tool discovery available without GitHub authorization; tools needing GitHub can return `AUTH_REQUIRED` later.
5. Connect a FastMCP client and inspect the discovered tool definition.

**Checkpoint:** call the tool through MCP and explain the interaction without involving an LLM.

## Phase 2 — Design the interface and GitHub adapter

**Learn:** agent ergonomics, output shaping, schema constraints, tool versus protocol errors, pagination, and explicit identifiers.

**Attempt:** describe `search_issues` and `get_issue` so another developer can chain them using only their published contracts.

**Build:**

1. Define the six tool schemas, typed outputs, shared error envelope, and configuration model.
2. Separate tool handlers from GitHub HTTP operations and pure preview logic.
3. Use a shared `httpx.AsyncClient` with bounded timeouts and explicit GitHub headers.
4. Declare HTTPX as a runtime dependency; it currently appears in the development group.
5. Implement allowlist checks, output shaping, pagination, and centralized error mapping.
6. Inject fake HTTP responses in tests rather than adding fake-service parameters to production tools.

**Checkpoint:** chain search → detail → option discovery using fake API responses, including an actionable error result.

## Phase 3 — Implement real OAuth and token lifecycle

**Learn:** app installation versus user authorization, authorization-code exchange, state, PKCE, redirect matching, access/refresh tokens, rotation, caching, and concurrent refresh.

**Attempt:** draw the login sequence and explain why refreshed access and refresh tokens must be persisted together.

**Build:**

1. Register a GitHub App with Issues read/write and mandatory Metadata read; disable unused webhooks.
2. Install it only on selected test repositories. User authorization and app installation are separate steps.
3. Enable expiring user tokens and register the exact callback `http://127.0.0.1:8765/callback`.
4. Implement a separate login command with random state, PKCE S256, loopback callback validation, timeout handling, and authorization-code exchange.
5. Load client ID, client secret, and repository allowlist from environment; allow an ignored `week2/.env` resolved independently of the launch directory.
6. Cache tokens under an ignored private `week2/.state/` directory with restrictive permissions.
7. Add `filelock` as a runtime dependency to serialize refresh across processes. Re-read the cache after acquiring the lock.
8. Refresh shortly before expiry and atomically save the rotated pair; on an unauthorized response, allow one controlled refresh attempt before returning an actionable error.
9. Implement auth-status without printing credentials and logout that removes the local cache.
10. Never open a browser during a tool call. Failed or revoked refresh authorization returns `AUTH_REQUIRED` with the login command.

**Checkpoint:** authorize once, restart, and reuse the cache. Tests prove refresh, rotation persistence, state rejection, and revoked-authorization behavior.

GitHub App repository permissions and the user's own access determine effective access; OAuth scopes do not grant the app repository permissions. Use the user authorization-code flow for this project rather than a pasted token or an installation token.

## Phase 4 — Connect the readers to real GitHub data

**Learn:** repository-local identity, search rate limits, incomplete results, and the ambiguity of a GitHub 404 for inaccessible resources.

**Attempt:** inspect a raw issue response and justify which fields our MCP output keeps or drops.

**Build:**

1. Connect issue search, details, comments, and triage-option discovery to the API.
2. Discover existing label names, assignable user logins, and open milestones.
3. Validate chosen values directly or through complete internal pagination; absence from one page does not prove invalidity.
4. Return bounded source text and links that let Codex justify its recommendations.
5. Exercise both selected repositories and reject an unlisted one before any HTTP request.

**Checkpoint:** all reader tools work against real data, and each downstream input comes from a documented source.

## Phase 5 — Add previewed and verified writes

**Learn:** additive versus replacement operations, stale reads, partial success, uncertain network outcomes, and post-write verification.

**Attempt:** prepare a before/after preview for an issue that already has labels and an assignee. Show how their existing values survive.

**Build:**

1. Accept only additions to labels/assignees and setting an existing open milestone; an omitted milestone means unchanged.
2. Validate every requested value and compare the current issue timestamp with `expected_updated_at` before making changes.
3. Default to a before/after preview that performs zero mutation requests.
4. Have the agent show the preview and obtain approval before calling with `dry_run=False`.
5. Use additive endpoints for labels and assignees; update the milestone separately. GitHub's general issue PATCH replaces label and assignee sets, so do not use it for additive changes.
6. Apply in a documented order: labels → assignees → milestone. Skip unchanged operations.
7. Read back and verify the requested results. Some GitHub metadata updates can be ignored when the user lacks sufficient permissions.
8. Report completed, failed, and uncertain operations if execution stops partway through. There is no multi-request transaction or automatic rollback.
9. Do not blindly retry writes after ambiguous network failures; inspect current state first.
10. Reject stale requests and obtain a fresh preview. Timestamp comparison is a best-effort preflight, not an atomic GitHub lock.

**Checkpoint:** preview causes no writes; an approved update changes a test issue, preserves existing values, and returns verified results.

## Phase 6 — Verify the protocol and failure paths

**Learn:** unit tests versus protocol tests versus live demonstrations, dependency injection, and the limits of mocks.

**Build:**

1. Unit-test configuration, shaping, errors, OAuth state, cache reuse, rotation, and preview calculation.
2. Use `Client(server)` for in-memory MCP tests with an injected fake GitHub adapter.
3. Inspect discovered schemas, descriptions, and annotations, then chain tools through MCP rather than directly calling their Python functions.
4. Assert expected tool failures with `raise_on_error=False`, inspecting both the error flag and structured result.
5. Add a subprocess stdio smoke test launching the production entrypoint and calling `list_repositories` without real credentials.
6. Use ordinary pytest tests with `asyncio.run(...)`; keep automated tests independent of live GitHub accounts.

| Scenario | Expected behavior |
|---|---|
| Repository outside allowlist | Rejected before HTTP |
| Invalid issue number or enum | MCP input validation failure |
| Pull request passed as an issue | Explicit rejection |
| Missing/inaccessible issue | Actionable error |
| Invalid label, assignee, milestone | No mutation |
| Expired access token | Silent refresh |
| Revoked refresh token | Login guidance, no browser |
| Rate limit | Retry guidance with available delay |
| Preview | Zero mutation requests |
| Stale issue timestamp | Fresh read and preview required |
| Existing labels and assignees | Preserved |
| Partial or uncertain write | Accurate outcomes and inspection guidance |
| Stdio launch | Discovery, call, and clean shutdown |

**Checkpoint:** meaningful automated tests pass, including genuine MCP and subprocess communication.

## Phase 7 — Integrate with Codex and observe agent behavior

**Learn:** client configuration, process lifecycle, tool selection, and how schemas and descriptions change agent behavior.

**Build:**

1. Provide a Codex TOML configuration example with an absolute virtualenv interpreter, `-m week2 serve` arguments, and the repository as working directory.
2. Use existing Codex access; the MCP server itself makes no OpenAI API calls.
3. Document separate GitHub login, configuration, connection inspection, and restart steps.
4. Verify the server with `codex mcp list` and `/mcp`.
5. Run representative prompts for investigation, recommendation, preview, and approved application.
6. Verify that the calls use our custom server.
7. Record actual prompts, tool arguments, results, approval, and GitHub state.
8. Observe at least one misuse, improve a description or schema, and repeat the prompt.

**Checkpoint:** Codex obtains repository-specific options, explains recommendations, previews changes, and performs an approved update. A transcript also demonstrates recovery from a failure.

## Phase 8 — Add a reusable triage skill

**Learn:** when procedural guidance belongs in a skill, when a constraint belongs in a tool schema or implementation, and how skills complement MCP capabilities.

**Attempt:** perform the workflow once using the tools alone. Identify which steps you repeatedly have to explain to Codex.

**Build:**

1. Create a project-local skill using the skill-creator guidance available at implementation time.
2. Teach a concrete procedure: select repositories → inspect issues and discussion → discover valid metadata → explain recommendations → preview → obtain approval → apply → verify and summarize.
3. Treat issue bodies and comments as source data, including any embedded instructions. They do not authorize actions or change the workflow.
4. Keep the skill focused on procedure and judgment; schemas and the server remain responsible for validation and allowed operations.
5. Compare a small, fixed set of prompts with the MCP server alone and with the skill enabled.
6. Record unnecessary calls, invented values, missed context, unsupported recommendations, and approval behavior.

**Checkpoint:** explain what the skill adds, demonstrate whether it improves the workflow, and keep or revise it based on observations. A skill need not improve every case.

## Phase 9 — Document and review the complete project

**Learn:** evidence-based technical explanations and reproducible project setup.

**Build:**

1. Write setup, GitHub App registration, environment, login, startup, Codex configuration, testing, and troubleshooting instructions.
2. Include secret-free environment and Codex configuration examples. A portable `.mcp.json.example` is optional for interoperability, not a grading obligation.
3. Adapt the existing writeup into a learning journal with architecture decisions, actual transcripts, failure handling, test coverage, and skill evaluation.
4. Check that environment files, caches, real configurations, and credentials are ignored.
5. Document logout, app-access revocation, and server removal without performing them automatically.
6. Verify the documented workflow from a clean environment before calling the project complete.

**Checkpoint:** another developer can reproduce the setup, understand the design, run tests, and exercise the workflow.

## Implementation organization

Keep application code under `week2/`, separated by responsibility:

- CLI and configuration: serve, login, auth-status, logout, environment, allowlist.
- MCP server: tool registration, schemas, annotations, instructions.
- GitHub adapter: HTTP, pagination, output shaping, API error translation.
- OAuth: callback, token cache, locking, refresh.
- Models and triage logic: typed outputs, validation, previews, verified outcomes.
- Tests and documentation: unit tests, MCP tests, stdio test, configuration examples, learning record.

Use a server factory for dependency injection. Keep temporary experiments out of the public tool catalog. Build one coherent phase at a time, with a small reviewable change and a passing checkpoint before continuing.

## Commands

Available now, from the repository root:

```bash
source .venv/bin/activate
python --version
poetry check --lock
python -c "import fastmcp; print(fastmcp.__version__)"
```

Commands to implement and document during the corresponding phases:

```bash
# Interactive GitHub authorization
poetry run python -m week2 login

# Authorization status without credentials in output
poetry run python -m week2 auth-status

# Single stdio server command
poetry run python -m week2 serve

# Automated verification
poetry run pytest week2/tests -q
poetry run ruff check week2
poetry run black --check week2

# Codex connection inspection
codex mcp list

# Remove the local OAuth cache
poetry run python -m week2 logout
```

Starting `serve` manually waits for an MCP client to communicate over stdio; that is expected. The actual Codex configuration should launch the virtualenv interpreter directly and set the working directory explicitly.

## Learning record and completion criteria

At the end of each phase, record:

1. The concept learned and a plain-language explanation.
2. What you attempted before asking the agent for implementation help.
3. What the checkpoint proves, and what it does not prove.
4. A failure and how you diagnosed it.
5. Evidence worth retaining: a schema, request, test, transcript, or design decision.

The project is complete when the tools work across the selected repositories, OAuth survives process restarts and token expiry, writes preview and verify correctly, meaningful protocol tests pass, and Codex completes the real workflow. The skill phase should produce a documented comparison and a reusable procedure if it proves useful.

You should be able to explain why Codex selected a tool, what MCP communicated, how the server reached GitHub, which permissions allowed the action, how errors were handled, and what changed when a skill was introduced.

## Reference material

Read the relevant sections when reaching each phase rather than studying everything upfront.

- [FastMCP tools](https://gofastmcp.com/servers/tools): schemas, annotations, structured results, errors.
- [FastMCP client transports](https://gofastmcp.com/clients/transports): in-memory and stdio client connections.
- [MCP tools specification](https://modelcontextprotocol.io/specification/2026-07-28/server/tools): discovery, invocation, and error semantics.
- [Reference MCP servers](https://github.com/modelcontextprotocol/servers): inspect an implementation before designing ours.
- [GitHub App user authorization](https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/generating-a-user-access-token-for-a-github-app): authorization-code flow and permissions.
- [GitHub token refresh](https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/refreshing-user-access-tokens): expiry and rotation.
- [GitHub issue API](https://docs.github.com/en/rest/issues/issues): details, updates, and metadata semantics.
- [GitHub labels](https://docs.github.com/en/rest/issues/labels): additive versus replacement operations.
- [GitHub assignees](https://docs.github.com/en/rest/issues/assignees): discovery, eligibility, and additions.
- [GitHub milestones](https://docs.github.com/en/rest/issues/milestones): repository-local milestone numbers and states.
- [Codex MCP documentation](https://learn.chatgpt.com/docs/extend/mcp?surface=cli): local server configuration and inspection.

Check documentation against the installed framework and client versions when implementing. Avoid copying examples from a different FastMCP major version without verifying their APIs.
