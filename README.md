# workspec-schemas

The canonical **WorkSpec artifact schema registry**, published at
[`https://schema.workspec.io/`](https://schema.workspec.io/).

The WorkSpec artifact schemas are the shared substrate of the WorkSpec product
family — [Decision Studio](https://github.com/FieldstateNZ/workspec-decision-studio)
is their first consumer, not their owner. This repo owns the published schema
URLs; product repos contribute schemas to it.

## URL contract

Published schema URLs are **stable and must not change** — they are baked into
npm package `$id`s and into the `$schema` directive of every WorkSpec artifact:

| Artifact | File suffix       | Schema URL                                                |
| -------- | ----------------- | --------------------------------------------------------- |
| Decision | `*.decision.yaml` | `https://schema.workspec.io/v1alpha1/decision.schema.json` |
| Catalog  | `*.catalog.yaml`  | `https://schema.workspec.io/v1alpha1/catalog.schema.json`  |

Artifacts bind to their schema (for editor IntelliSense, completion, and hover
docs) with a `yaml-language-server` directive on the first line:

```yaml
# yaml-language-server: $schema=https://schema.workspec.io/v1alpha1/decision.schema.json
```

## Repository layout

```
index.html                      # root index listing the artifact family
v1alpha1/
  decision.schema.json          # JSON Schema (draft 2020-12) for Decision artifacts
  catalog.schema.json           # JSON Schema (draft 2020-12) for Catalog artifacts
```

Each schema's `$id` is its published URL. Future artifact kinds land in the
same flat version namespace (`v1alpha1/<kind>.schema.json`).

## Versioning policy

One flat `v1alpha1/` namespace: **all artifact kinds version together as one
model.** A breaking change to the artifact model introduces a new namespace
(e.g. `v1beta1/`) alongside the old one; published versions are never mutated
incompatibly or removed.

## Publishing

Every push to `main` deploys the repo content to GitHub Pages via
[`.github/workflows/pages.yml`](.github/workflows/pages.yml), which sets the
`schema.workspec.io` CNAME on each deploy.

## How product repos contribute schemas

The schemas are **generated, not hand-edited**. The source of truth is the Zod
model in `@workspec/decision-schema` (in the
[`workspec-decision-studio`](https://github.com/FieldstateNZ/workspec-decision-studio)
monorepo), which commits the generated JSON Schemas under its `json-schema/`
directory.

Sync contract (v1 — simplest thing): the monorepo's **release process copies
the regenerated schemas here via PR**, byte-identical to the generated files.
A push-on-release automation (deploy key or fine-grained PAT from the
monorepo's release workflow) is a planned follow-up.

Do not edit files under `v1alpha1/` by hand — change the Zod source in the
owning product repo and sync the regenerated output.

## Domain cutover runbook (one-time)

The custom domain moves here from `workspec-decision-studio`'s Pages. The
published URLs must not change — only what serves them. Sequence (a brief gap
is acceptable; no external consumers yet):

1. Run the [Verify live schemas match repo](.github/workflows/verify-live.yml)
   workflow (manual dispatch) — it fails unless this repo's copies are
   byte-identical to what `https://schema.workspec.io` currently serves.
2. Remove the custom domain from `workspec-decision-studio`'s Pages settings
   (and make its pages workflow stop setting the CNAME / serving schemas).
3. Point the domain at this repo's Pages:
   `gh api -X PUT repos/FieldstateNZ/workspec-schemas/pages -f cname=schema.workspec.io`,
   then enable `https_enforced` once the certificate re-mints.
4. Verify: `curl -I https://schema.workspec.io/v1alpha1/decision.schema.json`
   returns 200 and the body is byte-identical to the npm-published copy.

DNS already points `schema.workspec.io → fieldstatenz.github.io`; no DNS work
is required.
