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

### Safety gates (enforced by the app)

- **`family` gate** — entries whose `family` the installed binary can't
  build/parse (anything other than `claude` / `openai` today) are ignored,
  never sent. Adding e.g. `claude-sonnet-6` under `family: "claude"` is safe
  because the existing Anthropic request shape can send it.
- **`schemaVersion` gate** — bump `schemaVersion` only for an incompatible
  shape change; older binaries that don't recognize the version keep their
  bundled catalog.
- **Availability stays key-driven** — a manifest entry only becomes a usable
  provider when its API key is set/enabled in the app. The manifest controls
  *which models exist*, not *which are enabled*.

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
