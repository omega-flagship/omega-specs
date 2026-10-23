# LLM routing

The path an Omega agent's LLM call takes, from the loop that builds a request to
the endpoint that answers it. This vocabulary exists because three of its terms
have been used interchangeably by different authors in the same month, and each
confusion produced a different design.

Governed by [ADR-0001](../adr/ADR-0001-llm-routing-three-components.md).

## Language

**Envelope**:
Everything a single LLM call needs: messages, tools, limits, metadata, and —
once decided — the provider, endpoint and model. Passed down the path and
completed as it goes.
_Avoid_: request, payload, `LLMRequest` (the Python class is one representation
of an envelope, not the term)

**Router**:
The component that decides which provider, endpoint and model an envelope will
use, by choosing one entry of the catalogue. The core writes that entry into
the envelope's empty fields. It makes no call and holds no credential. A plugin: third parties write them and they are swapped often.
_Avoid_: model selector, dispatcher, LLM router (redundant in this context),
hook

**Provider**:
The component that makes the call to an endpoint. It makes no decision; it calls
exactly what the envelope names.
_Avoid_: backend, client, adapter, driver

**Gateway**:
The out-of-process component that holds every credential, injects it into the
outbound call, and publishes the catalogue. Addressed by configuration, never
loaded as a plugin. Swappable — nginx today, a passthrough container next,
LiteLLM later — without anything else changing.
_Avoid_: proxy, broker, credential layer, LiteLLM (an implementation, not the
term), passthrough (a stage, not the term)

**Catalogue**:
The allow-list of `(provider, endpoint, model)` combinations the gateway can
serve, with each entry's capabilities and the **name** of the credential it
needs. Holds no secret value. A router may choose only from it.
**The carrier is not part of the term.** Its first form is the existing
per-provider configuration keys, enumerated at startup, and in the first
increment it holds only the configured provider's models. A dedicated file with
capability fields comes later. A design that says "the catalogue is deferred"
means the file, never the allow-list — the allow-list holds from the first
increment.
_Avoid_: model list, registry (the plugin registry is a different thing),
manifest, permitted set (same thing under a second name)

**Endpoint**:
The address a call is actually sent to. Distinct from the provider, because one
provider kind may serve several endpoints — a self-hosted OpenAI-compatible
model being the ordinary case.
_Avoid_: URL, host, upstream

**Null router**:
The router that runs when none is configured: it completes the envelope from
configuration and nothing else. Exists so that "routing off" is a configuration
rather than a branch around the router.
_Avoid_: passthrough router, no-op router, bypass

**Hook**:
The point in the core where the loop hands every envelope to the router, then
checks the result against the catalogue before any provider is called. It has no
branch: when the agent already named a model, the router is still called, and a
router fills only empty fields (amended 2026-10-12, QA O6.3). A location, not a
component.
_Avoid_: middleware, interceptor

## Terms deliberately not defined here

**Tier** — a coarse class of model such as "economy" or "premium". It has no
place in this vocabulary because the shared envelope carries an exact provider,
endpoint and model. A router may use tiers in its own internal logic; nothing
outside that router should ever see one.
