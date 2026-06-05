# Local-First Planner Agent

## Summary

Build a local-first agent system where a local planning model decides what work should happen, then delegates execution to isolated local worker agents by default. Cloud models such as ChatGPT/OpenAI are optional fallback workers and may only receive redacted, policy-approved context.

The default planning model is `JetBrains/Mellum2-12B-A2.5B-Thinking`, served through an OpenAI-compatible endpoint such as vLLM or SGLang. Mellum2 is used as the planner, not as the default executor: it emits a typed action plan that the Rust orchestrator validates before any tool or worker is called.

The system also maintains a local graph-based source-code index. The graph gives local models relationship-aware repository context, which should improve code generation quality compared with relying only on raw file chunks, grep, or vector retrieval.

Primary goals:

- Reduce reliance on cloud-based models.
- Keep planning and code context lookup local and auditable.
- Delegate execution through explicit, typed contracts.
- Use graph-based code scanning to improve repository-level code understanding.
- Preserve the option to use cloud models when local agents are insufficient and redaction policy allows it.

## Target Architecture

The system has seven core subsystems:

- **Rust orchestrator:** owns task intake, context gathering, planner calls, policy checks, sandbox lifecycle, worker delegation, trace persistence, result aggregation, and final response synthesis.
- **Local planner:** Mellum2 produces typed JSON action plans using a local OpenAI-compatible chat-completions endpoint.
- **Code graph index:** scans source code and directory-level `AGENTS.md` files into a local relationship graph for symbols, modules, calls, imports, type references, tests, and directory summaries.
- **Hybrid retrieval layer:** uses graph lookup to narrow the relevant code neighborhood, then ranks concrete snippets with text, BM25, or vector retrieval.
- **Policy layer:** validates every planner step before execution, blocks unsafe tool requests, enforces local-first routing, and requires redaction before cloud calls.
- **Worker runtime:** launches local agents as isolated subprocesses using JSON-RPC over stdio.
- **Cloud delegation adapter:** calls ChatGPT/OpenAI through the same model-client abstraction used by the local planner, but only after policy approval and redaction.
- **Trace store:** persists project plans, graph queries, retrieval decisions, worker traces, redaction summaries, and final output for auditability.

Default data flow:

1. User submits a task to the orchestrator.
2. Orchestrator reads current `docs/plan/` state and relevant directory-level `AGENTS.md` summaries.
3. Orchestrator consults the code graph to identify likely symbols, modules, files, dependencies, and related tests.
4. Hybrid retrieval ranks concrete snippets inside the graph-selected code neighborhood.
5. Mellum2 receives the task, planning context, graph summaries, and available graph-query tools.
6. Mellum2 returns a typed action plan.
7. Policy layer validates the action plan.
8. Orchestrator delegates approved steps to isolated local workers.
9. Cloud delegation is used only for approved, redacted subtasks.
10. Orchestrator aggregates worker results and synthesizes the final response.
11. Planner prompts, graph queries, plans, decisions, worker traces, redaction summaries, and final output are persisted.

## Phase 1: Architecture Document

Create the planning foundation before implementing code.

Deliverables:

- Create `docs/plan/` if it does not exist.
- Add this phased architecture document.
- Document the high-level architecture, data flow, trust boundaries, and implementation sequence.
- Record initial assumptions about Mellum2 runtime, graph scanning, worker protocol, cloud policy, and trace persistence.

Acceptance criteria:

- The architecture explains how local planning, graph-based code lookup, local workers, cloud fallback, redaction, and trace persistence fit together.
- The implementation phases are ordered so a future engineer can build and test incrementally.
- Major interfaces are named clearly enough to become Rust structs, traits, and modules.

## Phase 2: Core Interfaces

Define the Rust-facing contracts before connecting real models, graph scanners, or workers.

Implementation shape:

- Add a `ModelClient` abstraction for OpenAI-compatible chat-completions calls.
- Use the same abstraction for local Mellum2 and cloud ChatGPT/OpenAI providers.
- Define typed planner-output structs.
- Define graph node, edge, query, and response structs.
- Define worker JSON-RPC request and response structs.
- Define policy decision types.
- Define trace-event structs for planner calls, graph queries, retrieval decisions, policy decisions, worker calls, redaction, and synthesis.

Planner output schema:

```json
{
  "goal": "string",
  "assumptions": ["string"],
  "graph_queries": [
    {
      "id": "string",
      "query_type": "find_symbol | callers | callees | related_tests | import_neighbors | impact_radius | directory_summary",
      "target": "string",
      "impact_radius": 1
    }
  ],
  "context_strategy": "graph_first_hybrid_retrieval",
  "steps": [
    {
      "id": "string",
      "objective": "string",
      "delegate_target": "local_worker | cloud_worker | orchestrator",
      "required_context": ["string"],
      "allowed_tools": ["string"],
      "cloud_allowed": false,
      "redaction_required": false,
      "confidence": 0.0,
      "stop_conditions": ["string"],
      "validation": ["string"]
    }
  ]
}
```

Graph node types:

- `Crate`
- `Module`
- `File`
- `Struct`
- `Enum`
- `Trait`
- `Impl`
- `Function`
- `Method`
- `Test`
- `DirectorySummary`

Graph edge types:

- `Contains`
- `Imports`
- `Calls`
- `ReferencesType`
- `ImplementsTrait`
- `CoveredByTest`
- `OwnsSourceDirectory`
- `SummarizesDirectory`

Graph lookup tools:

- `code_graph.find_symbol`
- `code_graph.callers`
- `code_graph.callees`
- `code_graph.related_tests`
- `code_graph.import_neighbors`
- `code_graph.impact_radius`
- `code_graph.directory_summary`

Worker request schema:

```json
{
  "task_id": "string",
  "objective": "string",
  "context_bundle": {},
  "graph_context": {},
  "sandbox_policy": {},
  "tool_allowlist": ["string"],
  "expected_artifacts": ["string"],
  "budget": {},
  "response_schema": {}
}
```

Worker response schema:

```json
{
  "status": "success | failed | blocked",
  "summary": "string",
  "artifacts": [],
  "tool_trace": [],
  "graph_queries": [],
  "tests_or_checks": [],
  "risks": [],
  "follow_up_needed": []
}
```

Policy decisions:

- `Allowed`
- `Denied`
- `RedactionRequired`
- `ApprovalRequired`
- `RetryWithNarrowerContext`
- `FallbackToTextSearch`

Required error paths:

- Malformed planner JSON.
- Planner step requests a disallowed tool.
- Planner step requests cloud without policy permission.
- Graph index is stale, missing, or incomplete.
- Graph query returns no useful candidates.
- Redaction fails or cannot prove sensitive context was removed.
- Worker subprocess fails, times out, or returns malformed output.
- Cloud provider fails or returns invalid response shape.

Acceptance criteria:

- Interfaces are serializable and deserializable.
- Invalid planner, graph, or worker messages fail closed when safety-sensitive.
- Stale or incomplete graph data fails soft by falling back to text search when the task does not require graph certainty.
- Interfaces are small enough to unit test independently.

## Phase 3: Code Graph Index MVP

Build local graph-based code scanning before the planner begins executing code tasks.

Implementation shape:

- Scan Rust source directories and directory-level `AGENTS.md` summaries.
- Extract files, modules, structs, enums, traits, impls, functions, methods, tests, imports, call references, type references, and trait implementations.
- Store the graph locally in a lightweight MVP format such as serialized JSON or SQLite tables.
- Track file mtimes or content hashes to detect stale graph data.
- Expose read-only graph query tools to the orchestrator and approved workers.
- Use hybrid retrieval: graph lookup narrows the relevant code neighborhood, then text, BM25, or vector search ranks concrete snippets.
- Fall back to normal repo search when graph data is stale or incomplete, and record the degraded retrieval path in the trace.

MVP constraints:

- Target Rust repositories first.
- Keep the graph local-only.
- Do not require Neo4j or another external graph database.
- Allow full graph rebuild as the baseline before optimizing incremental updates.
- Treat graph lookup as an augmentation layer, not a replacement for direct file inspection.

Acceptance criteria:

- The graph can answer symbol lookup, callers, callees, import neighbors, related tests, directory summary, and impact-radius queries.
- The graph records enough path and span metadata for the orchestrator to retrieve exact source snippets.
- Stale-index detection works.
- Graph lookup decisions are persisted in traces.

## Phase 4: Local Planner MVP

Implement planning without execution first.

Implementation shape:

- Add a dry-run orchestrator path.
- Connect to a local OpenAI-compatible Mellum2 endpoint.
- Give Mellum2 concise graph summaries and graph-query tool descriptions.
- Require Mellum2 to request graph context before planning code edits that touch existing source.
- Prompt Mellum2 to return only the typed action-plan JSON.
- Parse and validate planner output.
- Run policy validation against every planned step.
- Persist planner prompt, raw planner output, parsed plan, graph queries, policy decisions, and validation result.
- Do not execute worker steps in this phase.

Default local runtime:

- `vLLM` or `SGLang` serving `JetBrains/Mellum2-12B-A2.5B-Thinking`.
- Endpoint compatible with `/v1/chat/completions`.
- Planner prompt should be deterministic enough for schema compliance; start with low temperature for MVP reliability.

Acceptance criteria:

- A user task can produce a validated local plan without network access to cloud services.
- The planner uses graph context for repository-aware code tasks.
- Invalid or unsafe plans are rejected with clear errors.
- Dry-run traces are written for successful and rejected plans.

## Phase 5: Local Worker Delegation

Add execution through isolated local workers.

Implementation shape:

- Launch workers as subprocesses.
- Communicate through JSON-RPC over stdio.
- Give each worker only scoped context, graph-selected snippets, explicit objective, tool allowlist, budget, and expected response schema.
- Run each worker in an isolated sandbox or isolated working directory.
- Start with fake deterministic workers for integration testing.
- Add real local agent workers after protocol tests are stable.
- Let workers request additional graph lookups only when `code_graph.*` tools are included in their allowlist.
- Aggregate worker outputs and graph queries into the run trace.

Worker lifecycle:

1. Orchestrator creates sandbox and context bundle.
2. Orchestrator starts worker subprocess.
3. Orchestrator sends `initialize` with protocol version and capabilities.
4. Orchestrator sends `run_task`.
5. Worker returns structured response.
6. Orchestrator validates response schema.
7. Orchestrator records result and tears down or archives sandbox according to policy.

Acceptance criteria:

- Fake worker success, failure, malformed output, and timeout are covered by tests.
- Worker tool access is limited to the allowlist from the approved plan.
- Worker graph lookups are recorded and scoped.
- Failed workers do not corrupt the orchestrator trace, graph index, or other worker sandboxes.

## Phase 6: Policy And Redacted Cloud Fallback

Add controlled cloud delegation after local graph lookup and local execution are working.

Implementation shape:

- Enforce local-first routing by default.
- Allow cloud routing only when the planner requests it, project policy permits it, and redaction succeeds.
- Add a redaction pipeline before any cloud model call.
- Redact graph-derived snippets, paths, symbol metadata, and directory summaries according to cloud policy.
- Record redaction summaries in the trace.
- Add ChatGPT/OpenAI as the first named cloud provider behind `ModelClient`.
- Keep the provider interface generic enough for other OpenAI-compatible providers later.

Cloud delegation rules:

- Raw repository context and raw graph dumps must never be sent to cloud workers by default.
- Cloud workers receive only the minimal subtask and redacted context required for that step.
- Cloud responses are treated as worker outputs and must pass the same response validation as local worker outputs.
- If redaction cannot produce acceptable context, the step is blocked or rerouted locally.

Acceptance criteria:

- Cloud fallback is blocked when project policy disallows it.
- Cloud fallback is blocked when redaction fails.
- Cloud fallback succeeds only when policy allows it and redaction produces an auditable summary.
- Cloud delegation uses the same trace and aggregation path as local workers.

## Phase 7: Synthesis, Auditability, And Validation

Complete the end-to-end agent loop.

Implementation shape:

- Add final response synthesis from worker outputs, graph lookup decisions, validation results, and trace summaries.
- Include unresolved risks and blocked steps in the final response.
- Report whether graph context was used, stale, incomplete, or bypassed.
- Add trace browsing or export for debugging and review.
- Add a replay mode that can inspect prior traces without re-calling models.
- Add validation commands for the Rust implementation.

Validation commands:

```sh
cargo fmt --check
cargo clippy --all-targets --all-features
cargo test --all-features
```

Acceptance criteria:

- A task can be planned locally and executed by fake local workers end to end.
- A task can complete without any cloud access.
- A code task can identify impacted files and tests using graph lookup before generation.
- A cloud-routed task sends only redacted context and records the policy decision.
- Final responses cite which steps succeeded, failed, or were blocked.
- Traces are sufficient to reconstruct planner decisions, graph lookups, retrieval choices, and delegated work.

## Test Plan

Unit tests:

- Graph node and edge serialization.
- Rust symbol extraction for modules, structs, enums, traits, impls, functions, methods, and tests.
- Graph queries for symbol lookup, callers, callees, imports, related tests, directory summaries, and impact radius.
- Stale-index detection.
- Planner JSON parsing accepts valid plans.
- Planner JSON parsing rejects missing required fields.
- Planner JSON parsing rejects unknown or unsafe delegate targets.
- Policy defaults to local-first routing.
- Policy blocks cloud requests without permission.
- Policy requires redaction before cloud delegation.
- Redaction handles representative secrets, paths, user data, source snippets, and graph metadata.
- Worker protocol structs serialize and deserialize correctly.

Integration tests:

- Fake graph index produces valid context for a dry-run plan.
- Fake graph index fails or is stale and falls back to normal search.
- Fake planner produces a valid dry-run plan with graph queries.
- Fake planner produces malformed JSON and fails closed.
- Fake worker returns success.
- Fake worker returns failure.
- Fake worker times out.
- Fake worker returns malformed output.
- Orchestrator persists traces for graph queries, successful steps, failed steps, and blocked steps.

Acceptance tests:

- Local-only planning works without cloud credentials.
- Local-only worker delegation completes a simple task.
- Local-only code task uses graph lookup to identify impacted files and tests.
- Cloud fallback is blocked without redaction.
- Redacted ChatGPT/OpenAI delegation succeeds when policy permits it.
- Replay mode can inspect a previous run without model calls.

## Implementation Principles

- Keep functions small, focused, and testable.
- Prefer pure helpers for schema validation, graph query planning, routing decisions, redaction decisions, and trace formatting.
- Keep model I/O, graph scanning, retrieval ranking, policy validation, sandbox management, and worker protocol handling in separate modules.
- Fail closed when model output is malformed or policy is ambiguous.
- Fail soft from graph lookup to normal search when graph certainty is not required.
- Use typed Rust structs and enums instead of loosely structured maps for core protocol data.
- Treat cloud use as an explicit capability, not a default dependency.

## Assumptions

- The first implementation target after this document is a Rust CLI MVP.
- Mellum2 is served locally through an OpenAI-compatible endpoint.
- vLLM and SGLang are the initial documented local serving examples.
- The first graph implementation targets Rust repositories.
- The graph index is local-only and read-only from the planner's perspective.
- The MVP graph store is lightweight JSON or SQLite, not an external graph database.
- Graph retrieval augments, but does not replace, grep, BM25, vector lookup, or direct file inspection.
- Worker agents communicate over subprocess JSON-RPC and run in isolated sandboxes.
- ChatGPT/OpenAI is the first cloud provider named explicitly.
- Cloud fallback is allowed only after redaction and policy approval.
- Persistent state is limited to project plans and execution traces, not a full long-term memory system.
- The system should remain useful when offline from cloud providers.
