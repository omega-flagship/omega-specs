# omega-specs

The durable decisions for SingularityNET's Omega: **architecture decision records (ADRs)** and the **glossary** they're argued in. Tickets, specs and pull requests come and go; what's recorded here lasts.

## Decisions

| ID | Title | Status |
|---|---|---|
| [ADR-0001](adr/ADR-0001-llm-routing-three-components.md) | LLM routing: router, gateway and provider are three components | proposed |

## Glossary

- [LLM routing](glossary/llm-routing.md): envelope, router, provider, gateway, catalogue, endpoint, null router, hook.

## Where the work is

- **OMEGA-376**, LLM routing, increment 1:
  - tickets [#2–#8](https://github.com/snet-omega-factory/Omega/issues/9) on the factory fork;
  - prepared code [PR #1](https://github.com/snet-omega-factory/Omega/pull/1);
  - review of [singnet/Omega#373](https://github.com/singnet/Omega/pull/373);
  - how the team worked: [Discussion #10](https://github.com/snet-omega-factory/Omega/discussions/10).

## How a decision lands here

- An ADR gets a stable ID (`ADR-NNNN`) that never changes. Links use the ID, never a heading or a line number.
- Every ADR states its premises, with how and when each was verified, and the alternatives it rejected.
- Statuses: `proposed` → `accepted`. An accepted ADR changes only by an amendment, which is a re-ratification, or by a new ADR that supersedes it.
- Changes arrive by pull request. **A human merges.** ADRs are drafted by `snet-omega-software-architect`.

## Licence

Apache-2.0 (`LICENSE`).
