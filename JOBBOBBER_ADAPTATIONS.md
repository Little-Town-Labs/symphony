# JobBobber adaptations of the Symphony spec

Companion doc to [`SPEC.md`](SPEC.md). Records which Symphony patterns were adopted for JobBobber's harness, where each pattern landed in the spec, what was explicitly *not* adopted, and what's still genuinely open.

This is not a spec port. JobBobber and Symphony solve different problems on different stacks. Symphony orchestrates coding agents implementing engineering tickets in a code repo. JobBobber orchestrates negotiation agents working long-running marketplace tickets. The shapes rhyme; the implementations don't.

The point of this doc is to make sure the rhyming was deliberate design influence, not accidental cargo-culting. With pass two of the spec transformation complete, this doc serves as a decisions-taken log: every "we'll do X" is now either implemented in `SPEC.md` (pointer below) or explicitly deferred.

## Reading order

If you haven't read the source materials, do them in this order. The *why* is here and in the OpenAI posts; the *how* is in `SPEC.md`.

1. The OpenAI blog post: <https://openai.com/index/open-source-codex-orchestration-symphony/>
2. The harness engineering post that Symphony cites as a prerequisite: <https://openai.com/index/harness-engineering/>
3. The README in this repo
4. This document, for the design rationale and the `SPEC.md` cross-references
5. `SPEC.md` itself, end-to-end, as the authoritative contract

The original Elixir reference implementation is removed from this fork; it is interesting from a curiosity standpoint but is not on the critical path. The spec is the product.

## Patterns adopted (and where they landed)

### 1. Proof-of-work as a first-class artifact (the match dossier)

**Symphony pattern.** When an autonomous agent hands work back to a human, it attaches a structured bundle of evidence: CI status, PR review feedback, complexity analysis, walkthrough video. The human reviews the bundle, not the raw work.

**JobBobber adaptation — landed.** Every match-ticket negotiation produces a versioned `MatchDossier` (`SPEC.md` §4.1.8) when the run reaches a terminal `complete` or `inconclusive` state. The dossier carries:

- Per-side, per-dimension rubric breakdowns with deterministic weighted totals
- Per-audience redacted transcript projections (seeker, employer, auditor, A2A receiver), produced once and stored on the dossier rather than computed at delivery
- Each side's one-paragraph rationale in the agent's own voice
- Flags surfaced for human attention, reconciled from both sides' proposals
- Full version metadata: contract refs, ruleset ref, harness version, model invocations
- Optional signature in production posture

The dossier is the artifact that flows to the human-review surfaces, the audit log, and downstream consumers (notification, ATS webhook, A2A). One canonical dossier; per-audience projections are pre-computed and stored alongside it (§13 explicitly separates the dossier from the canonical transcript store with its stricter access controls).

Open questions resolved by the spec:

- *Should the dossier be signed?* Yes. `dossier.signing_enabled` defaults to true in production (§6.4); the signature covers all dossier fields except the signature object itself using deterministic canonical serialization (§15.4). A verification helper is required.
- *Redaction policy when one canonical dossier feeds multiple consumers with different visibility rights?* Pre-computed per-audience projections stored on the dossier (§4.1.8 `transcript_projections`), not derived at delivery time. Each projection is the result of applying the privacy ruleset at the corresponding audience's disclosure stage.

### 2. The harness around the rubric, not the rubric inside the prompt

**Symphony pattern.** Agents only do good work when the environment around them gives structured inputs, deterministic verifications, and clear contracts. Symphony's harness is the test suite, lint, CI, and `WORKFLOW.md`.

**JobBobber adaptation — landed.** The scoring rubric is a versioned, structured artifact in its own registry (§4.1.6, §5.4). Concretely:

- Rubrics live in a versioned registry; `(rubric_id, version)` is immutable for all time (§5.1).
- Prompt templates MUST NOT embed dimension weights or scoring guidance. Rubric registry resolution and prompt rendering are separate paths (§5.4, §12.2).
- Agents score each dimension independently against the rubric. The harness computes the weighted total deterministically (§5.4, §16.7). Any holistic score the model produces is ignored and audited (§10.4) — this is in the production CI gate (§17.5, §18.2).
- Each dossier records the exact rubric version applied (§4.1.8 `version_metadata`).
- Each rubric version MUST carry a `bias_test_ref`. Production posture refuses to dispatch a run whose rubric reference lacks a bias-test artifact (§5.4, §17.1, §18.2). The bias-test artifact's content and methodology are out of scope for the spec; the dispatch-time gating is normative.

Why it matters. "The model said 6.5" doesn't survive an EEOC audit. "Here are six rubric dimensions, each scored against versioned criteria, weighted deterministically by the harness" does. The spec treats this as defensibility, with the bias-test gate making it impossible to ship a production rubric without an accompanying bias-test artifact.

Implementation notes still relevant:

- The two sides have different rubrics. `seeker.standard`-style and `employer.executive_search`-style are example identifiers in the spec; concrete rubric content is product/policy work.
- Rubric changes are version migrations. A run dispatched under rubric version A completes under rubric version A; a registry update does not invalidate in-flight runs (§7.4, §14.3).

### 3. Versioned agent contracts

**Symphony pattern.** Agent instructions live in `WORKFLOW.md` inside the repo so the agent's behavior is versioned alongside the code it's modifying.

**JobBobber adaptation — landed.** Symphony's `WORKFLOW.md` becomes the `AgentContract` (§4.1.2, §5). Contracts live in a durable versioned registry rather than a per-repo file. Each contract pins references to a versioned prompt template, a versioned rubric, a tool surface, model selection, and runtime settings. Every dossier records which `(contract_id, version)` produced it.

The shift away from `WORKFLOW.md`-style files is deliberate: JobBobber has one product repo and a separate harness deployment, with no analog to "the repo the agent is modifying." The versioning benefit comes from registry immutability + dossier metadata, not from a per-repo file convention.

### 4. Isolation between runs

**Symphony pattern.** Every implementation run is isolated — its own working directory, its own context, no state bleed between runs.

**JobBobber adaptation — landed.** §9 codifies five isolation invariants:

1. Per-run isolation. Context storage MUST be keyed by `run_id`; storage keyed only by `principal_id` violates the invariant.
2. Per-side isolation within a run. Seeker and employer contexts MUST NOT share storage.
3. Filter-mediated counterparty access. Side runners read counterparty data only through `counterparty_view` or `counterparty_filtered` tool results.
4. Context destruction on terminal transition.
5. No state inheritance across re-negotiations.

Invariants 1, 2, 3 are pinned as production CI gates (§17.4, §18.2): cross-match prompt-injection containment, per-side isolation, and counterparty-access type-level prohibition all block deployment if they regress. The "explicit invariant test" the original adaptation doc called for is now a normative requirement.

### 5. Run-to-completion

**Symphony pattern.** An autonomous run never blocks waiting for human input mid-execution. If the agent needs human input, the run fails and hands off; the run does not pause.

**JobBobber adaptation — landed.** Run-to-completion is the harness contract (§7, §10.5):

- Agents MUST NOT pause for human input mid-negotiation.
- Inability to score surfaces as an `inconclusive` dossier with flags describing what would resolve it (§4.1.8, §16.6) — not as a paused run.
- The harness MUST NOT advertise any tool whose semantics include "ask the principal" or "wait for human confirmation"; that constraint is enforced by a §17.5 test that scans the catalog.
- The seeker conversational surface (a person chatting with their agent outside a negotiation run) is explicitly not the harness's concern. Only the negotiation run between two agents on a match ticket is autonomous, and that run runs to completion.
- Failure modes that can't even produce an `inconclusive` dossier (timeouts, tool failures beyond retry) terminate as `timed_out` or `tool_failure` — but the harness still attempts a best-effort `inconclusive` dossier first (§7.1, §14.2). Re-negotiation is a fresh `run_id` triggered by an explicit `match_ticket.renegotiation_requested` event, never a transparent run-level retry.

The round cap (§7.2) bounds re-negotiation depth at the round level. The default is `runs.default_round_cap = 3` (§6.4); contracts MAY specify a lower cap via `round_cap_contribution`, and the effective cap is the minimum across both sides. Empirical validation of `3` is still open.

### 6. Untrusted-input posture

**Symphony pattern.** Tracker data, repo contents, prompt inputs, and tool arguments cannot be assumed trustworthy just because they originate inside a normal workflow.

**JobBobber adaptation — landed.** §15.1 enumerates the assumed-untrusted inputs (seeker resume text, employer JD/req text, ATS-imported content, tool-returned text, A2A-received content). The posture is enforced at three structural points:

- The privacy filter (§15.2) applies the ruleset's `untrusted_input_policy` deterministically before content reaches either side's view. Failing closed is mandatory.
- Prompt construction wraps every untrusted free-text field with sentinels containing a per-run nonce (§12.4) so a malicious payload cannot forge a closing sentinel.
- The privacy filter MUST contain no model invocation (§15.2). This is in the production CI gate as a no-gateway-reachability test (§17.6, §18.2): the filter's call graph cannot reach the gateway client.

A sentinel-injection attack test (§17.6) is also in the production gate.

### 7. Tool-call extension model

**Symphony pattern.** Implementations may expose a limited set of optional tools to the agent. Supported tools are advertised on session startup. Unsupported tool requests return a graceful failure and the session continues.

**JobBobber adaptation — landed.** The contract's `tool_surface` (§4.1.2, §5.5) is the per-run versioned advertisement of available tools. Each descriptor carries `(name, version, input_schema, output_schema, disclosure_class)`. New tools added to the catalog don't break older active runs — runs only see tools their contract version pinned (§5.5). Unsupported tool calls return `tool_unsupported` and the turn continues (§10.3).

The `disclosure_class` field — `principal_self`, `counterparty_filtered`, `platform_open` — is JobBobber-specific and routes tool outputs through the privacy filter when appropriate (§9.4, §10.3). The harness tool dispatcher is the only path that invokes tools; direct tRPC/SDK calls from side-runner code are rejected at type-check (§10.3, §17.5). Disclosure-class enforcement for `counterparty_filtered` outputs is in the production CI gate (§17.5, §18.2).

## Patterns explicitly not adopted

**Linear-as-control-plane.** Symphony reads from an external issue tracker because the engineering team already lives in Linear. JobBobber's match tickets are first-class entities in the JobBobber Postgres database. The `MatchTicketReader` (§3.1, §11) reads from there directly.

**Polling-based scheduler.** Symphony watches the board and spawns runs when tickets are ready. JobBobber's harness is event-driven through Inngest (§8). Six Inngest functions handle dispatch, coordination, per-side runs, privacy filtering, dossier production, and run invalidation. No polling loop.

**Codex App Server / JSON-RPC protocol.** Codex-specific. JobBobber uses Anthropic and OpenAI through the Vercel AI Gateway (§10). Side runners drive model sessions through the gateway, not through a stdio app-server.

**Elixir / BEAM runtime.** Symphony picks BEAM for managing hundreds of long-running supervisory processes; right tool for that job. JobBobber's stack is Next.js / tRPC / Inngest / Postgres on Vercel. Inngest provides durable workflow execution, retries, and supervision — Inngest is the BEAM-equivalent.

**`WORKFLOW.md` as a per-repo file.** Replaced by the versioned contract registry (§5.1). The versioning benefit comes from registry immutability + dossier metadata, not from a per-repo file convention.

**Hot-reload of policy files.** Symphony supports dynamic `WORKFLOW.md` reload. JobBobber deliberately does not (§6.2): harness config is frozen per deployment, and a run dispatched under one config completes under that config. Policy iteration happens in the contract/rubric registries, which already support fast iteration without touching harness config.

**Filesystem workspaces.** Symphony's per-issue workspace becomes JobBobber's per-side, in-memory `NegotiationContext` (§4.1.5, §9). The harness owns no filesystem state per run; durability comes from the audit log and dossier persistence, not from disk-backed workspaces. This eliminated Symphony's path-safety invariants and replaced them with the §9.5 isolation invariants.

**The "ask Codex to implement Symphony" build path.** Generating a TypeScript port of Symphony would give us a working Symphony, not a JobBobber. We're already a system that happens to share some patterns; building a literal port creates something we don't need and would have to maintain separately.

**Symphony's HTTP dashboard / runtime status endpoint.** The operator status surface for JobBobber is the JobBobber product UI consuming the audit log and `dossier.produced` events (§13.7). Embedding an HTTP server in the harness layer is unnecessary; the operator API (§13.8) is OPTIONAL and authentication-required, not a public dashboard.

**Symphony's SSH worker extension.** Removed. Vercel/Inngest is the execution substrate; remote-host execution is not a deployment shape JobBobber will use.

## Things still genuinely open

Items the spec does not pin and that need product/design work or empirical validation:

- **Calibration across users.** If seeker A's agent scores generously and seeker B's agent scores harshly, "5 or higher" means different things across users. The spec records per-dimension scores deterministically; whether those dimension scores get normalized against a population baseline before threshold checks is a downstream consumer concern, not a harness concern. Decide product-side whether to calibrate or accept that thresholds are personal-agent settings the human tunes through experience.
- **Asymmetric outcomes UX.** Match ticket scored 6.5 (seeker, clears their 5) and 4 (employer, doesn't clear their 7). Seeker hears about it; employer doesn't. The spec produces one canonical dossier with per-audience projections, so the data is there. What the seeker actually sees, and how they push back to trigger re-negotiation, is product UI. The dossier shape supports it.
- **Round-cap default validation.** The spec pins `runs.default_round_cap = 3` (§6.4). Empirical validation against real negotiations is needed before this becomes load-bearing in production.
- **Bias-test methodology.** The spec requires every production rubric version to carry a `bias_test_ref` and refuses dispatch otherwise (§5.4, §17.1, §18.2). The methodology behind bias-testing — what populations, what metrics, what thresholds — is a policy/legal artifact referenced from the rubric registry, not encoded in the harness.
- **Rubric content.** Dimension lists, weights, and scoring guidance for both seeker-side and employer-side rubrics are product/policy work. The spec gives them shape and immutability discipline; the content is not in scope.
- **Disclosure-stage progression triggers.** The spec defines `disclosure_stages` on the privacy ruleset (§4.1.7) and records the active stage on every projection. The trigger that advances a run from one stage to the next is implementation-defined per ruleset version; whether progression is round-counted, signal-driven, or hybrid is open.
- **Audit retention durations.** The spec requires retention to be no shorter than the regulatory floor and references `audit.retention_class` (§6.4); concrete durations per regulatory regime are policy artifacts.
- **A2A receiver projection.** The dossier carries an `a2a_receiver` transcript projection (§4.1.8). The actual A2A protocol JobBobber will speak to external negotiation peers is out of scope here; when that protocol stabilizes, the projection rules will need to be consistent with it.

## Pinning

This doc is written against Symphony at commit `58cf97d` (`fix(elixir): configure Codex app-server model via config`) — the last upstream commit before this fork, and the version of `SPEC.md` originally preserved in this repo before pass-two transformation. The harness patterns originated in Symphony's initial public release (`b0e0ff0`) and were unchanged through `58cf97d`; the later commit only refined spec prose (RFC 2119 normative language, tightened "may" vs "MUST" wording) without altering the pattern set.

`SPEC.md` in this repo is no longer the upstream Symphony text. It is the JobBobber Agent Harness Specification, transformed section-by-section from Symphony's structure to JobBobber's contract. Git history preserves the original.

OpenAI has stated they don't plan to maintain Symphony as a product, so we're not tracking upstream. If a major Symphony update worth re-evaluating ever lands, that's a fresh research task, not a continuous integration burden.

---

*Maintained by the JobBobber team. Updated when the design decisions in this doc actually change — not when the Symphony repo changes.*
