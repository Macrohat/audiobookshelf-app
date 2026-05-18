# Macrohat Audiobookshelf-App Fork

This is a Macrohat-flavoured fork of [advplyr/audiobookshelf-app](https://github.com/advplyr/audiobookshelf-app)
maintained for the sole purpose of building TestFlight binaries when the official Apple
TestFlight beta is full (10k Apple-imposed cap).

Upstream is AGPL-3.0; this fork is too. Source is published; nothing proprietary is added.

## What's different from upstream

All Macrohat-specific changes live on the `macrohat` branch.

| Area | Upstream | Macrohat fork |
|---|---|---|
| Bundle ID (Release) | `com.audiobookshelf.app` | `co.macrohat.audiobookshelf` |
| Bundle ID (Debug) | `com.audiobookshelf.app.dev` | `co.macrohat.audiobookshelf.dev` |
| Bundle ID (Tests) | `com.audiobookshelf.AudiobookshelfUnitTests` | `co.macrohat.audiobookshelf.tests` |
| Capacitor `appId` | `com.audiobookshelf.app` | `co.macrohat.audiobookshelf` |
| URL scheme name | `com.audiobookshelf.app` | `co.macrohat.audiobookshelf` |
| `DEVELOPMENT_TEAM` in pbxproj | `7UFJ7D8V6A` | injected at build time via `fastlane gym --xcargs DEVELOPMENT_TEAM=…` (pbxproj retained for parity with upstream merges) |
| CI | none | `.github/workflows/ios-testflight.yml` |
| fastlane | none | `ios/App/fastlane/` (Fastfile, Appfile, Matchfile) |

## Build pipeline

GitHub Actions, `macos-14` runner. Public repo → unlimited free runner minutes.

```
push to macrohat / workflow_dispatch
  → checkout
  → Xcode 15.4, Node 20, Ruby 3.3 (bundler cache)
  → npm ci && npm run generate (nuxt → static dist/)
  → npx cap sync ios
  → bundle exec pod install
  → bundle exec fastlane beta
       → app_store_connect_api_key (API key auth, no interactive login)
       → match (pulls or generates Apple Distribution cert + provisioning profile
         from Macrohat/ios-signing, encrypted with MATCH_PASSWORD)
       → latest_testflight_build_number + 1
       → build_app (Release config, manual signing, team injected via xcargs)
       → upload_to_testflight (skip wait, sets changelog)
```

## Required GitHub Secrets

All set under [Macrohat/audiobookshelf-app → Settings → Secrets and variables → Actions](https://github.com/Macrohat/audiobookshelf-app/settings/secrets/actions).
**Never paste secret values into commits, PRs, or chat — they live exclusively in Actions Secrets.**

| Secret | What it is | Where it comes from |
|---|---|---|
| `APP_STORE_CONNECT_KEY_ID` | 10-char alphanumeric key ID | App Store Connect → Users and Access → Integrations → Team Keys |
| `APP_STORE_CONNECT_ISSUER_ID` | UUID at the top of the Integrations page | same page |
| `APP_STORE_CONNECT_KEY_CONTENT_B64` | base64 of the downloaded `.p8` file | `base64 -i AuthKey_XXXXXXXXXX.p8` (you keep the `.p8` offline, only base64 form lives in Actions) |
| `APPLE_TEAM_ID` | 10-char Apple Developer team id | Apple Developer → Membership |
| `MATCH_PASSWORD` | passphrase used to AES-encrypt certs in the signing repo | invent a strong one, store in Vaultwarden |
| `MATCH_GIT_BASIC_AUTHORIZATION` | base64 of `user:pat` for read-write access to `Macrohat/ios-signing` | `printf 'alexagha:<GITHUB_PAT>' \| base64` — use a PAT with `repo` scope |

The App Store Connect key needs **App Manager** role (or higher) for TestFlight upload and provisioning.

## Storage repo for signing assets

`Macrohat/ios-signing` (private) — fastlane match writes its encrypted blobs there. On first CI run match will:

1. Discover the Distribution cert and provisioning profile don't exist
2. Generate both via App Store Connect API
3. AES-encrypt them with `MATCH_PASSWORD`
4. Commit to the signing repo

You don't initialise it manually. Just create the empty private repo.

## Apple-side prerequisites (one-time)

These can only be done by a human with Apple Developer portal access:

1. **App Store Connect API key** — Users and Access → Integrations → "Generate API Key", role = App Manager. Download the `.p8` (one chance), note the Key ID and Issuer ID.
2. **App ID** — Developer portal → Identifiers → "+" → App IDs → App. Bundle ID: `co.macrohat.audiobookshelf`. Capabilities: none required for the base app (no push, no Sign in with Apple, no associated domains).
3. **App Store Connect app record** — Apps → "+" → New App. Bundle ID: `co.macrohat.audiobookshelf`. SKU: any unique string e.g. `abs-macrohat`. Primary language: English. Name: must be globally unique — try `Audiobookshelf MH` or `ABS Macrohat`.

Steps 2 and 3 can alternatively be automated with `fastlane produce`, but doing them in the UI once is faster than wiring the produce lane.

## Updating from upstream

```bash
git checkout master
git pull upstream master
git push origin master
git checkout macrohat
git rebase master
git push --force-with-lease origin macrohat
```

Pushing `macrohat` triggers a new TestFlight build.

## Manual build trigger

GitHub UI: Actions → "iOS TestFlight (Macrohat)" → Run workflow → branch `macrohat` → optional changelog message.

## Vault notes

This repo is a *fork-and-build adapter*, not a Macrohat product, so it does not adopt the full
[REPO_LAYOUT.md](file:///C:/_Dev/_Vault/STANDARDS/REPO_LAYOUT.md) (no PRD.md / TDD.md / STATUS.yaml).
The relevant standing rule applied is "GitHub is only for audited/promoted code" — this is a public
fork of public AGPL code, no secrets cross the repo boundary; secrets are confined to GitHub Actions Secrets.
