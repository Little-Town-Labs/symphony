# JobBobber adaptations of the Symphony spec

Companion doc to `SPEC.md`. Records what we're cribbing from Symphony for JobBobber's design, what we're explicitly *not* taking, and what's still open.

This is not a spec port. JobBobber and Symphony solve different problems on different stacks. Symphony orchestrates coding agents implementing engineering tickets in a code repo. JobBobber orchestrates negotiation agents working long-running marketplace tickets. The shapes rhyme; the implementations don't.

The point of this doc is to make sure the rhyming becomes deliberate design influence, not accidental cargo-culting.

## Reading order before working through this doc

If you haven't read the source materials, do them in this order. Most of the *why* is in the blog post; the spec is the *how*.

1. The OpenAI blog post: <https://openai.com/index/open-source-codex-orchestration-symphony/>
2. The harness engineering post that Symphony cites as a prerequisite: <https://openai.com/index/harness-engineering/>
3. The README in this repo
4. SPEC.md in this repo, read once end-to-end without trying to map it to JobBobber
5. SPEC.md again, this time with this adaptations doc open alongside it

The Elixir reference implementation is interesting from a curiosity standpoint but is not on the critical path. The spec is the product.

## Patterns we're adopting

### 1. Proof-of-work as a first-class artifact (the match dossier)

**Symphony pattern.** When an autonomous agent hands work back to a human, it attaches a structured bundle of evidence: CI status, PR review feedback, complexity analysis, walkthrough video. The human reviews the bundle, not the raw work.

**JobBobber adaptation.** Every match ticket produces a *match dossier* when it transitions to a state requiring human review. The dossier is a versioned, structured artifact that contains:

- The two scores with per-dimension rubric breakdowns
- The full negotiation transcript, redacted appropriately for each viewer
- Comp range overlap and other key fit signals stated explicitly
- The agent's one-paragraph rationale in its own voice
- Flags surfaced for human attention (visa needs, start-date constraints, deal-breakers)
- Versions of the rubric, prompts, and model used to produce the dossier

The dossier is the artifact that flows to the human reviewer, the audit log, and the webhook payload. Three consumers, one canonical bundle.

**Implementation notes.**

- Add a `MatchDossier` entity tied to the match ticket. Versioned because re-negotiation can produce new dossiers on the same ticket.
- Stable schema. Integrators (ATS webhooks, A2A receivers) rely on it.
- Schema lives in code with explicit type contracts; not invented per-call by the agent.
- Generated dossiers are stored, not just streamed — auditability requires durability.

**Open questions.**

- Should the dossier be signed (HMAC or similar)? Symphony alludes to harness hardening; for JobBobber, signed dossiers would make audit and tampering claims much cleaner.
- What's the redaction policy when the same dossier feeds three consumers with different visibility rights? Probably one canonical dossier with viewer-specific projection at delivery time.

### 2. The harness around the rubric, not the rubric inside the prompt

**Symphony pattern.** Agents only do good work when the environment around them gives structured inputs, deterministic verifications, and clear contracts. Symphony's harness is the test suite, lint, CI, and `WORKFLOW.md`.

**JobBobber adaptation.** Treat the scoring rubric as a *harness artifact*, not a prompt string. Concretely:

- The rubric is a versioned, structured object: dimensions, weights, scoring guidance per dimension, examples of what a 1/3/5/7/9 looks like for each dimension.
- The rubric lives in code (or in a versioned data store), not embedded in a prompt.
- The agent doesn't generate the score holistically. It scores each dimension separately against the rubric.
- The weighted total is computed deterministically from the dimension scores. Not asked of the model.
- Every dossier records which rubric version was used.

**Why this matters.** "The model said 6.5" doesn't survive an EEOC audit. "Here are six rubric dimensions, each scored against versioned criteria, weighted deterministically" does. This is a defensibility investment, not a quality investment, though it improves quality as a side effect.

**Implementation notes.**

- The two sides have different rubrics. Seeker-side rubric scores comp, role match, growth path, commute/remote, culture signals. Employer-side rubric scores skills coverage, experience depth, comp expectations, retention signals, education match. Both rubrics are independently versioned.
- Bias-test the rubric explicitly. NYC Local Law 144, EEOC, state-level AI hiring law all require this for automated employment decision tools. Build the auditing into the rubric refactor.
- Rubric changes are migrations. When you change a rubric, decide whether dossiers in flight re-score against the new rubric or stay with the old version.

### 3. Versioned agent contracts (the WORKFLOW.md analog)

**Symphony pattern.** Agent instructions live in `WORKFLOW.md` inside the repo so the agent's behavior is versioned alongside the code it's modifying.

**JobBobber adaptation.** Every agent prompt that affects scoring or negotiation behavior is versioned, and every match ticket records which version of which prompt it ran against. We already have `src/server/agents/` and the custom-prompts router — this is mostly a discipline change rather than a new system.

**What to add.**

- Prompt version (hash or tag) recorded on every dossier alongside the rubric version.
- A simple changelog discipline for prompt changes that affect scoring or negotiation.
- A way to A/B test prompt changes against real outcomes — the dossier history is the dataset.

### 4. Isolation between runs

**Symphony pattern.** Every implementation run is isolated — its own working directory, its own context, no state bleed between runs.

**JobBobber adaptation.** Every match ticket's agent context is isolated from every other match ticket's. The seeker agent working match ticket A has no access to match ticket B's state, even though both are agents-on-behalf-of-the-same-seeker.

**Why this matters.** Prompt injection cross-contamination. If an employer's req description (untrusted text) tries to manipulate the seeker agent during negotiation on one match ticket, that manipulation can't leak into the seeker agent's behavior on other match tickets. Each match ticket is its own sandbox.

**Implementation notes.**

- Inngest functions naturally give us this — each invocation is a fresh execution context. The discipline is to verify nothing in the agent layer accidentally shares state across match tickets.
- The prompt-guard component is the right place to enforce input sanitization on each match ticket boundary.
- Write an explicit invariant test: an instruction in match ticket A's negotiation context cannot influence match ticket B's behavior.

### 5. "Treat user-input-required turns as hard failure"

**Symphony pattern.** An autonomous run never blocks waiting for human input mid-execution. If the agent needs human input, the run fails and hands off; the run does not pause.

**JobBobber adaptation.** This is a real design call worth being explicit about. Lean toward Symphony's stricter rule: agent runs to completion (produces a dossier), then surfaces. Agents do not ping the human mid-negotiation asking "should I do X?"

**Why it's worth this constraint.** Autonomous-with-pauses is a much harder system to reason about. Run-to-completion-then-surface lets the human review a complete artifact instead of a snapshot.

**Edge cases to think about.**

- What happens when the agent genuinely doesn't have enough information to score? Not "ping the human" — "produce a dossier flagged as inconclusive, with a list of what would resolve it."
- The seeker conversational surface is *not* the agent run. The seeker can chat with their agent any time. But the *negotiation run* between two agents on a match ticket is the autonomous part, and that runs to completion.

### 6. Harness hardening / untrusted input posture

**Symphony pattern.** The spec is explicit that tracker data, repo contents, prompt inputs, and tool arguments cannot be assumed trustworthy just because they originate inside a normal workflow. Implementations should treat hardening as core, not optional.

**JobBobber adaptation.** Apply the same posture to every input the agent layer touches:

- Seeker-supplied resume text and profile fields: untrusted.
- Employer-supplied JD and req description: untrusted.
- ATS-supplied content via REST or A2A: untrusted.
- Anything that crosses a network boundary into the agent context: untrusted.

The privacy filter and prompt guard are the choke points where this posture is enforced. Document the threat model alongside them.

### 7. Tool-call extension model

**Symphony pattern.** Implementations may expose a limited set of optional tools to the agent (e.g. `linear_graphql`). Supported tools are advertised on session startup. Unsupported tool requests return a graceful failure and the session continues.

**JobBobber adaptation.** Same pattern for how JobBobber agents call back into the platform's tRPC layer for tools (job search, profile lookup, etc.):

- A versioned, advertised list of tools the agent has access to in this run.
- New tools added without breaking older agent contracts.
- Unsupported tool calls return a structured failure; they don't crash the run.

This matters for evolving the agent layer over time without breaking active match tickets.

## Patterns we're explicitly not adopting

**Linear-as-control-plane.** Symphony reads from an external issue tracker because the engineering team already lives in Linear. JobBobber's tickets are first-class entities in our database. Routing them through an external tracker would just add a moving part.

**Polling-based scheduler.** Symphony watches the board and spawns runs when tickets are ready. We have Inngest for event-driven execution. The pattern translates ("spawn a workflow when a ticket is ready") but the implementation is `inngest.send`, not a polling loop.

**Codex App Server / JSON-RPC protocol.** Codex-specific. We use Anthropic and OpenAI through the Vercel AI Gateway. Not relevant.

**Elixir / BEAM runtime.** Symphony picks BEAM for managing hundreds of long-running supervisory processes. It's the right tool for that job. Our stack is Next.js / tRPC / Inngest / Postgres on Vercel; Inngest already gives us durable workflow execution, retries, and supervision. Inngest is our BEAM.

**WORKFLOW.md as a per-repo file.** Symphony's `WORKFLOW.md` lives in the repo being modified by the agent. JobBobber has one product repo and its own service for the agents — there's no analog to "the repo the agent is modifying." We get the versioning benefit by recording prompt and rubric versions on the dossier, not by adopting the file convention.

**The "ask Codex to implement Symphony" build path.** Symphony's README suggests pointing a coding agent at the spec to generate a port. Generating a TypeScript port of Symphony would give us a working Symphony, but it'd be a Symphony-shaped thing, not a JobBobber-shaped thing. We're already a system that happens to share some patterns; building a literal port creates something we don't need and would have to maintain separately.

## Concrete next steps (in order)

1. **Define the dossier schema.** Fields, shape, versioning semantics. This affects the data model, the API, the webhook payloads, and the admin console design — settle it early.
2. **Refactor the rubric out of the prompt.** Two structured rubrics (seeker-side, employer-side), versioned, with explicit dimensions and weighting. Bias-test before shipping.
3. **Add prompt and rubric versioning to the match ticket.** Light lift, big audit/debugging payoff.
4. **Write the isolation invariants down.** Document and test that match tickets don't share state.
5. **Decide the "no pauses mid-run" rule.** Document the agent-run contract; build the inconclusive-dossier path for cases where the agent can't complete.
6. **Document the untrusted-input threat model.** The privacy filter and prompt guard already exist; this codifies what they're defending against.

## Things still open

- **Calibration across users.** If seeker A's agent scores generously and seeker B's agent scores harshly, "5 or higher" means different things. Decide whether to calibrate to a population baseline or accept that the threshold is a personal-agent thing the human tunes through experience.
- **Asymmetric outcomes UX.** Match ticket scored 6.5 (seeker, clears their 5) and 4 (employer, doesn't clear their 7). Seeker hears about it, employer doesn't. What does the seeker actually see? How do they push back to trigger re-negotiation? This is product design, not architecture, but the dossier shape should support it.
- **Re-negotiation cap.** Symphony's "treat user-input-required as hard failure" rule has us leaning toward hard caps on re-negotiation rounds. ~3 is the placeholder. Validate empirically.
- **Dossier signing.** Mentioned above. Decide once we have the schema settled.
- **Bias testing protocol.** Required for the rubric refactor. Find an industry-standard methodology rather than inventing one.

## Pinning

This doc is written against Symphony at commit `b0e0ff0082236a73c12a48483d0c6036fdd31fe1` (initial public release). OpenAI has stated they don't plan to maintain Symphony as a product, so we're not tracking upstream. If there's a major Symphony update worth re-evaluating, that's a fresh research task, not a continuous integration burden.

---

*Maintained by the JobBobber team. Updated when the design decisions in this doc actually change — not when the Symphony repo changes.*
