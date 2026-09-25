# Streambert Android Port — Complete Issues Log

Every issue/error encountered during the push + EAS APK build phase (2026-09-23 → 2026-09-24),
with root cause, fix, and current status.

---

## A. Environment / git / workspace issues

| # | Issue | Root cause | Fix | Status |
|---|-------|-----------|-----|--------|
| A1 | First push to GitHub only pushed a **gitlink** (commit pointer) for `expo-app/`, not real files | Nested git repo: `expo-app/.git` made the parent treat expo-app as a submodule | `git rm --cached expo-app`, moved internal `.git` to backup, re-committed as real tree | FIXED |
| A2 | Pushed tree **silently missed `expo-app/assets/dist/`** despite working-tree "identical" diff | Parent-directory gitignore trap: unanchored `dist/` in root `.gitignore` pruned descent, so `!expo-app/assets/dist/**` negations could not fire | Anchored to `/dist/` in root `.gitignore` (commit `67227f9`); verify pushes via `git ls-tree -r` | FIXED + regression trap |
| A3 | Workspace snapshot restores **delete `.git` metadata** (remotes, identity) and randomly roll back worktree files | Sandbox snapshot policy (git config/credentials excluded; best-effort capture) | Recovery procedure: `git init → remote add → fetch → reset --mixed FETCH_HEAD`; verify `git status` against expected file set before every commit. Caught and repaired a rollback of `expo-module.config.json` (commit `7473996`) | ENVIRONMENTAL — STILL A HAZARD (mitigated, procedural) |
| A4 | `node_modules` / `dist` / caches wiped on restore; eas-cli broke with "Failed to resolve plugin expo-router" | Same snapshot policy (dependency dirs excluded by design) | Re-run `npm ci` / `npm install` per repo after every restore | MITIGATED (recurring) |
| A5 | Tokens/credentials cannot persist between sessions | Security rule (tokens env-only, never written) | User re-pastes per phase; advise rotation after exposure | STANDING RISK |

## B. EAS gradle build failures (4 sequential root causes — builds: c0eace4f → 39f02194 → fc269ba5 → 8bc9d9a0 → 68d94831 → 099ef266 SUCCESS)

| # | Exact gradle error (from brotli-decoded EAS worker logs) | Root cause | Fix | Status |
|---|----------------------------------------------------------|-----------|-----|--------|
| B1 | `Cannot run program "class groovy.util.Node"` at module `build.gradle:9` | Bare `Node` identifier (resolves to `groovy.util.Node` class) used where the string `"node"` was required — `[Node, "--print", ...].execute(...)` on lines 9/12/13 | Quoted all three: `["node", ...]` | FIXED + parity regression trap |
| B2 | `unable to resolve class org.jetbrains.kotlin.gradle.tasks.KotlinCompile` compiling `expo-modules-core/android/build.gradle:1` | Module applied expo-modules-core's `build.gradle` from inside its own `buildscript{}` (standalone-package template) — nested buildscript scoping breaks script compilation classpath | Re-wrote header to canonical npm-module form: apply `ExpoModulesCorePlugin.gradle` + `applyKotlinExpoModulesCorePlugin()` + `useDefaultAndroidSdkVersions()` + `useExpoPublishing()` | FIXED |
| B3 | `Using singleVariant publishing DSL multiple times to publish variant "release"` at `build.gradle:38` | Leftover `android { publishing { singleVariant("release") } }` conflicted with `useExpoPublishing()` (does the same internally) | Removed our publishing block | FIXED |
| B4 | `ExpoModulesPackageList.java:29: illegal start of expression` — generated file contained literal `[object Object].class` | `expo-module.config.json` used object-form `modules: [{packageName, className}]`; expo-modules-autolinking expects **string FQCN** list. **The old parity test itself enforced the buggy object form** | Config → `"modules": ["expo.modules.externalplayer.ExternalPlayerModule"]`; test rewritten to require string form | FIXED + test hardened |
| B5 | `uses-sdk:minSdkVersion 21 cannot be smaller than version 24 declared in library com.facebook.react:react-android:0.76.9` (manifest merger, `:app:processReleaseMainManifest`) | app.json set minSdk 21; React Native 0.76 mandates ≥24 | `minSdkVersion: 24` in app.json (Android 6/API 23 devices now unsupported — unavoidable) | FIXED |
| B6 | `expo export --platform android` → "platforms array: [web]" on clean checkouts | Android platform only inferred when `android/` build dir exists; dir is gitignored so clean clones/fresh nodes lack it | `expo.platforms: ["android"]` added to app.json | FIXED |
| B7 | Recurring secondary error `':expo' Could not get unknown property 'release'` (all failed builds) | Cascade/fallout of B2–B4 aborting configuration midway | Resolved automatically once B2/B3/B4 fixed; build 099ef266 clean | RESOLVED (correlation confirmed) |
| B8 | `run_all.sh` parity suite false-FAIL on clean clone | Orchestrator order: config parity ran before frontend `dist` was produced | Moved `frontendDist` step before `parity` in `tests/run_all.sh` | FIXED |

## C. Tooling/access friction (transient — all fixed in-flight)

| # | Issue | Resolution |
|---|-------|-----------|
| C1 | `eas build:view --json` exposes no phase detail | GraphQL `app{byId(appId){builds{... logFileUrls}}}` |
| C2 | Log URLs = binary "data" blobs (13897 B) | They are **brotli-compressed NDJSON** (worker log lines) — decode with `pip brotli` |
| C3 | Signed GCS log URLs expire (`X-Goog-Expires=900`) | Regenerate via fresh `build:view` and download immediately |
| C4 | Older GraphQL attempts invalid (`buildById`, `buildPhases` fields don't exist) | Correct schema: `Build.logFileUrls`, phases only in decoded worker logs |
| C5 | `git push -c core.askPass=...` (invalid — config goes before subcommand) | Use transient `https://x-access-token:...@github.com/...` URL |
| C6 | eas-cli refuses build outside a git repo (after .git wipe) | Restore git metadata (A3 procedure) before submitting |
| C7 | Release-create 422 (my payload env bug) | Rebuilt payload via file; fixed immediately |
| C8 | `file` reports "data" / raw-deflate multi-segment attempts failed | Wrong codec guesses; brotli was the container |

## D. STILL OPEN / pending (not errors, but unverified or risky)

| # | Item | Detail |
|---|------|--------|
| D1 | **Physical device testing NOT done** | No hardware in this environment. Open items per `DEVICE_VERIFICATION.md`: install, cold start, proxy host reachability, external-player launch, TV mode, stream/proxy playback. Never claimed as tested. |
| D2 | **Tokens exposed in chat** | GitHub PAT (`ghp_ikQ…`) + Expo access token were pasted in plaintext. Must be **revoked & rotated** (GitHub → Settings → Developer settings → Tokens; expo.dev → Access Tokens). Advised twice; still open unless done. |
| D3 | **Play Store signing decision** | APK signed via EAS-managed keystore (APK Signature Scheme v2). For Play release must pin credentials (EAS credentials storage or own keystore) so update signatures stay stable. ADVISORY only — no action yet. |
| D4 | **Corrupt/stray workspace dirs** | `streambert-fork/` (rolled-back stale copy), `expo-app-git-backup/`, three `.bundle` backups — kept as safety; cleanup decision pending (this task). |
| D5 | **Android support floor moved to API 24** | Consequence of B5: Android 6.0 (API 23) devices excluded. Documented, not an error. |
| D6 | Hardware-verified features | Proxy/CDN/TV-mode behaviors remain "hardware-pending" until D1 completes. |
