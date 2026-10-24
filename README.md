# smoozy-windows-ci

CI/CD pipeline for the **smoozy Windows Tauri app**.

The actual application source lives in a **private GitLab repository**:
<https://gitlab.com/smoozy-app/smoozy-windows>

This GitHub repository contains **only the GitHub Actions workflows**. It has no application code. Its purpose is purely to give us unlimited free Actions minutes (public GitHub repos get unlimited CI minutes) while the source code stays private on GitLab.

## How it works

```
┌────────────────────────────────┐     ┌─────────────────────────────┐
│ GitLab (private)               │     │ GitHub (this repo, public)  │
│ gitlab.com/smoozy-app/         │     │ Smoozy-LLC/smoozy-windows-ci│
│   smoozy-windows               │     │                             │
│                                │     │   .github/workflows/        │
│ - all source code              │     │   release.yml               │
│ - all commits (incl. version   │     │   ci.yml                    │
│   bumps pushed back by CI)     │     │                             │
│ - issues, MRs, code review     │     │   (no application code)     │
└──────────┬─────────────────────┘     └──────────┬──────────────────┘
           │                                      │
           │  git clone with GITLAB_TOKEN         │  workflow_dispatch
           └──────────────────────────────────────┘  triggers
                                │
                                ▼
                    ┌───────────────────────┐
                    │ windows-latest runner │
                    │  1. clone GitLab      │
                    │  2. pnpm install      │
                    │  3. pnpm tauri:build  │
                    │  4. upload artifacts  │
                    │  5. publish latest.json│
                    │  6. git push bump →   │
                    │     GitLab            │
                    └───────────────────────┘
```

## Triggering a release

1. GitHub → **Actions** tab → **Release Windows** workflow → **Run workflow**
2. Choose `bump: patch` or `minor`
3. Click **Run workflow**
4. ~15–25 minutes later: new `.exe` signed and published, Tauri updater picks it up on next user app launch

## Required secrets

Add these in GitHub → Settings → Secrets and variables → Actions:

| Secret | Purpose | Where to get |
|---|---|---|
| `GITLAB_TOKEN` | Clone + push back to GitLab | GitLab → User Settings → Access Tokens (scopes: `api`, `read_repository`, `write_repository`) |
| `GITLAB_DEPLOY_TOKEN_USERNAME` | Upload artifacts to Package Registry | GitLab → Project → Settings → Repository → Deploy tokens (scopes: `read_package_registry`, `write_package_registry`) |
| `GITLAB_DEPLOY_TOKEN_PASSWORD` | Same deploy token, token value | Same |
| `TAURI_SIGNING_PRIVATE_KEY` | Sign updater artifacts (minisign) | Existing key from monorepo — keep identical, DO NOT regenerate (would break auto-update for existing users) |
| `TAURI_SIGNING_PRIVATE_KEY_PASSWORD` | Passphrase for the key | Existing |
| `TAURI_SIGNING_PRIVATE_KEY_WEST` | Sign updater artifacts of the **west** build (`flavor: west`) — its own key pair, pubkey in `src-tauri/tauri.west.conf.json` | Owner's password manager (generated 2026-10-24, key id `3EA2495B90742A8E`). Same key in `macos-ci`. Never the Russian key: `release.mjs` checks the key id against the pubkey and stops |
| `TAURI_SIGNING_PRIVATE_KEY_PASSWORD_WEST` | Passphrase for the west key | Same |
| `UPDATE_SECRET` | Bearer token for backend POST `/api/update/windows` (and `/windows-west`) | Backend env |

## Flavors (W2)

`flavor: ru` is the release as it always ran: bump, build, publish, push the bump commit.

`flavor: west` re-builds the last Russian release (`chore: release <version>` on GitLab `main`) as the west build — own update channel `/api/update/windows-west`, own key, package `smoozy-windows-west`, downloads record `?market=west`. No bump, no push; `bump` is ignored. Run it after the Russian release of the same version; `publish_manifest` works the same way for it.

## Branch protection

This repo is public, but only collaborators can trigger workflows (`workflow_dispatch`). External PRs cannot run CI without approval.

## Not using this approach?

Alternative architectures considered and rejected:

- **Everything on GitHub (public source):** code becomes open — not acceptable
- **Everything on GitHub (private source):** 2000 min/month Free tier Windows multiplier 1.67 ≈ 1200 effective min. Works, but burns through quickly
- **Everything on GitLab CI:** Windows runner still in beta after 5+ years (2026). No SLA, provisioning 5+ min/job

This split (source on GitLab + CI on public GitHub) gives us: private source **and** unlimited CI minutes.
