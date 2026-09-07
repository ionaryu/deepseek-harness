# Agent Note: Dropping stored models the catalog no longer describes

Status: implemented

English | [中文](2026-09-07-catalog-drift-model-drop-tolerance.zh.md)

## Problem

A stored `providers.<route>.models` list under the `llm-pi-ai` settings namespace is a snapshot of the pi-ai catalog it was taken against. When the installed catalog changes between pi-ai releases — real case: the opencode-go catalog went 19 models (including `grok-4.5`) at 0.84.2 to 25 models at 0.84.4, removing `grok-4.5` and adding `glm-5.3-flash` — a previously valid stored list becomes unserviceable. `resolveRouteModels` threw for the first entry the installed catalog no longer described (`model "X" needs an api` or `needs a baseURL`; the installed catalog does not describe it), so `resolveProfiles`, and through it the settings namespace validator `assertServiceable`, propagated the error.

The failure surfaced at the worst point. The settings seam registers each namespace by resolving the stored section through the owner's validator; with no last good value yet, a stored section that fails rejects the registration itself. The whole `dsh-llm-pi-ai` plugin therefore failed to activate at boot: the web Models page lost every pi-ai provider row (only the deepseek-official family remained) and its Add controls stopped working. One stale model id disabled an entire provider family, and the failure recurs on every pi-ai catalog change.

## Decision

A stored model entry that cannot be materialized **only** because the installed catalog no longer describes it no longer fails its route. The cause is narrow and structural: the route's catalog currently ships at least one model, the entry is not among them, the route sets neither `api` nor `baseURL` (a stored snapshot never touched them), and the route-level fallbacks (`routeApi`, the provider `baseUrl`) cannot fill the wire protocol or endpoint either. `resolveRouteModels` records such entries on the returned `RouteCatalog.dropped` (each with its id and reason) and serves every remaining model, mirroring how `discovery.ts` `readListing` skips an unusable listing row instead of failing the whole interrogation. The skip is never silent: the resolved profile carries `droppedModels`, and `dsh-llm-pi-ai`'s apply logs one warning per changed configuration naming the provider, the dropped ids, and the remediation ("re-fetch models on the Models page or upgrade pi-ai"). The stored `settings.yaml` line is left untouched — nothing deletes it from disk — so an id a future pi-ai release describes again returns automatically on the next resolve.

Every other refusal is unchanged. Empty and duplicate ids, `modelOverrides` that land nowhere (including naming a model the catalog does not describe), `modelOverrides` beside a `models` list, an empty `defaultInput`, invalid headers and capacities, compat fields a protocol does not take, and reasoning-effort errors all throw exactly as before. A route whose configured list drops **every** entry still refuses with a message naming the stale ids: a route with nothing to serve is a configuration error, never a silent empty route. The drop path is also closed to routes the catalog does not ship at all — an underspecified hand-declared entry keeps its original `needs an api` / `needs a baseURL` refusal, because there is no snapshot to be stale.

Writes name the same cause and are now tolerated too. The settings seam runs one validator over loads (registration and external publishes) and writes (`update` / `replace` / `mutate`) alike, and resolution happens through that single hook, so a load-tolerant validator cannot also refuse the write inside the current seam without changing `dsh-settings`. Uniform tolerance for this cause alone is accepted and documented in `assertServiceable`'s JSDoc. The exposure is bounded: the Models UI adopts only discovery results, which are always describable by the installed catalog, so a genuine typo on a catalog route is unreachable through the page; a hand-declared route carries `api` and `baseURL`, making every entry describable by construction; and `modelOverrides` remains a hard refusal.

## Alternatives considered

**Keep refusing and let registration fail.** The status quo leaves the plugin dead at boot on every pi-ai catalog change — the incident. Rejected as the failure this change exists to remove.

**Rewrite the stored section.** Prune the stale ids out of `settings.yaml` on load, or refuse-to-tolerate and force a re-fetch. Both delete or mutate user-owned data without consent and would fight the file watcher and revision accounting; neither lets a removed id come back when a later catalog describes it again. Rejected — the stored line is never touched.

**Load-tolerant, strict writes.** The preferred split: tolerate the drift cause when reading a stored section, keep refusing the same content when a caller writes it. It is impossible inside the current settings seam, whose `register`, `publish`, and write paths all resolve through one `validate` hook, and splitting them would mean changing `dsh-settings` for one consumer's cause. Rejected for scope; the trade-off is recorded here and in the JSDoc.

**Drop silently.** Serve the remaining models with no log and no `droppedModels` record. Rejected — a drop an operator cannot see would resurrect the silent-disable failure it replaces, only smaller. The all-dropped refusal also keeps "stale snapshot" from quietly meaning "serve nothing".

**Tolerate the cause on any route.** Drop undescribed entries regardless of whether the route's catalog still ships models. An underspecified entry on a route pi-ai does not ship is a configuration error being written now, not a stale snapshot, and its refusal message is what tells the author to add `api`/`baseURL`. Rejected — restricting the drop to routes whose catalog still ships models preserves that diagnostic and the hand-declared-route tests that pin it.

## Consequences

The plugin mounts and keeps serving when a stored section names ids the installed catalog no longer describes: the boot no longer dies, the Models page keeps the provider's remaining rows and working Add controls, and a warning names what was dropped and how to restore it. The settings document is untouched, so adopting a newer catalog (or reverting pi-ai) restores dropped ids on the next resolve with no edit.

The tolerated-write trade-off is the one behavioral relaxation. A settings write naming a stale model id is stored instead of refused (the previous behavior), because the seam cannot judge loads and writes separately; the earlier reasoning — the Models UI can only write what discovery returns, and discovery always returns catalog-describable models — is what bounds the exposure. All other refusals, including the all-dropped route, behave as before. Because the drop is logged once per changed configuration inside the memoized profile resolution, the memoization identity the adapter's snapshots rely on is untouched: registration facts and directory facts still derive from the same subset of each profile.

## Testing

The package spec suite pins the semantics with catalog-drift cases: a stored list naming one stale id plus a live model serves only the live model and reports the dropped id; a list whose every entry is stale refuses, naming the ids; a route that declares transport for an undescribed id still refuses its missing protocol; a stored document with a stale id boots the plugin, serves the surviving model, and logs the drop once; and a settings write naming a stale id is accepted with the stored line untouched. The two pre-existing specs that pinned `xai`'s mixed-protocol catalog now select whichever installed route satisfies their premise, so catalog churn cannot break them again. The full `dsh-llm-pi-ai` suite passes on the installed pi-ai release.
