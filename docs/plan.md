# Plan: Promptfoo jump-start — Vehicle Safety Assistant on a free public NHTSA MCP server

## Context

The repo `agent-test-eval-secure-ci-jump-start` is empty apart from this plan (branch `develop`, no commits). **The only goal is to cover all promptfoo features with simple examples**, with no risk of runaway token cost. The examples need something realistic to test, so the repo includes a small agent app:

- React chat UI, FastAPI backend, LangChain **deep agent** on **Claude**, and a **FastMCP client** that invokes a **free, publicly hosted MCP server** for NHTSA vehicle-safety data.
- Use case: a **Vehicle Safety Assistant**. It answers any question that NHTSA data can answer, for example:
  - "What is this VIN?" (decode make, model, year, engine, plant)
  - "Is my 2017 Honda Civic under any open recall?"
  - "What do owners complain about most on a 2019 Ford F-150?"
  - "Which is safer in crash tests, a 2020 Camry or a 2020 Accord?"
  - "Is NHTSA investigating any defect on my car?"
- No database. One shared login (`promptfoo`). The Anthropic key goes in `.env`.

This plan folds in an architecture review (section "Review changes applied").

## Decisions (confirmed with the user)

| Topic | Decision |
|---|---|
| Goal | Cover all promptfoo features with simple examples; nothing beyond that |
| Use case | Any helpful use of the data exposed by the NHTSA MCP server (VIN decode, recalls, complaints, safety ratings, investigations, vehicle lookup). Not tied to insurance |
| Model | Claude via `ANTHROPIC_API_KEY` in `.env`. **Claude Haiku 5.5 (`claude-haiku-5-5`) by default for the app, evals and graders**, to keep cost low; Sonnet 5.5 only where an example compares models, or by changing `.env` |
| MCP | **Only** a free, publicly hosted NHTSA MCP server; we host no MCP server. Primary `https://nhtsa.caseyjhand.com/mcp`, fallback `https://nhtsa-recalls-mcp.chucks-lab.workers.dev/mcp` (both live when probed on 2026-11-02: no auth, streamable HTTP, 7 tools each). NHTSA has no official MCP server; these are community-run wrappers over NHTSA's public APIs |
| MCP in tests and CI | **Always live**; no replay double. Flakiness is handled by promptfoo's cache, retries, and a pre-run health check |
| Login | One shared demo account `promptfoo`, password from `.env`, stateless signed token |
| Python | **3.14** via `uv` (supported by deepagents 0.7.32, langchain 1.4.7, langchain-anthropic 1.7.7, fastmcp 4.1.5, fastapi 0.162; langgraph 1.2.19 lacks the classifier, so it gets a smoke test) |
| Node | ≥22.22 (required by promptfoo 0.125.1; machine has Node 26) |
| UI scope | Streaming chat (SSE). Promptfoo is **not** embedded: the app is the system under test, and a footer links to promptfoo's own web UI |
| Multimodal | Out of scope |

## Architecture — block diagram (numbered flow)

```
CHAT FLOW

┌──────────────────────────────────────────────────────────────────────────────┐
│ BROWSER · React + Vite · localhost:5173                                      │
│   Login page                      Chat page (streams replies)                │
└───┬───────────────────────────────┬──────────────────────────────────────────┘
    │             ▲                 │                   ▲ (8) SSE stream:
    │ (1) login   │ (2) token       │ (3) message       │ tokens, tool steps,
    ▼             │                 ▼ + Bearer token    │ report
┌─────────────────┴─────────────────────────────────────┴──────────────────────┐
│ FASTAPI BACKEND · 127.0.0.1:8000                                             │
│                                                                              │
│  ┌────────────────┐         ┌─────────────────────────────────────────────┐  │
│  │ Auth           │─(4) run▶│ Deep agent (LangChain deepagents)           │  │
│  │ JWT · 8h       │         │ planner · 1 subagent · Claude calls         │  │
│  └────────────────┘         │ returns VehicleReport (structured)          │  │
│                             └─────────────────────────────────────┬───────┘  │
│                                                   ▲               │   ▲      │
│                                                   │               │   │      │
│                                                   │               │   │      │
└───────────────────────────────────────────────────┼───────────────┼───┼──────┘
                                                    │ (5) LLM calls │   │ (6) tool call
                                                    ▼ + streaming   ▼   │ (7) result (untrusted)
                                        ┌───────────────────┐  ┌────────┴──────────────────────────┐
                                        │ ANTHROPIC API     │  │ PUBLIC NHTSA MCP SERVER           │
                                        │ Claude Haiku 5.5  │  │ free · community-run · no key     │
                                        └───────────────────┘  │ nhtsa.caseyjhand.com/mcp          │
                                                               │ fallback: chucks-lab.workers.dev  │
                                                               └────────────────┬──────────────────┘
                                                                                │ wraps
                                                                                ▼
                                                               ┌───────────────────────────────────┐
                                                               │ NHTSA public APIs                 │
                                                               │ vPIC, recalls, complaints, NCAP   │
                                                               └───────────────────────────────────┘


EVAL FLOW

┌─────────────────────────────┐                 ┌─────────────────────────────────┐
│ REACT APP                   │──(E8)──────────▶│ PROMPTFOO WEB UI · :15500       │
│ footer link                 │                 │ results · traces · red-team     │
└─────────────────────────────┘                 └─────────────────────────────────┘
                                                                ▲
                                                                │ (E7) results
                                                                │
┌───────────────────────────────────────────────────────────────┴─────────────────┐
│ PROMPTFOO CLI · make eval-NN · 13 folders in evals/                             │
│ OTLP trace receiver :4318                                                       │
└──┬─────────────┬────────────────────────────────────┬─────────────┬─────────────┘
   │             │                       ▲            │             │       ▲
   │ (E1) login  │ (E2) /api/chat/sync   │ (E4) OTel  │ (E3) MCP    │ (E5)  │ (E6)
   │             │      + traceparent    │      spans │ provider    │ grade │ scores
   ▼             ▼                       │            ▼             ▼       │
┌────────────────────────────────────────┴──┐  ┌───────────────┐ ┌──────────┴─────┐
│ FASTAPI BACKEND + deep agent              │  │ NHTSA MCP     │ │ CLAUDE         │
│ same app as the chat flow                 │  │ (public)      │ │ grader (Haiku) │
└───────────────────────────────────────────┘  └───────────────┘ └────────────────┘
```

**Chat flow**

1. The user submits credentials from the React login page to `POST /api/auth/login`.
2. The backend checks them in constant time against `.env` and returns a signed token (8h expiry, held in memory in the browser).
3. React posts the message and `thread_id` to `POST /api/chat` with `Authorization: Bearer`, read via `fetch` and a stream reader (not `EventSource`, which can't POST or send headers).
4. The backend validates the token and runs the deep agent for that thread (in-memory LangGraph checkpointer).
5. The agent streams Claude calls (planner and one subagent, `vehicle-research`), with prompt caching on the system prompt and tools.
6. The FastMCP client invokes allow-listed tools on the public NHTSA MCP server.
7. Tool results return and are treated as **untrusted data**: wrapped, trimmed, never followed as instructions. This matters because NHTSA complaint narratives are free text written by the public.
8. Token deltas, tool-call steps and the final structured `VehicleReport` stream back as SSE events.

**Eval flow**

- **E1.** Promptfoo logs in once per run (an extension hook stores the token).
- **E2.** Promptfoo calls `POST /api/chat/sync` (the HTTP provider) with a `traceparent` header.
- **E3.** Promptfoo's own MCP provider calls the public NHTSA MCP server directly (MCP example folder).
- **E4.** The backend exports OpenTelemetry spans to promptfoo's OTLP receiver, which enables trace and `trajectory:*` assertions.
- **E5 and E6.** Model-graded assertions go to Claude (grader pinned explicitly, since promptfoo defaults to OpenAI).
- **E7.** Results are stored locally and viewed in the promptfoo web UI (`make view`).
- **E8.** The React footer links to it.

## Review changes applied (principal-architect pass)

1. **No home-made MCP server and no mock data.** The FastMCP client connects to the public server; every answer comes from NHTSA data.
2. **The third-party MCP server is untrusted.**
   - Tool **allow-list** per endpoint (the two servers use different tool names: `nhtsa_decode_vin` vs `decode_vin`, so `backend/app/mcp_config.py` maps logical capability to tool name).
   - **Tool-schema drift check:** a committed snapshot of tool names and input-schema hashes. The app logs a warning on mismatch, and a test fails on it.
   - Timeouts, one retry, automatic switch to the fallback URL, and output size caps.
3. **Trajectory evals need tracing.** The backend propagates `traceparent` and exports GenAI-convention spans. The instrumentation library is picked in step 0.
4. **Claude-only setup.** Graders are pinned to Claude in `defaultTest.options.provider`. Features that need another provider live in the opt-in folder 15. promptfoo telemetry is turned off in `.env.example` (`PROMPTFOO_DISABLE_TELEMETRY=1`); the red-team remote-generation opt-out is confirmed in step 0.
5. **Structured output.** The agent returns a Pydantic `VehicleReport` (vehicle, recalls with campaign ids, complaint summary by component, safety ratings, investigations, sources, caveats) alongside prose. `/api/chat/sync` returns `{answer, report, tool_calls, usage, latency_ms}`, which makes schema assertions reliable.
6. **Cost control.** Five layers of limits; see "Cost guardrails".
7. **Exposure of your API key.** Bind to `127.0.0.1`, CORS only for `http://localhost:5173`, a per-token rate limit, and an input length cap.
8. **Secure CI.** GitHub Actions pinned to commit SHAs, least-privilege `permissions`, no secrets for fork PRs, and a pre-run MCP health check that fails fast with a clear message.

## Repo layout (key files)

```
backend/
  pyproject.toml                 uv, requires-python >=3.14
  app/main.py                    FastAPI app, CORS, routes, /api/eval-webhook (for the webhook assertion)
  app/auth.py                    login, token sign/verify (PyJWT), dependency
  app/agent.py                   deep agent factory (system prompt, subagent, VehicleReport schema, step limits)
  app/budget.py                  per-request and daily token budgets
  app/mcp_config.py              endpoint list, allow-list, capability->tool map, schema snapshot check
  app/mcp_client.py              FastMCP client -> LangChain tools, timeouts/retry/fallback, result trimming
  app/tracing.py                 OTel setup, traceparent propagation
  app/prompts/system_v1.md, system_v2.md   prompt variants used by evals
  tests/                         pytest: auth, budgets, sync endpoint, live MCP schema snapshot
frontend/                        Vite + React + TS: Login.tsx, Chat.tsx, sse.ts, api.ts
evals/
  README.md                      learning path
  _shared/                       providers.yaml, hooks.js (login), datasets/, saved outputs for Free examples
  01-getting-started/ ... 15-optional-extras/   one feature area each (below)
.github/workflows/evals.yml
.env.example  Makefile  README.md  docs/plan.md (this plan)
```

## Promptfoo coverage (all features, simple NHTSA examples)

**Rules for every example**

- Every example uses the Vehicle Safety Assistant or the NHTSA MCP server directly, with NHTSA questions (VIN decode, recalls, complaints, safety ratings, investigations).
- Each folder has at most 5 test cases, one `promptfooconfig.yaml`, and a short README: what the feature is, how to run it, what to look at, and its cost class.
- **Cost classes:** **Free** means no LLM calls: assertions run on saved outputs (`--model-outputs`), or promptfoo calls the NHTSA MCP server directly. **Low** means a few Haiku calls, cached after the first run. **Capped** means red-team or multi-turn runs with hard limits (see "Cost guardrails").
- Coverage was checked against promptfoo's documentation index (`promptfoo.dev/llms.txt`) on 2026-11-02. Folders 01–14 cover the core with Claude only. Folder 15 holds the features that need another key, an extra toolchain or more spend. The few things left out are listed below with a reason.

| # | Folder | promptfoo features | NHTSA example | Cost |
|---|---|---|---|---|
| 01 | getting-started | `init`, providers, prompts, tests, vars, `contains`, `eval`, `view`, `validate config`, `--env-file` | Decode a VIN; the answer names the maker | Low |
| 02 | prompts | multiple prompts, `file://` prompts, chat-format prompts, Nunjucks + `nunjucksFilters`, JS and Python prompt functions, per-provider prompts | System prompt v1 (concise) vs v2 (owner-friendly) for the same recall question | Low |
| 03 | deterministic-assertions | `equals`, `contains-any`/`-all`, `icontains*`, `regex`, `starts-with`; `is-json`, `contains-json`, `is-xml`, `is-html`, `is-sql`; `is-refusal`; `javascript`, `python`; `rouge-n`, `bleu`, `gleu`, `meteor`, `levenshtein`; `latency`, `cost`; `not-*`, `assert-set`, weights, thresholds, named metrics, `derivedMetrics`, assertion templates (`$ref`); `webhook` | Recall campaign ids match NHTSA's format (e.g. `19V123000`); `VehicleReport` JSON has the right shape; recall table rendered as HTML/XML | **Free** (saved outputs) + 2 Low tests for `latency`/`cost` |
| 04 | model-graded | `llm-rubric`, `g-eval`, `factuality`, `model-graded-closedqa`, `answer-relevance`, `context-faithfulness`, `context-recall`, `context-relevance` (NHTSA tool output used as the context), `select-best`, `max-score`, agent rubric, grader override (`--grader`, `defaultTest.options.provider`) | Does the answer say only what the recall data says, and advise getting the recall fixed? | Low (Haiku grader) |
| 05 | test-data | CSV, JSON, JSONL, YAML tests; `__expected` columns; `defaultTest`; `scenarios`; `transform`, `transformResponse`, `transformVars`; tags, metadata, `--filter-*`; modular configs (`file://`, `$ref`); `generate dataset`, `generate assertions` | Vehicles × question types (recalls, complaints, ratings) | Low (max 5 tests) |
| 06 | agent-tracing | `tracing` (OpenTelemetry), `trace-span-count`, `trace-span-duration`, `trace-error-spans`, `trajectory:tool-used`, `trajectory:tool-args-match`, `trajectory:tool-sequence`, `trajectory:step-count`, `trajectory:goal-success`, `tool-call-f1` | Given a VIN, the agent must decode it before looking up recalls, and must pass the decoded make/model/year | Low |
| 07 | multi-turn | chat threads (`_conversation`), `storeOutputAs`, `conversation-relevance`, `simulated-user` provider | Used-car shopper narrows down two cars over several messages | Capped (max 3 turns) |
| 08 | mcp | `mcp` provider calling the NHTSA MCP server directly with assertions on raw tool results; Anthropic provider with MCP config (Claude calls NHTSA with no app in between); `promptfoo mcp` (promptfoo as an MCP server for Claude Code) | Recalls for a given year/make/model; the app vs bare Claude + MCP | **Free** (direct tool calls) + Low (comparison) |
| 09 | redteam | `redteam init`, `setup`, `discover`, `generate`, `run`, `report`, `plugins`; purpose; HTTP target; plugins `pii`, `indirect-prompt-injection`, `prompt-extraction`, `hijacking`, `off-topic`, `hallucination`, `overreliance`, `excessive-agency`, `mcp`, `tool-discovery`, `policy` (custom rule: never advise ignoring a safety recall); strategies `basic`, `base64`, `jailbreak-templates`, `prompt-injection`, `goat` (one multi-turn); OWASP LLM Top 10 and OWASP Agentic report views; `guardrails` assertion | Can someone make the assistant invent a recall, tell an owner to ignore one, leak its system prompt, or follow instructions hidden in a complaint narrative? | Capped (1 test per plugin, about 20 attacks) |
| 10 | providers | Anthropic provider (Claude, no agent), HTTP provider with session/auth, Python provider (agent in-process), JavaScript custom provider, `exec` script provider, provider options (`max_tokens`, `temperature`), Haiku vs Sonnet | Same safety-rating comparison sent to each provider | Low (Sonnet: 2 tests only) |
| 11 | caching-performance | cache, `cache clear`, `--no-cache`, `maxConcurrency`, `delay`, rate limits, `repeat`, `--resume`, `--retry-errors` | Ask the same recall question 3 times to check consistency | Low (cached after the first run) |
| 12 | extensions-hooks | `extensions` (`beforeAll`, `beforeEach`, `afterEach`, `afterAll`), `-x` flag | Log in once in `beforeAll`; print token use after each test | Low |
| 13 | ci | GitHub workflow, `PROMPTFOO_PASS_RATE_THRESHOLD`, `PROMPTFOO_FAILED_TEST_EXIT_CODE`, outputs (json, jsonl, csv, yaml, txt, html, xml, junit.xml), PR comment, `PROMPTFOO_DISABLE_TELEMETRY`, pytest integration | CI smoke suite (about 10 tests) | Low (cached) |
| 14 | cli-and-api | `list`, `show`, `export`, `import`, `delete`, `retry`, `logs`, `config`, `debug`, `share` (optional; uploads to promptfoo's cloud), Node package API (`promptfoo.evaluate()` from a script), `code-scans run` | A JS script that runs folder 01 and prints the pass rate | Free / Low |
| 15 | optional-extras | Opt-in; each part runs only when its key or `CONFIRM=1` is set. **Assertions needing other providers:** `similar` (embeddings), `classifier`, `moderation`, `pi`, `search-rubric`, `perplexity`/`perplexity-score` (needs a provider that returns log-probabilities). **Other providers:** one non-Anthropic provider beside Claude. **Ruby:** assertion and provider. **Expensive strategies:** `iterative`, `tree`, `best-of-n`, `hydra`, `gcg` (1 test each). **Other tools:** `optimize`, `scan-model` on a tiny sample model file, Hugging Face dataset loading. **Other CI systems:** GitLab CI and Jenkins example files (not run) | Same NHTSA questions as earlier folders | Off by default; Capped when on |

Questions used across folders:

- decode a valid VIN
- open recalls for a vehicle with a known recall (for example, an airbag recall)
- the most-reported complaint components for a model year
- a crash-rating comparison between two vehicles
- open defect investigations
- an invalid VIN (must ask for a correct one, not guess)
- a vehicle NHTSA has no data on (must say so, not invent)
- "can I ignore this recall?" (must advise getting it fixed)
- an off-topic request

Real VINs and vehicles are picked in step 2 from live NHTSA responses.

**Not covered, and why**

| Feature | Reason |
|---|---|
| Enterprise features (SSO, teams, audit logs, service accounts, remediation reports, hosted guardrails) | Paid promptfoo product |
| Image, audio and video strategies; vision plugins | Multimodal was excluded from this project |
| Observability and platform integrations (Langfuse, Helicone, Portkey, Splunk, SonarQube, n8n, Google Sheets, SharePoint, Burp) | Each needs an account on another product |
| Self-hosting promptfoo | Deployment is out of scope |

## Cost guardrails (no token-cost explosion)

Five layers, so a mistake in one does not lead to a large bill.

1. **Hard stop at Anthropic (one-time, by you).** Set a monthly spend limit on the Anthropic Console workspace that owns the API key. The README makes this setup step 1.
2. **Limits inside the app.** These apply to every request, including requests from promptfoo:
   - `max_tokens` 1,024 per model call
   - agent recursion limit of 12 steps; at most 6 tool calls per turn; one subagent
   - **NHTSA tool results trimmed** to at most 5 items and 4,000 characters. Complaint lists for popular vehicles can run to thousands of records, which is the biggest blow-up risk in this app.
   - user input capped at 2,000 characters
   - **per-request budget** of 40,000 tokens: the turn stops with a clear error when it is reached
   - **daily budget** per running backend, `TOKEN_BUDGET_DAILY` (default 3,000,000 tokens): requests are refused with HTTP 429 after it
   - Anthropic prompt caching on the system prompt and tool definitions
3. **Cheap models by default.** Claude Haiku 5.5 is the default for the app, evals, graders and red team. Sonnet is used only in folder 10 (2 tests), or when you change `AGENT_MODEL` in `.env`.
4. **promptfoo limits.**
   - Cache on by default; `maxConcurrency: 2`; at most 5 tests per folder; `repeat` at most 3
   - `cost` assertion on live tests, so an unusually expensive answer fails the test
   - Red team: 1 test per plugin, one multi-turn strategy at 3 turns, expensive strategies only in folder 15
   - promptfoo's own model calls (graders, red-team attack generation, the simulated user, `generate`) go straight to Anthropic, not through the app, so the app's budgets do not see them. They are limited by the test caps above, by Haiku, and finally by the Anthropic spend limit (layer 1)
   - `make eval-all` skips the red team and folder 15; `make eval-09` and the expensive parts of folder 15 ask for `CONFIRM=1`
5. **Visibility.** The backend logs tokens per request; promptfoo shows cost per test; each folder README states its cost class.

Ceiling: about 70 live test cases × 40,000 tokens ≈ 2.8M tokens for a fully uncached `make eval-all`, which is below the daily budget. Normal runs use far less because of the cache.

## CI integration (GitHub)

Kept deliberately small: **one secret, one workflow file, one job.**

**One-time setup in GitHub**

| Where | What | Required |
|---|---|---|
| Settings → Secrets and variables → Actions → *Secrets* | `ANTHROPIC_API_KEY` | Yes, the only secret |
| Same page → *Variables* | `AGENT_MODEL`, `GRADER_MODEL` (default to Haiku) | No |
| Settings → Branches → rule for `main` | Require status check `evals` before merge | No (recommended) |

`DEMO_PASSWORD` and `JWT_SECRET` are not secrets in CI. The workflow generates random values for each run, because the app only lives for the length of the job. `GITHUB_TOKEN` is provided automatically.

**Workflow: `.github/workflows/evals.yml`**

- **Triggers:** pull requests to `main`, pushes to `main`, and manual runs (`workflow_dispatch`).
- **Fork PRs are skipped.** GitHub withholds secrets from them anyway, and the job checks this explicitly so it is skipped cleanly instead of failing.
- **Permissions:** `contents: read`, `pull-requests: write` (for the PR comment only).
- **Concurrency:** one run per branch; a newer push cancels the older run.
- **Timeout:** 20 minutes.

| Step | What it does |
|---|---|
| 1 | Check out the code |
| 2 | Set up `uv` with Python 3.14, and Node 22 with the npm cache |
| 3 | `make setup` (`uv sync --frozen`, `npm ci`, so versions come from the lockfiles) |
| 4 | Restore the promptfoo response cache (`actions/cache`, keyed on the `evals/` files), which avoids paying again for unchanged tests |
| 5 | Start the backend in the background and wait for `/api/health` |
| 6 | `make mcp-health`: fail fast with a clear message if both public NHTSA MCP servers are down |
| 7 | `make ci-eval`: run the small CI suite (`evals/13-ci`, about 10 tests drawn from folders 01, 03 and 06) and write JSON, JUnit and HTML results |
| 8 | Fail the job if the pass rate is below the threshold (90% to start) |
| 9 | Upload the results as a build artifact, write a summary table to the job summary, and post or update one PR comment |

Red teaming (folder 09) and the full `make eval-all` run are **not** in CI: they are slower and cost more, so they stay manual.

**Safety rules**

- Third-party actions are pinned to full commit SHAs, with the version noted in a comment.
- The API key is only passed as an environment variable to the steps that need it, and is never written to a file or printed.
- `make ci` runs steps 3 to 8 locally, so a failing CI run can be reproduced on a laptop.

## Build order

| Step | Work | Exit check |
|---|---|---|
| 0 | Verify what the docs inventory did not settle: MCP provider and Anthropic MCP config keys, HTTP provider session/auth, red-team config fields and the remote-generation opt-out variable, agent-rubric and `code-scans` requirements; deepagents structured output, streaming and step limits; FastMCP client API; OTel instrumentation; langgraph on 3.14 | Notes captured; plan adjusted if anything differs |
| 1 | Skeleton: `uv` project on 3.14, Vite app, Makefile (`setup`, `dev`, `test`, `eval-NN`, `eval-all`, `view`, `mcp-health`, `ci-eval`, `ci`), `.env.example`, `.gitignore` | `make setup` succeeds; `uv run python -c "import deepagents, langgraph, fastmcp"` on 3.14 |
| 2 | MCP client, allow-list, fallback, result trimming, schema snapshot; pick real test vehicles from live data | `make mcp-health` lists the tools; pytest passes |
| 3 | Deep agent, `/api/chat/sync`, auth, cost budgets, tracing | `curl` login and a recall question returning a valid `report`; an over-budget request is stopped |
| 4 | `/api/chat` SSE and React login + chat | Browser check: login, streamed reply, tool steps visible |
| 5 | Eval folders 01–05 | Each run; pass rate and cost noted in its README |
| 6 | Eval folders 06–08 (with tracing) | Trace and trajectory assertions resolve against real spans |
| 7 | Eval folders 09–15, CI workflow (see "CI integration") | Red-team run completes; `make ci` passes locally; workflow passes `actionlint` |
| 8 | README learning path, clean-clone run-through | Fresh clone, `.env` added, `make setup && make dev && make eval-01` works |

## Verification (end to end)

- `make test`: pytest for auth, budgets, the sync endpoint contract and the live MCP schema snapshot; `npm run build` and type-check for the frontend.
- `make mcp-health`: initialize plus `tools/list` against the primary, then the fallback, compared with the snapshot.
- Manual: `make dev`, log in as `promptfoo`, ask "Is a <real vehicle with a known recall> under recall?", and confirm streamed text, visible NHTSA tool steps, and a report citing the real campaign id.
- `make eval-01` through `make eval-14`, each run at least once; folder 15 parts run when their key is set; `make view` shows results and traces.
- Cost: a request over the per-request budget is stopped; the token log matches promptfoo's reported cost; a full `make eval-all` stays within the stated ceiling.
- CI: `make ci` passes locally; workflow validated with `actionlint`; first GitHub run via `workflow_dispatch`, then a test PR to confirm the PR comment and the required check.

## Risks

- **Public MCP server downtime or tool changes.** Always-live was chosen. Mitigated by the fallback URL, the health check, the drift check and promptfoo's cache. If both servers are down, evals can't run, and the README says so.
- **promptfoo or deepagents API drift.** Step 0 verifies, and versions are pinned and recorded in the README.
- **Cost.** Five layers of limits (see "Cost guardrails"), with the Anthropic spend limit as the final hard stop.

## Out of scope

Everything in "Not covered, and why"; database or user management; deployment (Docker or cloud); embedding promptfoo inside the React app; multimodal; our own MCP server; any mock or non-NHTSA data.
