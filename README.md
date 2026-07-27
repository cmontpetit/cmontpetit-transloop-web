# cmontpetit-transloop-web

Public website + support for the TransLoop iOS app, and the read-only model
catalog (`models.json`) the app fetches. Serving the manifest here lets new
models become selectable **without an App Store release** — edit
`docs/models.json`, push, done.

The TransLoop app repo is **private**; GitHub Pages on a private repo needs a
paid plan and is public anyway. The manifest content (model ids, names, tiers)
is not sensitive — it's already derivable from the shipped binary — so it lives
in this **separate public repo** while the app source stays private.

## One-time setup

1. Push the contents of this `manifest-site/` directory to the public repo
   `cmontpetit/cmontpetit-transloop-web` (so it has `docs/models.json`,
   `docs/index.html`, `docs/.nojekyll`, and `.github/workflows/pages.yml`).
2. In the repo: **Settings → Pages → Build and deployment → Source: GitHub
   Actions** (the included `pages.yml` validates `models.json` and deploys
   `docs/`). Alternatively use **Source: Deploy from a branch → main → /docs**.
3. Confirm the manifest is live at
   `https://cmontpetit.github.io/cmontpetit-transloop-web/models.json`.
4. The app is already pointed at that URL via the **`ModelManifestURL`** key in
   `ios/TransLoop/Info.plist`. Until Pages is live the fetch simply 404s and the
   app keeps its bundled catalog (safe/dormant). If you add a custom domain,
   update both that key and the URL above.

> This repo also doubles as the app's **website / support / privacy-policy**
> host — the App Store submission needs a Support URL and Privacy Policy URL,
> which can live here (e.g. `/support`, `/privacy`) alongside `models.json`.

## Adding or bumping a model

Edit `docs/models.json` and push. The app picks it up on next launch/foreground
(fetched read-only, cached, with the bundled copy as offline fallback).

Keep `docs/models.json` in sync with the app's bundled copy at
`ios/TransLoop/Assets.xcassets/ModelCatalog.dataset/models.json` and the
compiled fallback in `ios/TransLoop/Models/ModelCatalogEntry.swift`.
`scripts/run_model_catalog_checks.sh` in the app repo asserts all three agree,
so run it after editing any of them.

> ⚠️ **This directory is a scaffold that is mirrored by hand into the public
> repo.** The app repo's copy is the source of truth for `pages.yml` and this
> README; the public repo is what actually runs them. Changing the validator
> here does nothing until it is copied over — that has already drifted once.
> When you touch anything outside `docs/`, mirror it in the same sitting.

### Safety gates (enforced by the app)

- **`family` gate** — entries whose `family` the installed binary can't
  build/parse (anything other than `claude` / `openai` today) are ignored,
  never sent. Adding e.g. `claude-sonnet-6` under `family: "claude"` is safe
  because the existing Anthropic request shape can send it.
- **`schemaVersion` gate** — the app requires an *exact* match, so bumping this
  strands every already-installed binary on its bundled catalog with no signal.
  When an incompatible shape change is needed, publish it at a versioned path
  (`models.v2.json`) and leave `models.json` serving schema 1 for old clients.
- **Availability stays key-driven** — a manifest entry only becomes a usable
  provider when its API key is set/enabled in the app. The manifest controls
  *which models exist*, not *which are enabled*.
- **Endpoints and keys are never in the manifest** — the API URL, the key
  source, and the request builder are hardcoded per `family` in the app. A bad
  push can change which model id is sent, but not where the request goes.
- **Bounds** — at most 64 models; `displayName`/`shortName` are truncated to 40
  characters. The CI validator rejects a manifest that exceeds either.

> ⚠️ **The `family` gate covers request *shape*, not model *capability*.** It
> guarantees the app knows how to build an Anthropic-shaped request; it does not
> know whether that particular model accepts every optional parameter the shape
> carries. Before publishing a new model, run `scripts/probe_reviewer.sh` in the
> app repo against its id — it sends the exact reviewer request, with and
> without structured output, so a 400 shows up before users see it. If the model
> rejects one of the optional parameters, turn it off with the capability flags
> below rather than skipping the model.

### Entry shape

```json
{
  "id": "claude-sonnet-5",          // wire model id sent to the provider
  "providerId": "claude_sonnet",    // stable storage key — keep constant across id bumps
  "displayName": "Claude Sonnet",
  "shortName": "Sonnet",
  "icon": { "type": "system", "value": "brain.head.profile" },  // or {"type":"asset","value":"OpenAIProvider"}
  "family": "claude",               // claude | openai  (selects the request builder)
  "externalFamily": "anthropic",    // anthropic | openai  (billing/routing)
  "roles": ["translator", "reviewer"],   // reviewer-only models omit "translator"
  "tier": "standard"                // economy | standard | premium
}
```

`tier` drives reviewer defaults: the highest-tier reviewer per external provider
is enabled by default; lower tiers are economy opt-ins.

#### Optional fields

| Field | Default | Meaning |
|---|---|---|
| `reasoningEffort` | omitted | OpenAI reasoning models only (`gpt-5` family): sends `reasoning_effort` (`minimal`\|`low`\|`medium`\|`high`). Leave it out on non-reasoning models — they reject the parameter. |
| `supportsStructuredOutput` | `true` | Set `false` if the model rejects a schema-constrained response (Anthropic `output_config.format`, OpenAI strict `response_format`). The reviewer then falls back to tolerant JSON parsing instead of 400ing. |
| `supportsDisabledThinking` | `true` | Anthropic only. Set `false` if the model rejects `thinking: {"type": "disabled"}` — Claude Fable 5 rejects it outright, and Claude Opus 5 rejects it above `high` effort. |

Both capability flags exist so a model that needs one of these parameters turned
off can still ship via the manifest, without an App Store release.

## Validation

`.github/workflows/pages.yml` validates **both** `models.json` and
`models-dev.json` before deploying — required keys, enum vocabularies
(`family` / `externalFamily` / `tier` / `roles` / `icon.type`), duplicate
`providerId`s, name lengths, model count, and at least one reviewer. This
matters because the app drops an entry with an unrecognized enum value
*silently*: a typo'd `tier` would otherwise deploy green and quietly remove that
model from every device on the next fetch.

`docs/models-dev.json` is what **DEBUG** builds prefer (via the
`ModelManifestURLDev` Info.plist key), so experimental model config can be tried
there without touching production. Keep it valid — it is deployed from the same
push.
