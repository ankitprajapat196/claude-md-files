# Production-Ready Agent / Workflow Checklist (CCAF Course)

> Order = build order. Go top to bottom. Tick each box before shipping.

```mermaid
flowchart LR
    A[1. Decide] --> B[2. Architecture] --> C[3. Prompts] --> D[4. Tools]
    D --> E[5. Response Contract] --> F[6. Validation] --> G[7. Errors]
    G --> H[8. Loop Control] --> I[9. Guardrails] --> J[10. Context]
    J --> K[11. State] --> L[12. Human-in-Loop] --> M[13. Security]
    M --> N[14. Evals] --> O[15. Ops]
```

---

## 1. Decide: Workflow or Agent?
- [ ] **Start simple** — single LLM call → workflow → agent. Add complexity only when needed.
- [ ] **Agent only if** steps are unpredictable, need dynamic tool choice, or need replanning on observations.
- [ ] **Check trade-offs** — cost vs capability, auditability vs flexibility, reliability vs adaptability.
- [ ] **Hybrid allowed** — deterministic workflow with agent behavior only at the steps that need it.

## 2. Architecture & Decomposition
- [ ] **Decompose the task** — right-sized subtasks (not monolithic, not over-split); clear handoff points.
- [ ] **Pick the pattern** — sequential (dependencies), parallel (independent work), hierarchical (large scope).
- [ ] **Draw a dependency graph** — anything without a dependency can run in parallel.
- [ ] **Single vs multi-agent** — multi-agent only if gains beat orchestrator overhead, fan-out and stitching cost.
- [ ] **Topology** — hub-and-spoke (central control), pipeline (fixed stages), P2P (rare, hard to debug).
- [ ] **Orchestrator jobs** — decompose → assign → aggregate (dedupe, resolve conflicts, find gaps).
- [ ] **Sub-agents** — narrow scope, isolated context, only the tools they need, clear authority boundary.
- [ ] **Message schema** — typed, versioned messages between agents (task ID, input, constraints, expected output).
- [ ] **Replanning** — define when the plan is revised (failed step, new info) vs when to stop.

## 3. Prompt Design
- [ ] **System prompt with 5 sections** — Role, Context, Instructions, Constraints, Examples.
- [ ] **Clear & specific** — direct instructions, concrete criteria, defined output format; no vague words.
- [ ] **Scope + constraints explicit** — what is in/out of scope, what must never happen.
- [ ] **No conflicting instructions** — one rule per behavior; check across system prompt, tools and examples.
- [ ] **XML tags** — separate instructions, documents, data and examples; long documents at the top, query at the end.
- [ ] **Few-shot examples** — diverse, include edge cases, clean (no contamination), consistent format.
- [ ] **Reasoning** — CoT / extended thinking only for multi-step reasoning tasks, not simple lookups.
- [ ] **Output steering** — prefill or format anchoring where the exact output shape matters.

## 4. Tool Design
- [ ] **Descriptions first** — what it does, when to use it, when NOT to use it; inputs and outputs explained.
- [ ] **No overlap** — consolidate similar tools, namespace them, add explicit exclusions.
- [ ] **Input schema** — clear param names, required vs optional chosen deliberately, enums for fixed values.
- [ ] **Strict tool use** — enable where params must exactly match the schema.
- [ ] **Tool scoping** — each agent gets only the tools it needs (fewer tools = better routing).
- [ ] **tool_choice** — `auto` / `any` / specific tool / `none` chosen per step.
- [ ] **Large tool sets** — use tool search / deferred loading instead of loading all definitions upfront.
- [ ] **Built-in vs custom** — prefer built-in tools when they fit; custom for business logic.
- [ ] **MCP** — right primitive (tools = model, resources = app, prompts = user); right transport (STDIO local, Streamable HTTP remote).
- [ ] **Test routing** — run sample queries and confirm the right tool gets chosen.

## 5. Response Contract (Success / Partial / Error)
- [ ] **Status enum on every tool response** — `success`, `empty_result`, `partial_result`, `access_failure`.
- [ ] **Never treat empty as "nothing exists"** — distinguish genuine empty from access failure (tool wrapper).
- [ ] **Success** — data + metadata (source, timestamp) so downstream can trust and trace it.
- [ ] **Partial success** — return completed items, failed items with reasons, and what is safe to retry.
- [ ] **Tool error response (4 fields)** — stable error code, human message, context (what/why), suggested action.
- [ ] **User vs system error** — user error = fix input; system error = retry/fallback/escalate.
- [ ] **Agent-to-coordinator error envelope** — error type, severity (`warning` / `recoverable` / `fatal`), context, recovery hint.
- [ ] **Every severity has a defined policy** — warning = log & continue, recoverable = retry/fallback, fatal = halt & escalate.

## 6. Structured Output & Validation
- [ ] **Structured outputs / JSON schema** for machine-consumed output.
- [ ] **Know the guarantee gaps** — some schema constraints are only SDK-transformed, not enforced; validate them in code.
- [ ] **Schema design** — required only when truly always present; nullable for missing data (prevents hallucinated values).
- [ ] **Two validation layers** — schema (structure) + semantic (meaning, business rules).
- [ ] **Validation loop** — on failure, feed the specific error back and retry within a limit.
- [ ] **Hallucination prevention** — ground on source, require citations/quotes, allow "not found".
- [ ] **Reviewer pattern** — second pass (or separate reviewer) for high-stakes output.

## 7. Error Handling
- [ ] **Classify errors** — tool (transient vs permanent), reasoning, environment.
- [ ] **Retry only transient errors** — exponential backoff + jitter + max attempts; never retry permanent errors.
- [ ] **Idempotency** — idempotency keys or conditional writes so retries don't duplicate side effects.
- [ ] **Circuit breaker** — closed → open → half-open for unreliable services; thresholds set deliberately.
- [ ] **Fallback chain** — primary → alternate tool/model → cached/degraded result → human; tell the user when degraded.
- [ ] **Silent failure detection** — output validation, state verification, sanity checks.
- [ ] **Propagation boundary** — fix locally what you can; propagate what changes the plan; always log both.
- [ ] **Coordinator recovery options** — backoff, fallback agent, accept partial completion, human escalation.

## 8. Loop Control
- [ ] **4 stop conditions** — task complete, error threshold, budget exhausted, stop signal.
- [ ] **Turn/token/cost budget enforced in code**, not just in the prompt.
- [ ] **Progress vs spinning** — detect repeated identical calls or no new info → stop or escalate.
- [ ] **Iterate vs escalate** rule defined.

## 9. Guardrails & Enforcement
- [ ] **Critical rules in code, not only prompt** — prompts guide; code enforces.
- [ ] **Pre-execution checks** — validate inputs, permissions, allow/deny lists before actions.
- [ ] **Post-execution checks** — validate output before it reaches users/systems.
- [ ] **Runtime checks** — budgets, rate limits, anomaly detection while running.
- [ ] **Hard block vs soft warning** — chosen by risk level.
- [ ] **Claude Code** — hooks for deterministic control, permission modes, deny rules for sensitive files.

## 10. Context Management
- [ ] **Budget the context window** — input + output must fit; reserve room for output.
- [ ] **More context ≠ better** — include only what the step needs (watch lost-in-the-middle).
- [ ] **Prompt caching** — stable content (system prompt, tools, docs) first, variable content last.
- [ ] **Trim verbose tool outputs** — return only needed fields; paginate/summarize big results.
- [ ] **Persistent fact block** — critical facts (IDs, amounts, decisions, constraints) kept outside summaries.
- [ ] **Extract-first summarization** — pull facts into the fact block before compressing history.
- [ ] **Trim strategy** — sliding window / summarize / compaction chosen deliberately.
- [ ] **Delegate verbose discovery to sub-agents** — main agent gets only the condensed result.

## 11. State & Memory
- [ ] **Durable state outside the context** — DB / memory tool / scratchpad for long-running tasks.
- [ ] **Checkpointing** — resume from the last good step after a crash.
- [ ] **Scratchpad hygiene** — structured blocks, prune stale entries, read at start + write at end.
- [ ] **Shared state rules** — who can write what; avoid conflicting writes in multi-agent setups.

## 12. Human-in-the-Loop
- [ ] **Escalation triggers** — hard (policy, deny list, high-risk action) + soft (low confidence, repeated failure).
- [ ] **Interruption points** — human approval before irreversible / high-impact actions.
- [ ] **Handoff payload (4 fields)** — conversation summary, triggering question (verbatim), what was tried, suggested next step.
- [ ] **Per-field confidence** — route individual fields to review, not whole records.
- [ ] **Calibrate confidence** — set thresholds from a validation set, not raw model scores.
- [ ] **Review queue** — prioritized, with SLAs; stratified sampling (confidence, category, stakes) for QA.
- [ ] **Provenance** — source, method, timestamp per value; conflicts annotated, not silently overwritten.
- [ ] **Close the loop** — reviewer decisions feed back into prompts, examples and thresholds.

## 13. Security
- [ ] **Least privilege** — minimal tool permissions, scoped credentials, read-only where possible.
- [ ] **Tool results are untrusted** — guard against prompt injection from fetched/external content.
- [ ] **Trust boundaries** — validate data crossing agent/MCP/server boundaries.
- [ ] **MCP production** — auth, versioning, rate limiting, monitoring; human approval for sampling.
- [ ] **No secrets in prompts/logs**; mask PII.

## 14. Evaluation
- [ ] **Test dataset** — typical, edge and adversarial cases (from real logs, hand-written, synthetic).
- [ ] **Grading** — code-based where possible (exact match, schema, regex); model-based for subjective quality.
- [ ] **Calibrate model graders** against human judgment.
- [ ] **Eval on every change** — prompt, tool, model or example changes run the full suite (regression).
- [ ] **Test failure paths** — tool errors, timeouts, empty results, partial results.

## 15. Observability, Cost & Deployment
- [ ] **Tracing** — trace/correlation ID through all agents and tool calls.
- [ ] **Log** — tool calls, inputs/outputs, errors, retries, escalations, token usage.
- [ ] **Cost tracking** — per run / per agent; right model tier per subtask.
- [ ] **Latency** — parallelize independent steps; stream where users wait.
- [ ] **CI/CD** — evals in pipeline; non-interactive runs with scoped permissions.
- [ ] **Versioning** — prompts, tools, schemas versioned; rollback plan ready.

---

## Quick Summary
Decide workflow vs agent first (start simple). Decompose the task, pick a pattern and topology, and keep sub-agents narrow with isolated context. Write a 5-section system prompt with clear constraints, XML structure and good examples. Design tools with strong descriptions, tight schemas and minimal scope. Every tool returns a status enum (success / empty / partial / access_failure) and structured errors (code, message, context, suggested action); agents propagate errors in an envelope with severity, and every severity maps to a defined policy. Validate structure and meaning, retry only transient errors with idempotency, use circuit breakers and fallbacks, and stop loops on 4 conditions. Enforce critical rules in code, manage context with fact blocks and trimming, persist state outside the window, escalate to humans with a full handoff payload, treat tool output as untrusted, eval on every change, and trace + cost-track everything in production.
