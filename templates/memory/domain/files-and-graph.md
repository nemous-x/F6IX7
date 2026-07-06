# Domain file formats

`related-files.json` — split by technical layer so agents load only their layer:

```json
{
  "updated": "{{date}}",
  "backend": [ { "path": "", "role": "service | controller | entity | migration", "note": "" } ],
  "web":     [ { "path": "", "role": "page | component | hook | store", "note": "" } ],
  "mobile":  [ { "path": "", "role": "screen | component | hook | service", "note": "" } ],
  "shared":  [ { "path": "", "role": "type | validator | util", "note": "" } ]
}
```

Omit layers the project doesn't have.

`graph.json`: follows `schemas/graph.schema.json`, scoped to this domain's nodes and their one-hop edges.

Layer files (`backend.md`, `web.md`, `mobile.md`) and the rest (`tests.md`, `history.md`, `known-issues.md`, `decisions.md`) follow the same shape as overview.md: curated marker on line 1, dense factual bullet sections, `## Learned` heading at the end, hard cap ~150 lines. History entries are dated one-liners, newest first:

```markdown
## {{date}} — {{task/feature title}} (artifact {{NNN}})
- What changed in this domain, one line, in the project's own terms.
```
