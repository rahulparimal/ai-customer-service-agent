# GPT-5.6 / GPT-5.4 / GPT-5.5 Engineering Workflow

These models are development-time reviewers and generators. They are not runtime dependencies of the deployed customer-service agent.

## Stage 1 — GPT-5.6: solution and technical design authority

Inputs:

- Business requirements and supported customer journeys
- Security, privacy, retention, and regulatory constraints
- Expected concurrency and latency objectives
- Approved open-source licenses and infrastructure

Required outputs:

- Context, container, component, and deployment diagrams
- ADRs for transport, STT, LLM serving, TTS, vector storage, and persistence
- API/WebSocket contracts and sequence diagrams
- Data classification and threat model
- NFRs, SLOs, capacity assumptions, and test acceptance criteria
- Implementation backlog broken into independently testable stories

Design prompt:

```text
Act as the architecture authority for an open-source, production-grade voice customer-service agent. Review the attached requirements and repository. Produce updated architecture decisions, precise interfaces, failure modes, security controls, performance budgets, implementation stories, and acceptance tests. Do not write application code. Identify every unresolved assumption and choose a conservative default where evidence is missing.
```

Gate: implementation does not start until interfaces, NFRs, and acceptance tests are explicit.

## Stage 2 — GPT-5.4: code development

Each request should be limited to one story or bounded component.

Development prompt:

```text
Implement story <ID> in this repository exactly against the approved design and contracts. Write production code, tests, migrations, configuration, and operational documentation. Preserve cancellation safety, tenant isolation, typed interfaces, structured logging, and error handling. Do not weaken tests or change public contracts without an ADR. Return a change summary, files changed, tests executed, and remaining risks.
```

Mandatory code-generation rules:

- No unbounded queues, buffers, retries, or conversation history
- No business action directly from free-form LLM output
- No secrets or PII in logs
- All blocking inference runs outside the event loop
- All external calls have timeouts and cancellation behavior
- Every bug fix includes a regression test

## Stage 3 — GPT-5.5: review, correction, and troubleshooting

Review prompt:

```text
Review the implementation for correctness, security, async cancellation, race conditions, resource leaks, tenant isolation, prompt injection, dependency/license risk, observability, failure recovery, and performance. Reproduce failures where possible. Fix confirmed defects and add regression tests. Do not perform cosmetic refactoring unless it removes operational risk. Report severity, evidence, fix, and verification for each finding.
```

Required review passes:

1. Contract and functional correctness
2. Security and privacy
3. Async concurrency and cancellation
4. Inference latency and memory
5. Deployment and recoverability
6. Test quality and missing cases

## Pull-request evidence package

Every change should contain:

- Linked requirement/story and design decision
- Test output
- Static analysis and dependency scan
- Before/after latency or resource data for performance changes
- Rollback instructions
- Review findings resolved or explicitly accepted

## Troubleshooting handoff format

```text
Symptom:
Environment and versions:
Expected behavior:
Actual behavior:
Reproduction steps:
Relevant logs and correlation id:
Recent changes:
Metrics before/during failure:
Hypotheses already tested:
```

This format prevents the review model from guessing without evidence.
