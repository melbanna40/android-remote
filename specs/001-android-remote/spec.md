# 001 — android-remote: spec

Status: draft for owner review · 2026-11-05 · License: MIT · Target repo: `melbanna40/android-remote`

## 1. Problem

Running an Android emulator locally costs 3–6 GB of RAM and 2–4 cores. Developers on small laptops, and AI coding agents (Claude Code, GLM on Hermes, Orca workers) that run many tasks in parallel, cannot afford one emulator per task. GitHub-hosted Linux runners have KVM, so they can run a hardware-accelerated x86_64 emulator, and Linux minutes are free for public repositories.

`flutter-remote` (AbdAlftahSalem/flutter_remote, MIT) does this for iOS: it builds a Flutter app on a macOS runner, boots the Simulator and streams it to a browser through a cloudflared quick tunnel. `android-remote` is the Android equivalent. It is reusable by any repository, and it adds headless E2E runs and an API for AI agents.

## 2. Goals

- G1: **Interactive session.** The emulator streams to a browser behind a private link and access key. The session length is an input.
- G2: **Headless E2E.** Run Maestro flows and Flutter `integration_test` on the emulator. Upload screenshots, screen recordings, logcat and JUnit as artifacts. Usable as a PR check.
- G3: **AI-agent access.** An agent can drive the emulator remotely through an authenticated HTTP API and a local MCP bridge: tap, type, swipe, key, screenshot, UI tree, install, launch, logcat and deep links. Raw adb is never exposed.
- G4: **App input.** Build from source (Flutter flavor, target, dart-defines, or a Gradle task for native apps), or install a prebuilt APK from a URL, a release asset or a workflow artifact.
- G5: **Easy to integrate.** A few lines of YAML (`uses: melbanna40/android-remote...@v1`) plus an npm CLI (`npx android-remote up|down|status|doctor|init|mcp`).

## 3. Non-goals (v1)

- iOS. flutter-remote already covers it.
- Self-hosted or larger runners as a supported target. They may work, but v1 does not test them.
- Physical devices, Play Store sign-in, multiple emulators per session.
- Patrol as a first-class runner (planned for v1.1; v1 offers a generic `e2e: custom` command that can call `patrol test`).
- WebRTC/TURN. Streaming is WebSocket over the tunnel.
- Uploading a local APK file through `up`. That would need a push or a release, which the CLI must never do. In a live session, use the agent API's `install` upload.

## 4. Hard constraints (owner)

- C1: The CLI **never** commits, pushes, creates repositories, creates releases or writes repository secrets. `init` only writes a file to disk. `up` only dispatches a workflow on a ref that already exists on GitHub.
- C2: Public repositories expose Actions logs, job summaries and artifacts to everyone. No access key, agent token or session URL may appear in plain text in any of them.
- C3: MIT license. Third-party GPL tools may be downloaded and executed at runtime as separate programs, but are never vendored.
- C4: Node 20+ for the CLI.

## 5. Users and stories

| ID | As a… | I want… | So that… |
|---|---|---|---|
| US1 | developer on a weak laptop | `npx android-remote up --flavor development` and a link that opens my app | I can use the app without a local emulator |
| US2 | developer | to choose the session length (`--minutes 45`) and end it early (`down`) | I don't waste runner minutes |
| US3 | maintainer | a PR check that runs my Maestro flows on API 34 and attaches a video and logcat when a flow fails | I can debug failures without reproducing them |
| US4 | Flutter developer | to run `integration_test/` on the emulator with JUnit output | test results show in CI |
| US5 | AI agent | MCP tools (tap/type/screenshot/ui-tree/logcat/deep-link) on a remote emulator | I can verify my UI changes without a local device |
| US6 | developer | to run a prebuilt APK (release asset, URL or another run's artifact) | I can check a build without rebuilding |
| US7 | integrator | `npx android-remote init` to write a workflow file I review and commit myself | nothing gets pushed for me |
| US8 | maintainer of a public repo | to know that the link and key never reach the public logs | strangers can't take over my session |
| US9 | developer with secrets (`.env`, `envied`) | to inject build-time env from repository secrets | secrets stay out of inputs and logs |

## 6. Functional requirements

### FR-1 Integration surface
- FR-1.1 A root composite action `melbanna40/android-remote@v1` runs the full pipeline (setup → app → mode) inside a caller's own job.
- FR-1.2 A reusable workflow `melbanna40/android-remote/.github/workflows/android-remote.yml@v1` (`on: workflow_call`) wraps the action on `ubuntu-latest`, with a computed timeout. It accepts `secrets: inherit`.
- FR-1.3 A caller template (written by `init`) exposes `workflow_dispatch` with **at most 10 inputs**. Rarely used options go in one `config` JSON input.
- FR-1.4 Standalone sub-actions: `actions/setup-emulator`, `actions/e2e`. Teams that already build their APK can use them on their own.

### FR-2 App input
- FR-2.1 `app_source: flutter` builds an APK with `flutter build apk --<build_mode> [--flavor] [-t target] [--dart-define…] [--dart-define-from-file]` in `app_dir`. `build_mode` is `debug` (default), `profile` or `release`.
- FR-2.2 `app_source: gradle` runs a configurable Gradle task (default `assembleDebug`) in `app_dir`, then picks the APK with a configurable glob.
- FR-2.3 `app_source: apk` installs from `apk: url:<https-url>`, `release:<tag>/<asset>` (current repo), or `artifact:<run-id>/<artifact-name>` (current repo).
- FR-2.4 Optional `pre_build` script path (in the caller repo) runs before the build, for code generation such as `build_runner`.
- FR-2.5 Secrets: if the repository secret `ANDROID_REMOTE_DOTENV` exists, its content is written to `dotenv_path` (one or more paths, comma-separated) before `pre_build`. If `ANDROID_REMOTE_DART_DEFINES_JSON` exists, it is passed as `--dart-define-from-file`. Neither is ever echoed.
- FR-2.6 After install, the APK is checked: package name, launchable activity and native ABIs (via `aapt2 dump badging`). A warning appears when the APK has native libraries but no `x86_64` ones.
- FR-2.7 Built APKs are cached by a fingerprint of the source tree, build inputs and secret digests. A cache hit skips the build.

### FR-3 Emulator
- FR-3.1 Enable KVM, install the system image, create the AVD and boot it headless (`-no-window`) with the defaults in plan.md §8. Animations are disabled.
- FR-3.2 Inputs: `api_level` (30–36, default 34), `target` (`google_apis` default, `google_atd`, `aosp_atd`, `default`), `device` (AVD profile, default `pixel_7`), cores and RAM sized to the runner.
- FR-3.3 The AVD and a clean boot snapshot are cached per (api, target, arch, profile, emulator build).
- FR-3.4 If boot fails or takes longer than the timeout (default 600 s), the step fails with the emulator log attached.

### FR-4 Interactive mode (`mode: interactive`)
- FR-4.1 A browser viewer streams the emulator with touch, keyboard, scroll, back/home/recents and rotation.
- FR-4.2 An auth gate in front of everything: the link carries a one-time `?k=<key>`. The gate swaps it for an `HttpOnly; Secure; SameSite=Strict` cookie and redirects to the same URL without `k`. Any request without a valid cookie or key gets 403, and WebSocket upgrades without one are refused.
- FR-4.3 The session is exposed through a cloudflared quick tunnel. No Cloudflare account is needed.
- FR-4.4 The session ends at the earliest of: `minutes` elapsed (1–300, default 30), `idle_minutes` with no authenticated request (default 10, `0` = off), `android-remote down`, or a run cancel.
- FR-4.5 The URL, viewer key and agent token reach the CLI **only** as a sealed (encrypted) artifact, readable only with the CLI's ephemeral private key (plan.md §6). They are masked in the logs.
- FR-4.6 A manual UI dispatch with no `client_public_key`: in a private repo, the link and key are written to the job summary (visible only to people with repo access). In a public repo, the run **fails before booting** with an explanation.
- FR-4.7 Gate logs never contain query strings, cookies or Authorization headers.

### FR-5 E2E mode (`mode: e2e`)
- FR-5.1 Runners: `e2e: maestro` (flows dir or file, `--include-tags`/`--exclude-tags`, env), `e2e: flutter` (`flutter test <path> -d <serial>`), `e2e: custom` (a shell command; Patrol users use this in v1).
- FR-5.2 Always collected: JUnit XML, full logcat (`-v threadtime`), a screen recording (mp4 segments), a final screenshot, `device.json` and Maestro's debug output and per-step screenshots when the runner is Maestro.
- FR-5.3 Everything goes into one artifact, `android-remote-e2e-<run_id>-<attempt>` (retention input, default 14 days).
- FR-5.4 A job summary with a pass/fail table and the failed test names. No secrets, no URLs.
- FR-5.5 Action outputs: `result` (`passed|failed`), `tests`, `failures`, `junit-path`, `artifact-url`.
- FR-5.6 The job's exit code reflects the test result, so it can be a required check.
- FR-5.7 Works on `pull_request` from forks (no secrets needed unless the build needs them). Interactive mode is refused on `pull_request*` events.

### FR-6 Agent API (`agent: true`, interactive mode only)
- FR-6.1 HTTP JSON API under `<tunnel>/agent/v1/*`, authenticated with `Authorization: Bearer <agent_token>`. This token is separate from the viewer key.
- FR-6.2 Endpoints (plan.md §7): device info, tap, swipe, type (ASCII and Unicode, including Arabic), key, screenshot, UI tree, install (upload or URL), launch, stop, clear-data, open-link, logcat.
- FR-6.3 No arbitrary shell endpoint. Every parameter is validated, and device-side arguments are shell-quoted.
- FR-6.4 Rate limits and auth-failure lockout (plan.md §7.4).
- FR-6.5 `npx android-remote mcp` runs a local stdio MCP server that proxies the API, using the token from the local session file. The token never goes into MCP config files.

### FR-7 CLI (`android-remote`)
- FR-7.1 `init [--app-dir] [--app-source] [--flavor] [--target] [--pin] [--force]` writes `.github/workflows/android-remote.yml` and prints the git commands for the user to run. Running `git`/`gh` write commands is forbidden.
- FR-7.2 `doctor` checks Node ≥ 20; `gh` installed, logged in and with repo access; Actions enabled; the workflow file present on the default branch and on the chosen ref; template version. It prints a fix for every failure.
- FR-7.3 `up [--ref] [--mode] [--minutes] [--idle-minutes] [--api] [--device] [--flavor] [--target] [--dart-define K=V…] [--apk <spec>] [--agent] [--e2e …] [--wait] [--no-open] [--json]` dispatches, follows the state machine, unseals the session, prints the link and saves the session.
- FR-7.4 `status [--json] [--show-key]`, `down [--all]`, `mcp`.
- FR-7.5 Before dispatch, `up` warns when the local tree has uncommitted or unpushed changes ("the runner builds `<ref>@<sha>`; local changes are not included").
- FR-7.6 Session state lives at `<git-dir>/android-remote/session.json` (mode 0600, never committed).
- FR-7.7 Exit codes: `0` ok, `1` failure, `2` usage, `3` prerequisite missing, `4` remote run failed.
- FR-7.8 `--json` gives machine-readable output for agents.

## 7. Non-functional requirements

- NFR-1 Time to live session: ≤ 6 min with warm caches (prebuilt APK); ≤ 12 min for a cold Flutter build.
- NFR-2 Stream latency target: ≤ 250 ms median round trip from input to frame for EU/US users on a 20 Mbps link, at 30 fps and max size 1024 px.
- NFR-3 Zero secrets in public outputs (logs, job summary, statuses, run names, plaintext artifacts). This is verified by an automated log scan (V5).
- NFR-4 All third-party actions pinned by commit SHA. Downloaded binaries pinned by version plus SHA-256.
- NFR-5 Compatible with GitHub-hosted `ubuntu-latest` (24.04, x86_64) standard runners in public and private repos.
- NFR-6 The CLI has zero runtime dependencies except `@modelcontextprotocol/sdk`, which only the `mcp` command loads.

## 8. Acceptance criteria

- AC1 From a fresh external test repo: `npx android-remote init` → the user commits → `npx android-remote up` prints a link. Opening it shows the app, taps work, and opening the link without `k` (or in a fresh profile) returns 403.
- AC2 `down` cancels the run within 30 s, and the tunnel URL stops answering.
- AC3 A PR check with `mode: e2e, e2e: maestro` passes on a green flow. On a failing flow it fails and uploads JUnit, mp4, logcat and screenshots.
- AC4 `e2e: flutter` produces JUnit from `integration_test/`.
- AC5 The agent smoke script (tap, type Arabic text, screenshot, ui tree, open-link, logcat) passes against a live session. Calls without a token get 401, and the 11th failure in a minute gets 429.
- AC6 A scan of the full run logs and job summary (`gh run view --log`) for the key, the token and the trycloudflare host finds nothing (public test repo).
- AC7 A manual UI dispatch in a public repo with no `client_public_key` fails fast with the documented message.
- AC8 The CLI source has no `git commit|push`, `gh repo create`, `gh release create` or `gh secret set` calls (grep check in CI).

## 9. Edge cases

- The workflow file exists only on a feature branch. `workflow_dispatch` needs it on the default branch first: `doctor`/`up` explain this.
- Dispatch accepted, but the run doesn't appear within 60 s (queue delay): keep polling up to 10 min, then exit 4 with the run list URL.
- Quick tunnel fails to start (Cloudflare outage or rate limit): retry 3× with `--protocol http2`, then fail with `cloudflared.log` in the logs artifact.
- More than 200 in-flight requests on a quick tunnel → 429 (Cloudflare limit). The viewer uses one WebSocket and the agent API is rate limited below that.
- SSE is unsupported on quick tunnels. No component may rely on SSE (this rules out a remote MCP server using SSE).
- An APK with only arm64 native libs: works on API 30+ x86_64 images through ARM translation, but slowly. Show a warning.
- The release build needs signing secrets the repo lacks: the build fails and the hint says to use `build_mode: debug`.
- An emulator with Arabic locale or RTL: the `config.locale` (e.g. `ar-EG`) setting is applied after boot.
- Concurrent `up` calls from the same checkout: each gets its own session. `status` lists all of them and `down` without `--all` stops the latest.
- The runner was cancelled while the CLI waits: the CLI reports `cancelled` and clears the session.
- The cache store is full (10 GB per repo): AVD snapshot and APK caches are evicted. This is reported, not fatal.

## 10. Research basis (read 2026-11-05)

- flutter-remote README, `src/runner/workflow/flutter-remote.yml`, `src/core/session.js`, `src/cli/commands/down.js`, `src/core/gh.js`. Patterns reused: session id in `run-name` for run lookup, the gate with a one-time `?k=` swapped for a cookie, nohup background services across steps, an app fingerprint cache, `gh run cancel` for `down`, session state under `.git/`. Patterns avoided: pushing and creating repos; passing `gate_token` as a dispatch input that is then interpolated into `env:` and `run:` (step headers print it in the logs); publishing the live URL through a commit status `target_url` (public on public repos).
- ReactiveCircus/android-emulator-runner v2.38.0 `action.yml` and README: KVM udev rule on `ubuntu-latest`, default options `-no-window -gpu swiftshader_indirect -no-snapshot -noaudio -no-boot-anim`, AVD snapshot caching recipe, `script` as the only place the emulator is alive.
- NetrisTV/ws-scrcpy README (MIT, last release v0.8.1 2024-03, scrcpy 1.19 fork, "no authorization on any level", emulator needs "proxy over adb").
- bilbospocketses/ws-scrcpy-web README and SECURITY.md (GPL-3.0, active 2026-11, vanilla scrcpy-server, WebCodecs only; Host allowlist `allowedHosts`, loopback = implicit admin, Docker uses wireless ADB).
- Cloudflare TryCloudflare docs: no account, 200 in-flight request limit, **no SSE**, no uptime guarantee, a new hostname per tunnel.
- budtmo/docker-android `mcp/src/server.py`, `adb.py`: FastMCP HTTP transport, a small tool set, no auth, device commands built by string interpolation (injection risk we avoid).
- actions/upload-artifact v4 README: `artifact-id` output usable with the REST API, retention 1–90 days.
- mobile-dev-inc/maestro `TestCommand.kt` (cli-2.12.1): `--format junit --output`, `--test-output-dir`, `--debug-output`, `--include-tags/--exclude-tags`, `--device`.
- `gh workflow run --help` (gh 2.103.3): "The created workflow run URL will be returned if available."
- senzhk/ADBKeyBoard (GPL-2.0): `am broadcast -a ADB_INPUT_B64` for Unicode text.
