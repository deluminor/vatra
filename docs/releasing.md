# Releasing Vatra

Vatra ships its own releases from the `main` branch of `deluminor/vatra`: desktop installers, the remote host packages, and the signed feed the app updates from. Upstream MonoCode releases are not used for anything.

## Cut a release

1. Merge the work into `main` and make sure CI is green.
2. Optional: write the notes under `## [Unreleased]` in `CHANGELOG.md` (Keep a Changelog sections: Added, Changed, Fixed, Removed).
3. **Actions → Release → Run workflow**, branch `main`, version `patch`, `minor`, `major`, or an exact `X.Y.Z`.

The workflow then:

- bumps `package.json`, `package-lock.json`, `Cargo.toml`, `Cargo.lock` and `tauri.conf.json` (`scripts/release/prepare.mjs`);
- writes the release section of `CHANGELOG.md`. An existing `## [X.Y.Z]` section is kept as is. Otherwise the `Unreleased` notes are combined, section by section, with notes generated from Vatra's conventional commits since the previous Vatra tag (`feat` → Added, `fix` → Fixed, `perf`/`refactor`/`revert`/other → Changed). These commits are left out of the generated notes:
  - `chore`, `ci`, `docs`, `test`, `build`, `style` and merge commits;
  - commits that edited `CHANGELOG.md` themselves, since they already wrote their notes;
  - MonoCode commits reachable from `upstream-main`, which the sync PR describes under `Unreleased`;
- opens a `release/vX.Y.Z` branch, squash-merges a PR into `main` (required by the branch ruleset), tags the merge commit, and pushes the tag (re-run if the PR conflicts because `main` moved);
- builds macOS (arm64 + x64), Windows, Linux (`.deb`, AppImage, `.rpm`) and the six host packages from that tag;
- publishes the GitHub release with the changelog section as its body, plus `latest.json` for in-app updates.

Pushing a `vX.Y.Z` tag yourself runs the same build, provided the tag points at a commit on `main` and the manifests and changelog already carry that version (`scripts/release/verify.mjs`).

A failed build leaves the release as a draft; use **Re-run failed jobs** on that run to resume it (starting a new run would try to bump the version again). Published releases are never modified.

## Versions

Vatra uses its own semver line, starting at 1.0.0. The remote host must match the desktop version exactly — SSH setup downloads `vatra-host-<os>-<arch>` from the release of the running app's version — so every release must include the host packages (the workflow refuses to publish without all six).

## Signing

| What | Key | Where it lives |
| --- | --- | --- |
| Update packages (`.app.tar.gz`, NSIS `.exe`) | Tauri updater key (minisign) | Secrets `TAURI_SIGNING_PRIVATE_KEY`, `TAURI_SIGNING_PRIVATE_KEY_PASSWORD`; public key in `src-tauri/tauri.conf.json` → `plugins.updater.pubkey` |
| macOS app | Apple Developer ID (optional) | Secrets `APPLE_CERTIFICATE`, `APPLE_CERTIFICATE_PASSWORD`, `APPLE_SIGNING_IDENTITY`, `APPLE_API_KEY`, `APPLE_API_KEY_P8`, `APPLE_API_ISSUER` |

**Keep a backup of the updater private key and its password outside GitHub** (a password manager). Installed apps only accept updates signed by that key: if it is lost, every existing install has to be updated by hand once with a build carrying a new public key. If it leaks, set `APP_UPDATER_DISABLED = true` in `src/app/model/forkPolicy.ts` (and `src-tauri/src/menu.rs`), rotate the key, and ship a manual release.

Without the Apple secrets the macOS build is ad-hoc signed and not notarized; users confirm the first launch once (see README → Download). Adding the secrets switches the workflow to signing and notarization automatically.

## Syncing from upstream

Vatra is a standalone repository, not a GitHub fork, so GitHub's **Sync fork** is unavailable and PRs cannot target MonoCode from here. Syncing is plain git against the `upstream` remote (`hardbeat920/monocode`), which works as long as MonoCode stays public.

| Ref | Role |
| --- | --- |
| `upstream/main` | MonoCode's default branch |
| `upstream-main` | Mirror of `upstream/main` on this repo; only ever fast-forwarded, never committed to |
| `main` | Vatra; upstream changes arrive through a `sync/upstream-into-main-YYYYMMDD` PR |

The **Sync MonoCode upstream** automation in Vatra (Tuesday and Friday, 09:00) does both steps: it fast-forwards `upstream-main`, then opens a sync PR into `main` with release notes and a cross-linked Issue. It keeps Vatra's side in the Vatra-owned paths below, and resolves product-code conflicts by combining both sides: it keeps Vatra's features and names and ports in the upstream change. A **Sync blocked** Issue is opened only when the two sides are truly incompatible, or when checks still fail after merge-caused errors are fixed. Merge sync PRs with a merge commit — a squash drops the upstream ancestry and the next sync conflicts again.

Every sync, including one that merges cleanly, also:

- renames MonoCode identifiers that upstream code brings in to Vatra's: `monocode.*` storage keys and `monocode:*` events become `vatra.*` / `vatra:*`, and the same goes for component names and user-facing text. Upstream issue links and the legacy-profile migration code keep their MonoCode names;
- adds the incoming user-visible changes under `## [Unreleased]` in `CHANGELOG.md`, citing upstream PRs as `MonoCode #NNN`. Upstream commits are not conventional commits, so without these notes the release script would leave them out;
- runs `npm run check:web` (plus `cargo check` and the host tests when the merge touches them), and lists user-visible default changes under **Breaking / attention** in the notes.

By hand:

```bash
git remote add upstream https://github.com/hardbeat920/monocode.git   # once per clone
gh repo set-default deluminor/vatra                                    # once per clone
git fetch upstream main
git push origin upstream/main:refs/heads/upstream-main                 # fast-forward the mirror
git switch -c sync/upstream-into-main-$(date +%Y%m%d) origin/main
git merge origin/upstream-main
# resolve conflicts, keep Vatra-owned paths below, open a PR into main
```

These parts are Vatra-owned; keep ours when porting upstream changes:

- `.github/workflows/release.yml`, `scripts/release/`
- `src/app/model/forkPolicy.ts`, `plugins.updater` in `src-tauri/tauri.conf.json`
- `RELEASE_DOWNLOAD_BASE` in `src-tauri/src/remote_ssh.rs`, `src-tauri/src/remote_bootstrap.{sh,ps1}`
- host names in `host/` (`~/.vatra-host`, `vatra-host`, `com.vatra.host`, `Vatra Host-<SID>`, the `host.vatra` capability)
- version numbers and `CHANGELOG.md`

License notices are the exception. Keep Vatra's additions, but always port upstream changes to them: a new or changed copyright line in `LICENSE`, `NOTICE` content, `LICENSE*`/`NOTICE*` files under `vendor/` or anywhere else, and copyright or SPDX headers in source files. MIT requires every copy to carry these notices, so dropping an upstream change here is a license violation, not just a style choice.

Every build ships the notices: macOS, Windows and Linux bundles include `LICENSE` and `NOTICE` as resources; `THIRD-PARTY-NOTICES.md` (generated by `vite build` from the bundled JavaScript dependencies) is part of the frontend that Tauri embeds into every binary; the host packages include `VATRA-LICENSE`, `VATRA-NOTICE` and Node's `LICENSE`. Bundle resources must be files committed to the repository, because `cargo check` validates them without a frontend build.
