# Stack skill: Feature-sliced architecture

Injected for greenfield frontend-heavy work when feature-sliced structure is chosen (the alternative for domain-heavy backends is `ddd-cqrs`).

- Layers, top to bottom: `app` → `pages` → `widgets` → `features` → `entities` → `shared`. Imports only point downward; a layer never imports from itself sideways except via its public API.
- Every slice exposes a public API (`index.ts`); nothing deep-imports another slice's internals.
- `entities` hold business objects (model + minimal UI); `features` hold user interactions delivering value (one verb: add-to-cart, filter-orders); `widgets` compose them into page blocks.
- `shared` is domain-agnostic only (ui-kit, lib, api client, config). If it knows a business term, it belongs in `entities`.
- Name slices with the ubiquitous language from F67 domain memory — slices should map cleanly onto F67 domains for graph tracking.
- Cross-feature communication goes through the layer above (widget/page) or events — never feature→feature imports.
- Start minimal: create layers when the second use case appears, not speculatively.
- Record the adoption and layer conventions as a decision record and in `memory/global/architecture.md`.
