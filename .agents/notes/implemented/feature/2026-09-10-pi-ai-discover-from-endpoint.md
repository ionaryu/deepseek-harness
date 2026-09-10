# Agent Note: Opting a route's model discovery into its endpoint

Status: implemented

English | [中文](2026-09-10-pi-ai-discover-from-endpoint.zh.md)

## Problem

"Fetch available models" answers a named route from the installed pi-ai catalog without a network call. That is the right default — pi-ai's registry is authoritative for its own providers and carries context windows and output caps a listing endpoint would not disclose — but it is also a snapshot: a provider that adds models after the installed pi-ai release never shows them on the Models page. Concrete case: the `opencode-go` route ships 27 installed entries while its real listing at `https://opencode.ai/zen/go/v1/models` advertises 36 models, and the documented workaround — re-declaring the same gateway as a custom provider to reach discovery's network path — duplicates the route to see what it really serves.

## Decision

A route may set `discoverFromEndpoint: true`. Its interrogation skips the installed-catalog answer and asks the endpoint instead: the draft's `baseURL` when the form shows one, the route's declared `baseURL` when the draft omits it, or — for a stored route on its catalog default — the endpoint the route's installed models of the listing protocol reach. The reply is enriched from the installed entries of the same ids: a listed id the catalog still describes inherits that entry's context window, output cap, and display name when the listing lacks one. A listed id the catalog does not describe cannot be materialized from installed defaults, so it carries the protocol and endpoint the interrogation itself used — an adopted row declares them per entry (`api` and `baseURL` on a `models` entry, winning over the route's own) — and serves through the route's capacity defaults for what no listing discloses. Catalog-only ids are never appended — the endpoint decides presence, which is the reason to ask it. Nothing is stored by the interrogation, and every route that does not opt in keeps the catalog answer unchanged.

## Alternatives considered

**Ask the endpoint for every named route.** Flipping the default makes every built-in provider's fetch a network call and downgrades the reply for providers whose listings disclose less than the registry. Rejected: the registry is the better answer where it is complete, and only a lagging catalog needs the wire.

**Keep the custom-provider workaround.** Re-declaring the gateway under a route key pi-ai does not ship reaches the network path today. Rejected as the product answer: it duplicates the route, splits one provider's identity across two keys, and hides the cause from the user.

**Endpoint reply without enrichment.** Every listed row would carry whatever the listing discloses; for a listing like opencode-go's (ids only), adopting any model means hand-typing the wire facts and capacities the installed catalog or the interrogation already records. Rejected: the registry stays the better source for what a row partially discloses.

**A provider-id rule for opencode routes.** Special-casing `opencode*` keys bakes one provider's endpoint convention into the adapter. Rejected: the lagging-catalog situation is generic, so the choice belongs to the route's configuration.

## Consequences

The Models page can surface a provider's live catalog without re-declaring it, and adopted rows keep real capacities where the catalog describes the id while unknown ids carry their own wire declaration and take the route's `defaultContextWindow`/`defaultMaxTokens` until the user fills them. The opt-in route's fetch costs one network call and reports endpoint failures as discovery errors, which is the point of asking. One configuration field, two per-entry fields, and their JSDoc join the generated configuration catalog; no stored format, wire shape, or session event changes.

## Testing

The discovery specs pin the opt-in interrogation against a local listing server (endpoint URL, stored-credential bearer, capacity enrichment, a completed row kept as sent, and an undescribed row carrying the interrogation's protocol and endpoint), the catalog-derived endpoint for a stored route that declares none (`https://opencode.ai/zen/go/v1/models`, through a stubbed fetch), the missing-protocol-family refusal for a catalog route, and the unchanged default catalog answer. The config spec pins the flag's explicit resolution, the catalog spec pins entry-level wire declarations on described and undescribed ids, the LLM runtime spec pins the normalized rows, and the provider-form spec pins the adopted row writing them through.
