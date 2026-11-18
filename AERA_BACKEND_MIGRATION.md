# Aera Launcher — backend migration plan

This document records the backend-focused state of the `aera/rebrand-launcher` working branch. It is intentionally separate from the UI redesign. The goal is to retain the useful Minecraft launcher engine and its feature set while removing runtime dependence on Millida-owned services before any public Aera release.

## Current baseline

The existing application is a substantial Tauri 2 + Rust + React/TypeScript launcher. The backend is not a blank scaffold and should not be rewritten just to change its identity.

Keep and regression-test the existing engine unless a concrete incompatibility is found:

- Minecraft version metadata, game files, assets, libraries and Java runtime installation.
- Fabric, Quilt, Forge and NeoForge installation and launch preparation.
- Profiles/instances, settings, import/export, backups and game-directory management.
- Modrinth and CurseForge discovery/download flows where they use the providers' public services.
- Microsoft device-code sign-in and the Xbox/Minecraft Services authentication chain.
- Local account-token vaulting; access tokens must remain in the Rust core and must never be returned to the webview.
- Game process lifecycle, launch progress, logs, crash reporting, performance settings and platform-specific launch support.
- Skins/capes, cloud/profile/social features, party voice and updater only after their service dependencies are migrated or explicitly made unavailable.

## Confirmed Millida-specific dependencies

The following are not just visual branding. They are runtime or persistent-identity dependencies and must be handled as separate migrations.

| Area | Current evidence | Required Aera action |
| --- | --- | --- |
| Microsoft OAuth app | `src-tauri/src/engine/accounts/microsoft.rs` reads `MILLIDA_MS_CLIENT_ID` and contains an upstream fallback client ID. | Register an Aera-owned Microsoft Entra application; use its client ID from build configuration. Remove the upstream fallback. Do not invent a client ID. |
| Millida API / account | `millida_api`, `millida_api_auth`, and the account module under `engine/accounts` are used by Millida-specific account/session features. | Map each command to an Aera service or a supported public provider. Keep the feature unavailable until a real Aera endpoint and auth contract exist. |
| Content mirror | `src-tauri/src/engine/core/mirror.rs` routes selected Modrinth, CurseForge, Mojang, loader and Java traffic through `api.millida.net` when direct access fails. | Preserve direct provider access. Replace the mirror only after an Aera-operated service exists; otherwise report direct-download failure rather than silently calling Millida. |
| Updater | `src-tauri/tauri.conf.json` and `engine/core/selfupdate.rs` use Millida update manifests, downloads and a Millida minisign public key. | Disable release updater artifacts until Aera has its own feed, signing key and release pipeline. Never reuse the upstream signing key or claim that an Aera package is independently updateable. |
| Party voice | `src-tauri/src/engine/party_voice.rs` requests tickets from the Millida API and connects to the Millida relay; `src-tauri/voice` is the voice client. | Keep the local voice engine isolated. Enable network voice only after an Aera relay, ticket API and protocol ownership/compatibility are established; otherwise show it as unavailable. |
| Data directory / app identity | Tauri identifier, deep-link scheme and `engine/core/paths.rs` still use `net.millida.launcher` / `millida://`. | Design a one-time, tested data migration before changing the identifier or data directory. Preserve existing profiles, worlds, settings and encrypted vault data. Do not simply switch paths and make user data appear lost. |
| Build identity | Cargo package/crate names and build-time `MILLIDA_VERSION` still contain upstream naming. | Rename only as a coordinated Rust/Cargo/CI change with a regenerated lockfile and build verification. Keep version stamping and frontend-ready build gates intact. |

## Non-negotiable engineering rules

1. Do not replace the Rust engine with frontend mocks or duplicate Minecraft logic in TypeScript.
2. Keep the Tauri IPC command names, argument shapes, return types and event contracts stable while the new UI is developed. If a contract must change, update Rust, TypeScript wrappers and tests together.
3. Keep secrets in the Rust core. Never expose Microsoft access/refresh tokens, API credentials, relay tickets or signing material to the webview.
4. Never ship an Aera build that calls Millida-owned APIs, mirrors, cosmetics, voice relay or updater endpoints as if they were Aera services.
5. Do not invent Aera domains, endpoints, OAuth client IDs, Discord application IDs, updater keys or credentials. Use explicit configuration supplied by Aera; otherwise gate the dependent feature as unavailable.
6. Keep upstream GPL-3.0-only, NOTICE and third-party attribution obligations. Do not remove upstream copyright or trademark notices to make the fork appear independently authored.
7. Do not change the visual UI in this backend workstream. Aera's redesigned UI is a separate frontend task.

## Recommended implementation order

### Phase 1 — Establish a safe backend baseline
- Keep this branch based on the current Aera-branded working branch.
- Inventory every Millida host, scheme, environment variable, app identifier, persistent path, credential, updater key and service-specific command.
- Classify each reference as: attribution/legal notice, compatibility/migration, public third-party provider, or Millida-owned runtime dependency.
- Record current IPC contracts and run the existing frontend/Rust test suites before behavior changes.

### Phase 2 — Separate providers from the engine
- Put service-specific endpoints and feature availability behind typed Rust configuration/provider boundaries.
- Keep Mojang, Microsoft, Modrinth, CurseForge, Fabric, Quilt, Forge, NeoForge and Java provider logic separate from Millida-only services.
- Make unavailable optional services return a clear typed/structured unavailable result; do not fake success or silently fall back to Millida.

### Phase 3 — Migrate identity and authentication
- Obtain Aera's own Microsoft Entra app registration and configure it without a hard-coded upstream fallback.
- Choose the Aera application identifier, deep-link scheme and data directory.
- Implement and test a one-time migration from the existing Millida-namespaced data folder before switching paths. Include a backup/rollback path and verify encrypted vault behavior.

### Phase 4 — Migrate optional network services
- Provision Aera-owned API, mirror, catalog/cosmetics, voice relay and updater services only where the product actually needs them.
- Define authentication, rate limits, privacy/data retention, error behavior and protocol contracts for each service.
- Enable each feature only after integration tests pass against Aera infrastructure.

### Phase 5 — Verify release readiness
- Run `bun run typecheck`, `bun run test`, `bun run build:web`.
- From `src-tauri`, run `cargo test` and `cargo clippy --all-targets -- -D warnings`.
- Build the intended platform package and inspect its product name, bundle identifier, deep-link registration, data paths, network destinations, updater configuration and installer metadata.
- Confirm a clean install and an upgrade/migration both preserve profiles and game data.

## Current release status

The Aera display name on this branch is not proof of a complete backend migration. The updater, application identifier, data directory, Microsoft client fallback, API/mirror integrations and party voice still contain upstream dependencies. Treat this branch as development-only until those items are resolved and verified.
