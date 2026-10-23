---
status: proposed
date: 2026-10-09
---

# ADR-0001 — LLM routing: router, gateway and provider are three components

## Summary

An Omega agent's LLM call passes through three distinct components. A **router**
decides which provider, endpoint and model to use by choosing one entry of
the catalogue, which the core writes into the envelope; it never makes a call and never holds a credential. A **provider**
makes the call and makes no decision. A **gateway** holds every credential, runs
out of the agent's process, and is reached by an address from configuration.
Routers are plugins — third parties write them and they are swapped often. The
gateway is not a plugin; it is infrastructure, swappable but addressed, not
loaded.

Anyone adding a router implements one function and touches nothing else. Anyone
adding a provider or an endpoint edits one catalogue file. **If either requires
changing core code, the design has been violated.**

## Context

Omega today uses one provider and one model for every call. Routing was proposed
in OMEGA-282 (2026-07-28, closed Done 2026-09-22) and drafted as a pull request,
#373, on 2026-10-01. Three problems made a decision necessary now.

**The design exists in two mutually contradictory versions in one comment
thread.** OMEGA-282's ticket description is empty; the design is three Jira
comments. The 2026-09-17 comment specifies tiers (`routine`/`deep`/`best`) and a
router that forwards to a gateway. The 2026-09-18 comment reverses all three
points — no tiers, the router does not pass the call on to the gateway, and the
router is not a provider — but nothing marks the earlier version superseded
except a prose heading inside a long comment.

**A second document differs on one question.** OMEGA-360, a proposal under
review, states that an LLM provider and an LLM router implement the same contract;
OMEGA-282 states the opposite.

**The implementation follows neither.** PR #373 returns a tier atom that
`providers/openrouter.py` interprets, which is the alternative OMEGA-282's own
rejected-options list names and rejects.

The underlying failure is that nothing was written down in a place that could be
read as a single current state. This ADR is that place.

## Premises

- *Today's nginx credential proxy is already an allow-list*: seven LLM locations
  (`anthropic`, `asicloud`, `openai`, `asione`, `openaiapi`, `openclaw`,
  `openrouter`), each pinning a `proxy_pass` upstream and injecting one
  `Authorization` header from a named environment variable — **read in
  `singnet/Omega@main:proxy/nginx.conf.template`, 2026-10-09.**
- *Credentials are already referenced by name, not value*, in that file
  (`${OPENROUTER_API_KEY}`), so a catalogue listing them contains no secret —
  **same source, same date.**
- *The endpoint axis already exists*: `openaiapi` uses `${OPENAIAPI_URL}` and
  `openclaw` uses `${OPENCLAW_URL}`, so provider and endpoint already come apart
  for self-hosted cases — **same source, same date.**
- *OMEGA-282's authoritative design is its 2026-09-18 comment*, which states that
  there are no tiers, that the router does not pass the call on to the gateway,
  and that the router is not a provider — **read in the OMEGA-282 Jira ticket
  (internal), 2026-10-09.**
- *OMEGA-360 states the opposite on the contract question*: a provider and a
  router implement the same contract, the one at `src/providers.py#L202` in
  `vsbogd/Omega@c193ab39` — **read in the OMEGA-360 proposal (internal; ask its
  author or Kabir Veitas), 2026-10-09; the named contract fetched from
  `vsbogd/Omega@c193ab39`, same date.**
- *OMEGA-360 proposes secrets inside plugins*, rejecting a co-located gateway on
  the grounds that adding a provider requires editing central configuration —
  **same document, same date.**
- *OMEGA-282 is the north star; OMEGA-360 is reviewed against it, not the
  reverse* — **operator, 2026-10-09.**
- *A production router already routes on conditions only visible after a call*:
  a router in use inside SingularityNET does triage, pre-emptive context
  escalation, overflow recovery, chunking, transient-overload fallback and
  per-model cooldown — **read in an internal SingularityNET repository (ask
  Kabir Veitas), 2026-10-09.**
- *PR #373's true delta is 10 files, +134/-9*, four commits ahead of PR #358 on
  which it is branched; the other 136 files are inherited — **GitHub compare
  API, 2026-10-09.**
- *The cheap/premium routing **policy** was agreed in the team's discussion; the
  **seam** was not.* On 2026-09-22 the design's author accepted whatever routing
  policy the implementer chose first. The detailed budget/premium plan that
  followed was answered on 2026-09-23 by pointing the pull request back to the
  2026-09-18 design — not an acceptance of the plumbing. OMEGA-282 was never
  updated either way. This ADR is consistent with both: a router may use tiers
  internally; nothing outside it sees one — **team discussion, 2026-09-22/23
  (internal; ask Kabir Veitas), read 2026-10-09.**
- *The implementer restated this design correctly before diverging from it*: on
  2026-09-21 the router's output was described as the complete envelope, with
  provider and model now determined, and on 2026-09-28 and 09-29 the
  core-dispatch gap was named as deferred with the "how" still open — **same
  source, same date.** The divergence in PR #373 is therefore not a misreading
  of the design.
- *The envelope's four optional fields are already agreed by the owner of that
  API.* Whether `LLMRequest` could get four optional fields — `provider`,
  `model`, `router` and `metadata`, all empty by default — was asked on OMEGA-282
  on 2026-09-18 and agreed the same day — **read in the OMEGA-282 Jira ticket
  (internal), 2026-10-09.** This is the carrier
  option C's envelope-completing router requires, so the decision below needs no
  further ratification on the envelope itself.
- *The gateway's **position** already exists in code; only its occupant is
  nginx.* `providers/lib_llm_ext.py:_create_client` reads
  `config_get_by_key("GATEWAY_URL")` and, when it is set, connects the provider
  at `{GATEWAY_URL}/{provider-name-lowercased}/` with `api_key="proxy"` — i.e. a
  gateway reached by an address from configuration, with the credential injected
  out of process. **Read in `singnet/Omega@193e9f58:providers/lib_llm_ext.py`,
  2026-10-09.** Consequence: this ADR's gateway component is a *replacement of
  the occupant*, not the introduction of a layer, and routing can ship without
  touching the credential path at all.
- *No adopted standard exists for describing a model's capabilities.*
  OpenRouter's `/api/v1/models`, LiteLLM's `model_prices_and_context_window.json`
  and models.dev each describe the same facts under different field names —
  **web search, 2026-10-09.**
- *The decisions' home, `omega-specs`, was created empty* on 2026-10-12 (then
  under `snet-omega-factory`; moved to `omega-flagship` on 2026-10-23), so 0001
  is the first number — **GitHub API, verified 2026-10-12.** (On 2026-10-09 the
  repository did not exist yet, and the number was an assumption.)

## Options considered

**A — One shared contract (OMEGA-360).** A router implements the same
`LLMProvider` interface as a provider: `start`, `stop`,
`chat(request) -> LLMResponse`, registered through `registerLLMProvider`. Costs
nothing to build; the registry already exists.

**B — A tier flag interpreted by providers (PR #373).** The router returns
`economy_model` or `premium_model`; each provider decides what that means. Small
and already written.

**C — Three components with three interfaces (chosen).** Router decides and
returns one catalogue entry; provider calls; gateway holds credentials out of
process.

**D — Secrets inside plugins (OMEGA-360 p.7).** Each plugin runs in its own
process or container and receives only its own secret, wiped from the entry
point.

## Decision

**Option C.** Three components, three interfaces:

```
Router    decide(envelope, catalogue, last_outcome) -> catalogue entry
Provider  call(envelope) -> response
Gateway   reached by address; holds every credential; publishes the catalogue
```

- The **router** is a plugin, named in configuration, loaded by the existing
  plugin loader into its own registry. It receives the catalogue of permitted
  targets as an argument and never queries the gateway itself. The
  `last_outcome` argument is **always passed and ignorable**: a keyword matcher need
  never read it. The core writes the chosen entry into the envelope's **empty**
  fields only, so a model the agent already named is kept, and the result is still
  checked against the catalogue.
- *Revised 2026-10-12, while still proposed (spec §14a B8/B9, engineer's report on
  R6):* the router returned an envelope in the first draft. Returning a catalogue
  entry means a router author learns only the catalogue, the core enforces "fill
  only empty fields" for every router, and the answer grows with the catalogue's
  element type (`endpoint` later). `last_outcome` was optional: it is now always
  passed, so increment 4 changes its value, not the arity.
- The **provider** makes no decision. It calls what the envelope names.
- The **gateway** holds every credential and runs out of the agent's process.
  **Its position already exists** — `GATEWAY_URL` plus a per-provider path
  prefix — and nginx occupies it today. A passthrough container carrying the
  existing nginx location blocks is the next occupant; LiteLLM, or anything
  else, a later one, behind the same address. Nothing else changes when the
  occupant does, which is the property the address buys.
- The **catalogue** is the allow-list of `(provider, endpoint, model)` the
  gateway can serve, with capability fields. Allow-list, never denylist. **Its
  carrier is not part of the decision.** In the first increment it is the
  existing per-provider configuration keys, enumerated at startup. A single file
  read by both the gateway and the core, carrying credentials by name only so it
  holds no secret, arrives when a router reasons about capabilities.
- **Increment 1 routes the model within the configured provider.** A pair that
  names any other provider is outside the catalogue and refused like any other
  (zero calls, the pair logged). Dispatching across providers is deferred: it
  needs a second provider instance behind the hook, and nothing varies across
  that seam yet.
- When no router is configured, a **null router** runs — not a bypass branch.
  One code path.
- The envelope and the catalogue are **plain serializable data**, so a router
  may later run in a container with no change to the interface.

## Why the others were rejected

**A — one shared contract.** `chat(request) -> LLMResponse` forces a router to
make the call, which puts credentials on its path. The router is the component
that third parties write and that gets swapped most often, so it should not
hold a credential. The contract also erases the distinction
between a component that decides and one that acts, so nothing in the type
system prevents a provider from routing or a router from calling an endpoint.

The argument *for* A was that a router must see responses to do overload
fallback and context-overflow recovery, as the production router does. That
argument does not survive inspection: that router fuses two concerns — choosing a model, and
recovering from failure. Separating them puts recovery in a bounded retry loop
in the core, which re-invokes the router with the failure attached. The optional
`last_outcome` argument buys the whole class of behaviour without a router ever
making a call.

**B — a tier flag interpreted by providers.** A tier on the shared envelope makes
every provider a decision-maker, which contradicts the design's own rule that
the provider decides nothing. It scales badly in the literal sense: each new
provider must independently implement a tier vocabulary, and nothing keeps their
interpretations consistent. It cannot express an exact model choice at all, so a
router that knows precisely which model it wants has no way to say so.
OMEGA-282's 2026-09-17 comment already rejected this option by name, because it
ties routing to one provider and needs a rebuild for every change of rules — and PR
#373 confirms the prediction, working only when the provider is OpenRouter.

**D — secrets inside plugins.** The objection it answers is real: a central
broker must be edited whenever a provider is added. But the remedy costs N
sandboxed processes with a secret-delivery mechanism each, and it moves the
trust boundary onto plugin authors, who are third parties. Today's property —
the agent process holds no credential, enforced by one audited component — is
the most finished thing in the system, and this would trade it for a property
only as strong as the least careful plugin.

The coupling objection is answered more cheaply by the catalogue: a plugin
declares the **name** of the credential it needs and the gateway resolves it, so
adding a provider edits one allow-list entry and no code.

## Consequences

**Made easy.** Writing a router: one function, no credentials, no network, no
lifecycle; testable as a pure function over two literals with no running stack.
Adding a provider or endpoint: one catalogue entry. Replacing the gateway:
change an address. Running a router in a container later: a transport change,
not an interface change.

**Made hard.** Routing decisions that genuinely require mid-call control —
streaming interception, token-level steering — are out of reach and would need a
new seam. Accepted: nothing proposed needs them.

**What must stay true.** The router never receives a credential; a router that
needs one is misdesigned, and that is checkable. The provider never branches on
a routing concept. The catalogue remains an allow-list. The gateway stays out of
the agent's process — if it is ever linked in, the credential-isolation property
is gone and this ADR is void.

**The check this owes.** The boundary worth machine-checking is *the agent
process cannot read any provider credential*. Per the architecture rule it is
not done until it has been seen to pass, then **fail on a deliberate
violation**, then pass again.

**The venue is confirmed.** `singnet/Omega`'s `build` workflow runs on every
pull request to `main` and calls `autotests`, whose MeTTa, pytest and Phase-1
mock suites are merge-blocking; run verdicts and per-job conclusions are
readable with this avatar's read-scoped token — **verified 2026-10-09.** So the
preferred shape applies: the rule is wired into the automation that will enforce
it, and the three-state proof is observed there.

**Two narrower checks replace the one named above, and this is a deliberate
change.** *"The agent process cannot read any provider credential"* is the
property that matters most, but it is a property of the **gateway**, which this
ADR does not change (see the premise above) — so a check on it would re-prove
something already true and would not bite on anything routing adds. The two
boundaries routing actually introduces are:

1. **Nothing under `routers/` reads a credential or makes a call** — no
   environment read, no HTTP client, no `py-call` into a provider.
2. **No provider branches on a routing concept** — the identifiers `tier`,
   `router`, `economy` and `premium` do not appear under `providers/` (excepting
   the shared `lib_llm_ext.py`, which carries the envelope's own fields).

Check 2 is the one that would have caught PR #373's shape automatically, which
is the argument for it. Both run in `tests/test_architecture_rules.py` via
`tests/pytest.sh`, inside the merge-blocking `autotests` job.

**The bite is still unproven** — neither check has been watched fail on a
deliberate violation — so the honest status is **rules authored, venue
confirmed, bite unproven**. The job to watch go red is named in the
specification: `build / autotests / autotests (linux/amd64, ubuntu-latest)`.

One caveat the venue's existence does not remove: the autotests job runs the
agent on the `TestMock` provider, so a green run says nothing about any
provider-specific path. A check must name the job expected to go red; relying on
the suite being green proves nothing.

**Deferred, with the question named.**

| Deferred | Owner |
|---|---|
| How the catalogue file reaches a sidecar — mounted, baked at build, or pushed | unassigned |
| Keeping gateway and core copies in step if they ever diverge into two files | unassigned |
| Per-agent token and cost accounting (arrives with LiteLLM, not with routing) | unassigned |
| Channel secrets — Telegram, Mattermost, Slack stay in nginx; out of scope here | unassigned |
| A `models.json` capability schema; no standard exists, so ours is local | this ADR's author |
| Advertised context exceeds effective context on every model benchmarked; a router escalating on the advertised figure escalates too late | unassigned |

## Changes since the first draft (2026-10-09, same day)

Recorded so the diff is readable without re-reading the whole ADR.

| Changed | Why |
|---|---|
| Premise: the cheap/premium **policy** was licensed, the **seam** was not | the 2026-09-22 budget/premium plan *was* answered on 09-23, by pointing back to the finished design. An earlier premise of mine recorded it as unanswered |
| Premise added: the envelope's four optional fields are already ratified | answered on OMEGA-282 on 2026-09-18, which I had missed |
| Premise added: the implementer restated this design correctly before diverging | removes "misread the design" as an explanation |
| Premise added: the gateway's **position** already exists (`GATEWAY_URL`) | read in the code; it shrinks the gateway work from "introduce a layer" to "replace an occupant" |
| Decision: gateway wording follows that premise | same |
| Consequences: venue **confirmed**; the owed check replaced by **two narrower ones** | the venue is readable, and the original check would not have bitten on anything routing adds |
| 2026-10-12: catalogue carrier wording follows the spec (config keys first, file later) | the Decision still said "a single configuration file" for the first iteration, contradicting the spec, R5 and the glossary. Raised by the engineer |
| 2026-10-12: increment 1 routes the model within the configured provider; another provider is refused | no issue said what happens when a router names a provider other than the configured one. Raised by the engineer; cross-provider dispatch deferred |
| 2026-10-12: a router returns **one catalogue entry**, and `last_outcome` is always passed (spec B8/B9) | the build showed a router choosing from the catalogue is simpler and checkable as one entry; an optional argument was a second code path. Revised in place while still proposed |
| 2026-10-12: published in `omega-specs`; the numbering premise is verified | the repository now exists and was empty |
| 2026-10-12: OMEGA-360 marked not yet reviewed, preliminarily adopted; PR #373 "partially satisfies" | operator review of PR #2; matches the posted #373 review |
| 2026-10-23: internal sources (Jira, team discussion, a private repository, the OMEGA-360 proposal) cited by date and summarized, not quoted; the private spec link replaced by where to ask | preparing the repository to be public; no decision, premise or rejection changed |

## Relationship to existing records

- **OMEGA-282** — the north star. This ADR implements its 2026-09-18 position
  and makes explicit what that comment left implicit: the catalogue, the null
  router, and the optional outcome argument.
- **OMEGA-360** — not yet reviewed. Preliminarily adopted and aligned with this
  ADR: its modularization direction and its separation of infrastructure from the
  agent core. It differs on two points, the shared contract (option A) and
  secrets inside plugins (option D).
- **OMEGA-376, increment 1** — design: the increment-1 specification, 2026-10-09 (internal; ask Kabir Veitas); issues: [index #9](https://github.com/snet-omega-factory/Omega/issues/9); code: [`omega-376-llm-routing-inc1`](https://github.com/snet-omega-factory/Omega/tree/omega-376-llm-routing-inc1).
- **PR #373** — partially satisfies this ADR. Its router is pure (no call, no
  credential), is chosen by configuration and decides before any provider runs;
  its keyword policy and `llmRequestContentsNormalized` are reused in increment 1.
  It diverges in returning a label that a provider interprets (option B), and a
  second router needs a change in core.
