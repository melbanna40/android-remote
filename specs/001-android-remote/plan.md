# 001 — android-remote: plan

Companion to [spec.md](spec.md). Sources are cited in spec.md §10. Items marked **[SPIKE]** must be confirmed by the spike packets in tasks.md before code depends on them.

## 1. Architecture

```text
developer / AI agent machine                         GitHub-hosted ubuntu-latest runner (x86_64, KVM)
────────────────────────────                         ────────────────────────────────────────────────────────────
npx android-remote up                                caller workflow (.github/workflows/android-remote.yml, written by init)
  ├─ X25519 keypair (private key stays local)          └─ reusable workflow melbanna40/android-remote/.github/workflows/android-remote.yml@v1
  ├─ gh workflow run … (inputs: session,                    └─ root composite action melbanna40/android-remote@vX.Y.Z
  │    client_public_key, mode, minutes, …)                     prepare.mjs → secrets.mjs (keys generated HERE, masked)
  ├─ poll run + artifacts list                                  emulator start (background) ∥ app build / fetch APK
  ├─ download android-remote-session-<id>                       emulator wait → install → mode:
  │    → unseal → URL + viewer key + agent token                  interactive: viewer (ws-scrcpy-web) → gate.mjs → cloudflared quick tunnel
  └─ print https://<rand>.trycloudflare.com/?k=…                             → publish.mjs (sealed artifact) → hold.sh (deadline/idle)
                                                                 e2e:         record.sh → maestro|flutter|custom → collect.sh → artifact + summary
browser ──https──▶ Cloudflare ──▶ cloudflared ──▶ gate.mjs :3199 ─┬─▶ viewer :8000  (cookie auth)
npx android-remote mcp (stdio) ──https──▶ …   ──▶ gate.mjs ───────┴─▶ agent-api.mjs :4310 (Bearer agent token + internal token) ──adb──▶ emulator-5554
```

## 2. Key decisions

| # | Decision | Why |
|---|---|---|
| D1 | **Own the emulator lifecycle** (scripts modeled on android-emulator-runner) instead of `uses: ReactiveCircus/android-emulator-runner` | That action keeps the emulator alive only inside its `script` input (each line run separately) and kills it when the step ends. A session needs the emulator alive across many steps (viewer, tunnel, artifact upload, hold). We copy its KVM rule, default flags and snapshot-cache approach. |
| D2 | **One root composite action is the engine.** The reusable workflow is a thin job that calls it. Two standalone sub-actions (`setup-emulator`, `e2e`) for custom pipelines. | A single place where the logic lives. Reusable workflows can't use `./` to reach their own repo's actions, so the reusable workflow pins `melbanna40/android-remote@<tag>` and the release script rewrites that one ref (§13). |
| D3 | **Keys are generated on the runner and returned sealed (X25519 + HKDF + AES-256-GCM) in an artifact.** Nothing secret goes in dispatch inputs. | Dispatch inputs end up in step `env:` headers and expanded `run:` scripts in public logs (flutter-remote passes `gate_token` as an input). Artifacts can be fetched while the run is in progress through the REST API (`artifact-id` output, v4). Ciphertext is safe even in a public repo. Node's built-in `crypto` covers both sides, so no dependencies. |
| D4 | **Viewer: ws-scrcpy-web (Docker image pinned by digest) behind our gate, with NetrisTV/ws-scrcpy as fallback**, through a viewer adapter contract | ws-scrcpy-web uses vanilla Genymobile scrcpy-server, so it keeps up with API 34–36. ws-scrcpy uses a 2021 scrcpy 1.19 fork and had its last release in 2024. GPL-3 is fine because we only *run* the published image and never vendor it. **[SPIKE S1]** confirms that streaming works through the gate on API 34/35 and lists the routes to deny. |
| D5 | **Our own small auth gate** (`gate.mjs`, Node built-ins only) in front of the viewer and agent API | Neither viewer has an auth boundary we can use behind a tunnel (ws-scrcpy: "no authorization on any level"; ws-scrcpy-web: login is opt-in and loopback = implicit admin). One place enforces the key, cookie, rate limits and path denylist. |
| D6 | **Cloudflared quick tunnel**, `--protocol quic` with an `http2` fallback, pinned version and SHA-256 | No account, and it is proven by flutter-remote. Its limits (200 in-flight requests, no SSE, no uptime guarantee) are compatible: one WebSocket plus a rate-limited JSON API. |
| D7 | **Agent API = plain authenticated REST on the runner. MCP = a local stdio bridge (`npx android-remote mcp`)** | Quick tunnels don't support SSE, so a remote MCP over SSE is ruled out and Streamable HTTP is fragile through them. A local stdio bridge works in every MCP client (Claude Code, Hermes, OpenCode), keeps the token out of MCP config, and lets `install_apk` upload a local file. |
| D8 | **No raw adb exposure** | adb gives a full device shell, file push and port forwarding, and the device can reach the runner's loopback through 10.0.2.2. Typed endpoints with validation remove that whole class of risk. budtmo/docker-android's MCP shows the string-interpolation injection risk we avoid. |
| D9 | **E2E v1 = Maestro first, then Flutter `integration_test`. Patrol in v1.1** (reachable now through `e2e: custom`) | Maestro is black-box: it works for Flutter and native apps and for prebuilt APKs, emits JUnit and screenshots natively (`--format junit`, `--test-output-dir`, `--debug-output`) and needs no project changes. `integration_test` is the zero-dependency Flutter default. Patrol needs native test-runner wiring in each project's `android/app` and a matching `patrol_cli` version, which is support-heavy for a first release. |
| D10 | **CLI in plain ESM JavaScript with JSDoc `// @ts-check`**, type-checked by `tsc --noEmit` in CI only | No build step, so it runs from `npx github:melbanna40/android-remote` and shares `shared/sealed-box.mjs` with the runner as-is. That is less config for the low-cost implementers, and types are still checked. |
| D11 | **Session signalling = run status + artifact presence. No commit statuses, issue comments or check runs.** | Needs only `actions: read` and `contents: read`, adds no noise on commits or PRs, and publishes nothing readable in public. The CLI shows the run URL for step-level detail. |
| D12 | **Default image `google_apis` x86_64, API 34, `pixel_7`** | x86_64 + KVM is the only fast path on GitHub Linux runners. `google_apis` includes Google Play services (FCM, Maps, Sign-In SDK) and allows `adb root` (locale changes, debugging). Play Store images block root and need an account. API 34 is stable and widely targeted. |
| D13 | **Caller template ≤ 10 dispatch inputs + one `config` JSON** | Keeps the template under every documented `workflow_dispatch` input limit, and new options don't break existing caller files. |
| D14 | **Screen recording = segmented `adb screenrecord`** (180 s per segment) | No extra dependency, works headless. Fallback **[SPIKE S4]**: scrcpy v5 `--record --no-window`, pinned linux tarball. |

## 3. Repository layout

```text
android-remote/
├── action.yml                         # root composite: the engine (all modes)
├── actions/
│   ├── setup-emulator/action.yml      # standalone: KVM + system image + AVD + boot (+cache)
│   └── e2e/action.yml                 # standalone: run tests on a booted emulator + collect + upload
├── .github/
│   ├── workflows/android-remote.yml   # reusable workflow (on: workflow_call) → uses the root action
│   ├── workflows/ci.yml               # actionlint, shellcheck, node --check, node --test, tsc, forbidden-call scan
│   ├── workflows/self-test.yml        # uses: ./ against examples/flutter-sample (e2e + 5-min interactive smoke)
│   ├── workflows/release.yml          # on tag vX.Y.Z: verify pins, move v1, npm publish --provenance
│   └── dependabot.yml                 # github-actions + npm
├── runner/                            # everything that runs on the runner (reached via github.action_path)
│   ├── lib/common.sh                  # log/group helpers, retry, wait_port, die, AR_STATE dir
│   ├── prepare.mjs                    # validate+normalize inputs and config JSON → $GITHUB_OUTPUT; policy checks
│   ├── secrets.mjs                    # generate viewer key / agent token / internal token; ::add-mask::; write $AR_STATE/secrets.json
│   ├── emulator/{kvm.sh,install.sh,start.sh,wait.sh,settings.sh,snapshot.sh}
│   ├── app/{fingerprint.sh,inject-env.sh,build-flutter.sh,build-gradle.sh,fetch-apk.sh,inspect-apk.sh,install.sh}
│   ├── viewers/ws-scrcpy-web/{start.sh,deny-paths.txt}
│   ├── viewers/ws-scrcpy/{start.sh,deny-paths.txt}
│   ├── session/{gate.mjs,tunnel.sh,publish.mjs,hold.sh,teardown.sh,redact.mjs}
│   ├── agent/{agent-api.mjs,adb.mjs,ui-xml.mjs,ime.sh}
│   └── e2e/{record.sh,maestro.sh,flutter-it.sh,custom.sh,collect.sh,summary.mjs}
├── shared/
│   ├── sealed-box.mjs                 # seal()/unseal(): used by runner/publish.mjs and cli/up
│   └── protocol.mjs                   # constants: artifact names, run-name prefix, template version, limits
├── cli/
│   ├── bin/android-remote.mjs
│   ├── src/{args.mjs,ui.mjs,errors.mjs,gh.mjs,git.mjs,session-store.mjs,template.mjs,states.mjs}
│   ├── src/commands/{init,doctor,up,status,down,mcp}.mjs
│   ├── templates/caller-workflow.yml
│   └── test/self-check.test.mjs       # node --test: sealed-box round trip, args, template render
├── examples/flutter-sample/           # tiny Flutter app: 2 flavors, .maestro/ flows, integration_test/
├── scripts/{release-pin.mjs,check-forbidden.mjs,agent-smoke.sh,log-leak-scan.sh}
├── docs/{getting-started,cli,workflow-inputs,e2e,agent-api,security,protocol,costs,troubleshooting}.md
├── package.json                       # name "android-remote", bin, files [cli, shared], engines node>=20
├── LICENSE (MIT) · README.md · CHANGELOG.md · .gitignore
```

The npm package is the repo root (`files: ["cli", "shared"]`), so the CLI and the runner share `shared/` without copying it. The runner reaches scripts through `${{ github.action_path }}/runner/...`. Sub-actions reach them through `${{ github.action_path }}/../../runner/...` (GitHub downloads the whole repo for any action ref).

**Runtime dependencies:** CLI: `@modelcontextprotocol/sdk` only, lazy-loaded by `mcp`. Runner: none (Node 20 built-ins). Dev: `typescript` (for `tsc --noEmit`), plus `actionlint` and `shellcheck` binaries in CI.

## 4. Integration contract

### 4.1 Root action inputs (`action.yml`)

| Group | Inputs (default) |
|---|---|
| mode | `mode` (`interactive`\|`e2e`, req.), `session` (auto from `run_id` if empty), `client_public_key` (`''`), `template_version` (`''`) |
| app | `app_source` (`flutter`\|`gradle`\|`apk`, `flutter`), `app_dir` (`.`), `flutter_version` (`''` → `flutter-version-file: <app_dir>/pubspec.yaml`, else `stable`), `build_mode` (`debug`), `flavor`, `target`, `dart_defines` (newline-separated K=V, **non-secret**), `gradle_task` (`assembleDebug`), `apk_glob` (`**/outputs/apk/**/*.apk`), `apk` (`url:…`\|`release:<tag>/<asset>`\|`artifact:<run-id>/<name>`), `pre_build` (repo-relative script path), `dotenv_path` (comma-separated), `dotenv` (**secret**, from `secrets.ANDROID_REMOTE_DOTENV`), `dart_defines_json` (**secret**), `launch` (`true`) |
| device | `api_level` (`34`), `system_image` (`google_apis`), `device` (`pixel_7`), `locale` (`''`, e.g. `ar-EG`), `cores` (`auto`), `ram_mb` (`auto`), `boot_timeout` (`600`) |
| session | `minutes` (`30`, 1–300), `idle_minutes` (`10`, 0 = off), `agent` (`false`), `viewer` (`ws-scrcpy-web`\|`ws-scrcpy`), `max_size` (`1024`), `bitrate` (`4M`), `max_fps` (`30`), `tunnel_protocol` (`quic`) |
| e2e | `e2e` (`maestro`\|`flutter`\|`custom`), `e2e_path` (`.maestro` / `integration_test`), `e2e_args` (extra args, validated), `e2e_command` (custom; path to script in repo), `record` (`on-failure`\|`always`\|`never`), `artifact_retention_days` (`14`), `export_apk` (`false`) |
| misc | `cache` (`true`) |

Outputs: `result`, `tests`, `failures`, `junit-path`, `artifact-url`, `session`, `package`.

Rule for all scripts: **never interpolate `${{ inputs.* }}` inside `run:`**. Inputs go through `env:` and are validated by `prepare.mjs`. This prevents script injection and keeps values out of the expanded script text. Secret-carrying inputs (`dotenv`, `dart_defines_json`) come from `secrets.*`, so GitHub masks them automatically. They are only ever written to files with `umask 077`.

### 4.2 Reusable workflow (`.github/workflows/android-remote.yml`)

`on: workflow_call` with the same inputs as the root action (strings), plus `config` (JSON string). Secrets: `ANDROID_REMOTE_DOTENV`, `ANDROID_REMOTE_DART_DEFINES_JSON` (optional; `secrets: inherit` works).
One job, `session`:
- `runs-on: ubuntu-latest`
- `timeout-minutes: ${{ fromJSON(inputs.minutes) + 50 }}` (50 covers build + boot + teardown, capped by the 6 h job limit)
- `permissions: { contents: read, actions: read }`
- Steps: `actions/checkout` (SHA-pinned), then `uses: melbanna40/android-remote@<pinned>` with every input forwarded and `config` merged by `prepare.mjs`. Precedence: config JSON > explicit input > default. Unknown config keys are an error.

### 4.3 Caller template (written by `init`)

```yaml
# android-remote-template: 1   (do not remove; the CLI checks it)
name: android-remote
run-name: android-remote ${{ inputs.session }} · ${{ inputs.mode }}
on:
  workflow_dispatch:
    inputs:
      session:           { description: Session id (set by the CLI), required: true }
      client_public_key: { description: CLI public key (safe to show), default: "" }
      mode:              { description: interactive | e2e, default: interactive }
      minutes:           { description: Session length in minutes (1-300), default: "30" }
      app_source:        { description: flutter | gradle | apk, default: "<from init>" }
      apk:               { description: "url:… | release:<tag>/<asset> | artifact:<run-id>/<name>", default: "" }
      flavor:            { description: Flutter flavor, default: "<from init>" }
      target:            { description: Flutter entry point, default: "<from init>" }
      config:            { description: Advanced options as JSON (see docs/workflow-inputs.md), default: "{}" }
permissions: { contents: read, actions: read }
concurrency: { group: "android-remote-${{ inputs.session }}", cancel-in-progress: true }
jobs:
  remote:
    uses: melbanna40/android-remote/.github/workflows/android-remote.yml@v1
    with: { session: ..., client_public_key: ..., mode: ..., minutes: ..., app_source: ..., apk: ..., flavor: ..., target: ..., config: ..., app_dir: "<from init>", pre_build: "<from init>", dotenv_path: "<from init>", template_version: "1" }
    secrets: inherit
```

### 4.4 Direct use in your own job (E2E PR check)

```yaml
jobs:
  android-e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@<sha>
      - uses: melbanna40/android-remote@v1
        with: { mode: e2e, e2e: maestro, e2e_path: .maestro, app_source: flutter, flavor: development, target: lib/main_development.dart, dotenv: "${{ secrets.ANDROID_REMOTE_DOTENV }}", dotenv_path: .env.development }
```

## 5. Session state machine

### 5.1 CLI view (`cli/src/states.mjs`)

```text
            dispatch ok          run visible            run in_progress          sealed artifact found
 IDLE ──▶ DISPATCHING ──▶ WAITING_RUN ──▶ QUEUED ──────▶ PREPARING ───────────▶ LIVE ──┬─▶ ENDED (run completed: success, deadline/idle/down)
   │          │ fail→ERROR(3/4)  │ >10 min→ERROR(4)            │ run completed before artifact      ├─▶ CANCELLED (down / run cancelled)
   │          ▼                  ▼                              ▼                                    └─▶ EXPIRED (expiresAt passed, run status unknown)
   └── e2e mode: … PREPARING ──▶ TESTING (no artifact expected) ──▶ PASSED | FAILED (conclusion) ──▶ results downloaded with --wait
                                                                  FAILED(stage = last failed step name, run URL, logs artifact name)
```

Polling: run lookup every 3 s (≤ 10 min), run status every 5 s, artifact list every 5 s once `in_progress`. Waiting for a live session times out after `minutes + 50`.
Run lookup: (1) parse the run URL that `gh workflow run` prints ("returned if available", gh ≥ 2.x); (2) otherwise `gh run list --workflow android-remote.yml --event workflow_dispatch --json databaseId,displayTitle,createdAt`, matching `displayTitle == "android-remote <session> · <mode>"`.

### 5.2 Runner view (stages = step names prefixed `[ar:<stage>]` in action.yml)

`prepare → emulator-start (background) → app (build | cache hit | fetch) → emulator-wait → install → { interactive: viewer → gate → tunnel → publish → LIVE(hold) | e2e: record → test → collect → upload → summary } → teardown (if: always())`

`hold.sh` loop (every 15 s): end when `now ≥ deadline`, or when `idle_minutes > 0` and `now − mtime($AR_STATE/last-activity) ≥ idle`, or when `$AR_STATE/stop` exists (created by the gate after the auth-failure lockout). Exit 0 for normal ends. `down` = `gh run cancel <id>`: GitHub stops the hold step, and the `always()` teardown then runs (kills the tunnel first, uploads redacted logs). If the run hasn't stopped after 60 s, `down` calls `POST /actions/runs/{id}/force-cancel`.

### 5.3 Local session file

`<git-common-dir>/android-remote/sessions/<session>.json`, mode 0600, directory 0700. Fields: `session, repo, ref, sha, runId, runUrl, mode, createdAt, state, privateKeyJwk` (removed after unseal), `url, viewerKey, agentToken, expiresAt`. `status` hides `viewerKey`/`agentToken` unless `--show-key`. `down` deletes the file. No git repo → `~/.android-remote/sessions/`.

## 6. Link and key delivery (protocol)

1. CLI: `generateKeyPairSync('x25519')`. It sends the public key as a base64url raw 32-byte string in `client_public_key` (public by design). The private key goes in the session file.
2. Runner `secrets.mjs` (first step after prepare): `viewerKey = randomBytes(32)`, `agentToken = randomBytes(32)` (only if `agent`), `internalToken = randomBytes(32)`, all base64url. It prints `::add-mask::<value>` for each, then writes `$AR_STATE/secrets.json` (0600). They never go to `$GITHUB_ENV`, `$GITHUB_OUTPUT` or stdout.
3. `tunnel.sh` captures `https://<rand>.trycloudflare.com` from `cloudflared.log`, then immediately runs `::add-mask::<host>` and `::add-mask::<url>`. It writes the URL to `$AR_STATE/url` only.
4. `publish.mjs` builds the plaintext `{v:1, session, runId, url, viewerKey, agentToken, expiresAt, package, apiLevel}` and seals it:
   - ephemeral X25519 keypair `e`; `shared = diffieHellman(e.priv, client.pub)`
   - `key = hkdfSync('sha256', shared, salt=utf8(session), info=utf8('android-remote/v1') ‖ e.pubRaw ‖ client.pubRaw, 32)`
   - AES-256-GCM, 12-byte random nonce, AAD = utf8(session)
   - file `session.sealed.json` = `{v:1, session, epk, nonce, ct, tag}` (base64url)
   It then uploads the artifact `android-remote-session-<session>` (retention-days 1).
5. CLI unseals, checks that `session` matches, deletes the private key, then stores and prints the link.
6. **No client key** (manual dispatch from the UI): `prepare.mjs` reads `github.event.repository.private`. If the repo is private, `publish.mjs` writes the link and key to `$GITHUB_STEP_SUMMARY` (visible only to people with repo read access). If it is public, `prepare` fails before any build with: "Interactive sessions in public repositories must be started with `npx android-remote up` (the link is delivered encrypted)."
7. Logs artifact (`android-remote-logs-<session>`, always uploaded, 3 days): `gate.log`, `viewer.log`, `cloudflared.log`, `emulator.log`, `agent.log`, each passed through `redact.mjs` first. It replaces every value from `secrets.json`, the tunnel host, and `k=`/`Authorization`/`Cookie` patterns with `***`.
8. Run name and job summary contain only the session id and non-secret settings.

## 7. Agent API

### 7.1 Transport and auth
- Base URL `<tunnel>/agent/v1`. `Authorization: Bearer <agentToken>`, compared in constant time by the gate. The viewer cookie is **not** accepted on `/agent/*`, and the agent token is **not** accepted on viewer routes.
- The gate forwards to `127.0.0.1:4310` with `X-AR-Internal: <internalToken>`. `agent-api.mjs` refuses any request without it. This blocks the device from calling the agent API (or the viewer) through `10.0.2.2`, which maps to the runner's loopback.
- JSON in/out: `{ok:true,data}` / `{ok:false,error:{code,message}}`. Error codes: `bad_request`, `unauthorized`, `rate_limited`, `adb_failed`, `timeout`, `too_large`.

### 7.2 Endpoints

| Method + path | Body / query | Does |
|---|---|---|
| `GET /device` | – | model, sdk, `wm size`, `wm density`, rotation, foreground package/activity, installed package of the session app |
| `POST /tap` | `{x,y}` px or `{nx,ny}` 0..1 | `input tap` |
| `POST /long-press` | `{x,y\|nx,ny, durationMs≤5000}` | `input swipe x y x y d` |
| `POST /swipe` | `{x1,y1,x2,y2 \| nx1…, durationMs≤5000}` | `input swipe` |
| `POST /type` | `{text ≤ 2000, submit?}` | ASCII → `input text` (escaped); otherwise ADBKeyBoard `ADB_INPUT_B64` broadcast (IME installed by `ime.sh` when `agent: true`); `submit` → ENTER |
| `POST /key` | `{key}`: `BACK\|HOME\|ENTER\|DEL\|TAB\|ESCAPE\|APP_SWITCH\|MENU\|DPAD_UP/DOWN/LEFT/RIGHT/CENTER\|VOLUME_UP/DOWN\|POWER` or int 0–300 | `input keyevent` |
| `GET /screenshot` | – | `exec-out screencap -p` → `image/png` |
| `GET /ui` | `format=json\|xml` | `uiautomator dump` → raw XML or flattened `[{text,desc,id,cls,pkg,bounds:[l,t,r,b],center:[x,y],clickable,enabled,focused,checked,scrollable}]` |
| `POST /install` | raw `application/vnd.android.package-archive` body ≤ 100 MB, or `{url}` (https) | `install -r -t -g` → `{package}` |
| `POST /launch` | `{package, activity?}` | `monkey -p <pkg> -c android.intent.category.LAUNCHER 1` or `am start -n` |
| `POST /stop` / `POST /clear-data` | `{package}` | `am force-stop` / `pm clear` |
| `POST /open-link` | `{url, package?}` | `am start -W -a android.intent.action.VIEW -d <url> [<pkg>]` |
| `GET /logcat` | `lines≤5000 (500)`, `level=V\|D\|I\|W\|E\|F`, `tag`, `package` (→ `--pid`), `since` (epoch ms) | `logcat -d -v threadtime …` → `text/plain` |
| `POST /logcat/clear` | – | `logcat -c` |
| `POST /rotate` | `{orientation: portrait\|landscape}` | `settings put system user_rotation` |

### 7.3 Command safety (`adb.mjs`)
- Use `execFile('adb', ['-s', serial, ...])` only, never `exec`.
- `adb shell` joins its arguments into one string that the **device** shell parses. Every device-side argument therefore goes through `shq()` (wrap in single quotes, `'` → `'\''`) before being passed.
- Validation: package `^[A-Za-z][A-Za-z0-9_]*(\.[A-Za-z0-9_]+)+$`; activity `^[A-Za-z0-9_.$/]+$`; URL ≤ 2048, `^[a-zA-Z][a-zA-Z0-9+.-]*:`, no control characters; numbers clamped to screen bounds.
- `input text` escaping: space → `%s`, then `shq()`. Non-ASCII goes through the base64 broadcast, so it can't be injected.
- Per-command timeouts: 15 s default, install 180 s. At most 1 install at a time; other commands are serialized through a small queue (adb input ordering matters).

### 7.4 Limits (enforced in `gate.mjs`)
- Auth failures per client IP (`CF-Connecting-IP`): 10/min → 429 for 5 min. 200 total failures in a session → the gate writes `$AR_STATE/stop` and the session ends (the link has probably leaked).
- Agent: token bucket of 10 req/s, burst 20. `/screenshot` and `/ui` max 3/s. `/install` max 10 per session-hour. Bodies ≤ 64 KB except `/install` (≤ 100 MB, the Cloudflare request body limit).
- Viewer: ≤ 3 concurrent WebSockets.
- Every authenticated request touches `$AR_STATE/last-activity` (for the idle timeout).

### 7.5 MCP bridge (`cli/src/commands/mcp.mjs`)
- `@modelcontextprotocol/sdk` stdio server. Tools: `device_info, tap, long_press, swipe, type_text, press_key, screenshot` (returns image content), `ui_tree, install_apk` (local path → upload, or url), `launch_app, stop_app, clear_app_data, open_link, logcat, rotate, session_status`.
- It uses the newest live session with `agent: true` from the session store, or `--session <id>`. If there is no live session, tools return a clear error that suggests `npx android-remote up --agent`.
- Registration: `claude mcp add android-remote -- npx -y android-remote mcp`. Docs also give Hermes/OpenCode snippets and raw curl examples.

## 8. Emulator

- **Host**: GitHub `ubuntu-latest` (24.04, x86_64). Public repos get a 4 vCPU / 16 GB standard runner; private repos get a smaller one (2 vCPU / 7–8 GB; `prepare` reads `nproc`/`free`). Defaults: `cores = min(4, nproc−1)`, `ram_mb = 4096` if total memory ≥ 14 GB, else `3072`.
- **KVM**: the udev rule from the android-emulator-runner README (`KERNEL=="kvm", GROUP="kvm", MODE="0666"…`, `udevadm control --reload-rules && udevadm trigger --name-match=kvm`). It fails fast if `/dev/kvm` is missing (e.g. an ARM runner).
- **Images**: `system-images;android-<api>;<system_image>;x86_64` through the preinstalled `sdkmanager` (accepting licenses). Supported API levels are **30–36**. CI tests 30, 34 and 35; 36 is best-effort (the 16 KB page-size variants `google_apis_ps16k` are allowed as `system_image`).
- **ARM64 vs x86_64**: only x86_64 images are supported. GitHub's Linux arm64 runners have no usable KVM for the emulator, and arm64 images on an x86 host aren't supported by the modern emulator. APKs with only `arm64-v8a` native libraries still run on API 30+ x86_64 images through the image's ARM translation, but slower. `inspect-apk.sh` warns about this. Flutter's default `flutter build apk` includes `x86_64`, so there's no problem for Flutter apps.
- **Play Services**: the default is `google_apis` (Play services yes, Play Store no, root yes). `google_atd`/`aosp_atd` are allowed for faster headless E2E. `google_apis_playstore` is accepted but documented as "no root, so `locale`/`rotate` may fail".
- **Flags**: `-no-window -gpu swiftshader_indirect -noaudio -no-boot-anim -camera-back none -camera-front none -no-metrics -accel on -cores N -memory M`, plus a snapshot load (`-snapshot ar-clean -no-snapshot-save`) when the cache hits. On a miss: cold boot, then `adb emu avd snapshot save ar-clean` after `settings.sh`, then the cache is saved.
- **Boot wait**: `adb wait-for-device`, then poll `sys.boot_completed == 1` and `pm path android` succeeds and `dev.bootcomplete == 1`, until `boot_timeout`.
- **settings.sh**: animation scales set to 0, stay-awake while charging, `show_touches 1` (interactive only), locale (`setprop persist.sys.locale` + zygote restart, root), screen unlocked (`wm dismiss-keyguard`), hidden-API-free.
- **Devices**: `avdmanager` profile ids. `prepare` validates against `avdmanager list device -c`. Documented set: `pixel_7` (default), `pixel_4a`, `pixel_6`, `pixel_8`, `medium_phone`, `small_phone`, `pixel_tablet`.

## 9. Viewer, gate, tunnel

- **Viewer adapter contract** (`runner/viewers/<name>/start.sh`): inputs `VIEWER_PORT=8000`, `ADB_SERIAL`, `PUBLIC_HOST`, `MAX_SIZE`, `BITRATE`, `MAX_FPS`. It must listen on `127.0.0.1:$VIEWER_PORT` (or the docker bridge), write `$AR_STATE/viewer-entry` (the path to open for the device stream), and ship `deny-paths.txt` (path prefixes the gate blocks with 403).
- **ws-scrcpy-web** (default): `docker run -d --name ar-viewer -p 127.0.0.1:8000:8000 -v $AR_STATE/viewer:/data -e … <image>@sha256:<digest>` in bridge mode (not host mode, because host mode would make every gate request "loopback = admin"). ADB reaches the emulator through `socat TCP-LISTEN:5557,bind=<docker0-ip>,fork TCP:127.0.0.1:5555` and the viewer's manual/wireless add (`<docker0-ip>:5557`). `config.json` gets `allowedHosts: [PUBLIC_HOST]`. The gate forwards the original `Host`, so the viewer starts **after** the tunnel host is known. The entry path is the "stream link" (`#!action=stream&udid=…&player=webcodecs`). **[SPIKE S1]** confirms: the image tag/digest, the config path inside the container, how to add a device without the UI (API or pre-seeded config), the stream-parameter names for size/bitrate/fps, and the admin/updater/scan routes for `deny-paths.txt`.
- **ws-scrcpy** (fallback): build the pinned commit with Node 20 (`npm ci && npm run dist`, cached by commit), run `node dist/index.js` with `WS_SCRCPY_CONFIG` (port 8000, http only). Entry: the device list → "proxy over adb" stream (per its README known issue for emulators).
- **gate.mjs** (Node built-ins only, ~350 lines): HTTP reverse proxy plus WebSocket upgrade piping (`net.connect` + raw upgrade head).
  - Viewer auth: `?k=` → `Set-Cookie: ar=<hmac(viewerKey, session)>; HttpOnly; Secure; SameSite=Strict; Path=/`, then `302` to the same path without `k` (other query params kept).
  - Agent auth: Bearer.
  - Headers on every response: `Referrer-Policy: no-referrer`, `Cache-Control: no-store`, `X-Frame-Options: DENY`.
  - Denylist from `deny-paths.txt`. Health check at `/__ar/healthz` (unauthenticated, returns only `ok`). Before the upstream is ready, `503` with a small "starting…" page.
  - Logs: method, path without query, status and ms only. Activity file touch on authenticated requests.
- **tunnel.sh**: download `cloudflared-linux-amd64` at a pinned version (2026.10.7 at the time of writing) and check its SHA-256. Run `cloudflared tunnel --no-autoupdate --protocol $P --url http://127.0.0.1:3199 --logfile …`. Wait up to 120 s for the URL. On failure, retry with `http2`, up to 3 attempts. Mask the URL and host.
- **Latency defaults**: max size 1024 px, 4 Mbit/s H.264, 30 fps. The browser decodes with WebCodecs (Chrome, Edge, Firefox 130+, Safari 17+ best-effort). Users on slow links can pass `config.max_size=720, bitrate=2M, max_fps=20`.

## 10. E2E mode outputs

| Item | Path in artifact | Source |
|---|---|---|
| JUnit | `junit/*.xml` | Maestro `--format junit --output junit/maestro.xml`; Flutter `--file-reporter json:…` → `tojunit` (`junitreport`, pinned); custom: any `*.xml` the command writes to `$AR_JUNIT_DIR` |
| Screenshots | `screenshots/` | Maestro `--test-output-dir` / `--debug-output`, plus a final `screencap` after the run (always) |
| Recording | `recording/segment-NNN.mp4` | `record.sh`: `screenrecord --time-limit 180 --bit-rate 4000000` loop; SIGINT on stop; segments pulled. With `record: on-failure` they are discarded when the run passes |
| Logcat | `logcat/full.txt`, `logcat/app.txt` | `logcat -d -v threadtime` (buffer raised to 16 MB at boot with `logcat -G 16M`); app = filtered by the app pid(s) seen |
| Device | `device.json` | api, image, profile, emulator version, app package/version |
| Maestro debug | `maestro/` | `--debug-output` |
| Summary | `summary.md` | also written to `$GITHUB_STEP_SUMMARY` |

Artifact: `android-remote-e2e-<run_id>-<run_attempt>`, retention `artifact_retention_days`. The step fails if `failures > 0` or the runner exited non-zero. Collection and upload run with `if: always()`. `export_apk: true` additionally uploads the APK. That is off by default because APKs embed build-time env, and envied obfuscation is not encryption.

## 11. Caching

| Cache | Key | Path | Notes |
|---|---|---|---|
| System image | `ar-sysimg-v1-<api>-<image>-x86_64` | `$ANDROID_HOME/system-images/android-<api>/<image>/x86_64` | ~1–1.5 GB |
| AVD + snapshot | `ar-avd-v1-<api>-<image>-<device>-<emulatorVersion>-<locale>` | `~/.android/avd/ar.avd`, `~/.android/avd/ar.ini` | ~1–2 GB. If the snapshot doesn't match, the emulator cold-boots automatically |
| Gradle | managed by `gradle/actions/setup-gradle` (SHA-pinned) | | read-only on non-default branches |
| Pub / Flutter SDK | `subosito/flutter-action` `cache: true` | | |
| APK | `ar-apk-v1-<fingerprint>` | `$AR_STATE/app.apk` | fingerprint = sha256(`git rev-parse HEAD:<app_dir>` tree id ‖ build inputs ‖ sha256(dotenv) ‖ sha256(dart_defines_json) ‖ flutter version) |
| Tools | `ar-tools-v1-<versions>` | cloudflared, maestro, junitreport, ws-scrcpy dist | |

Saves happen only on a cache miss (`actions/cache/restore` + `actions/cache/save`). `cache: false` skips everything. Docs warn that the repository cache budget (10 GB) fits about two API/image combinations; a large matrix evicts.

## 12. Runner minutes and cost

- Linux standard runners bill at 1×. They are **free for public repositories**. Private repositories use the account's included minutes (Free 2,000/mo, Pro/Team 3,000/mo at the time of writing), then per-minute billing.
- A typical interactive session costs `minutes + ~6` with warm caches (prebuilt APK) or `+ ~12` on a cold Flutter build. Idle timeout and `down` are the main savings.
- A job is capped at 6 h by GitHub, so `minutes ≤ 300` (+50 margin = 350).
- Artifact storage counts toward private-repo storage quotas (500 MB on Free). That's why `record: on-failure`, 14-day E2E retention, 1-day session artifacts and 3-day logs artifacts are the defaults.
- `docs/costs.md` gives a worked example per plan and recommends E2E on `develop` pushes or a `run-e2e` label rather than on every PR in private repos.

## 13. Versioning and release

- SemVer tags `vX.Y.Z` on `main` (Gitflow: `release/vX.Y.Z` from `develop`). The floating major tag `v1` is moved by `release.yml`. npm version = tag version (`android-remote@X.Y.Z`).
- **Self-reference pinning**: on `develop`, the reusable workflow uses `melbanna40/android-remote@develop`. `scripts/release-pin.mjs X.Y.Z` (run on the release branch) rewrites it to `@vX.Y.Z` and sets `package.json` version. `release.yml` refuses to publish if any `@develop` ref remains.
- `init` writes `@v1` by default. `init --pin` writes `@<full-sha> # vX.Y.Z`.
- The template carries `# android-remote-template: 1`, and the reusable workflow receives `template_version`. When it is older than supported, `prepare` emits a warning naming `npx android-remote init --force`. A breaking input change means v2.
- npm publish with `--provenance` from GitHub Actions (OIDC). `CHANGELOG.md` uses Keep a Changelog.

## 14. Security model

Trust boundaries: (a) the user's machine plus the gh token, (b) GitHub Actions (logs and artifacts are public in public repos), (c) Cloudflare (terminates TLS, so it sees the traffic), (d) the runner VM (ephemeral), (e) the emulator (untrusted: runs the app under test and anything the agent installs).

| # | Threat | Mitigation |
|---|---|---|
| T1 | Link leakage (chat paste, history, Referer, screen share) | Per-session 256-bit key; `?k=` swapped for a cookie and stripped from the URL; `Referrer-Policy: no-referrer`; short default (30 min) plus idle end (10 min); `down`; the link dies with the run |
| T2 | Key/URL in public logs, summaries or artifacts | Generated on the runner; never an input/env/output; `::add-mask::`; delivered as ciphertext; logs redacted before upload; public + no client key → refuse; CI log-leak scan (V5) |
| T3 | Brute force or scanning of the tunnel | 256-bit keys; per-IP 429; 200 failures end the session; unguessable trycloudflare host |
| T4 | Agent endpoint abuse (stolen token, prompt-injected agent) | Separate token; no shell/push/forward endpoints; argv + device-side quoting; strict validation; rate limits; body caps. Impact is limited to the ephemeral emulator |
| T5 | Secrets in builds | Only through repository secrets with fixed names, passed as masked inputs, written with umask 077, never echoed; `dart_defines` input documented as non-secret; APK export off by default |
| T6 | Tunnel exposure window | Tunnel starts only after the gate is up (the gate answers 503 until the viewer is ready); killed first in teardown; host is random per session; docs: Cloudflare can see plaintext, so never use production accounts in sessions |
| T7 | Fork PRs and untrusted triggers | Interactive mode refused on `pull_request*` events; docs forbid `pull_request_target` with this action; forks get no secrets |
| T8 | Supply chain | Actions pinned by SHA; binaries pinned by version + SHA-256; viewer image pinned by digest; Dependabot; npm provenance |
| T9 | Device → runner loopback (10.0.2.2) | Agent API and gate upstreams require the internal token; nothing sensitive listens unauthenticated |
| T10 | Script injection through inputs or config | Inputs only through `env:`; `prepare.mjs` allowlists keys and validates formats (flavor/target/paths/regex); `pre_build` and `e2e_command` must be repo-relative script paths, not inline shell |
| T11 | Local secrets on the dev machine | Session file 0600 in `.git/`; private key deleted after unseal; `status` hides keys by default |
| T12 | Cost abuse | Dispatch needs write access (GitHub permission model); `minutes` capped; concurrency group per session; `down --all` |
| T13 | GPL tools | ws-scrcpy-web (GPL-3) and ADBKeyBoard (GPL-2) are downloaded at runtime and run as separate programs, never vendored or linked; noted in README "Third-party software" |

## 15. Risks and spikes

| Spike | Question | Exit criterion |
|---|---|---|
| S1 | Does ws-scrcpy-web stream and accept input on API 34/35 through `gate.mjs` + a quick tunnel, in docker bridge mode? Which routes need denying? | Video in Chrome and Firefox, taps work; `deny-paths.txt` written. Otherwise switch the default to ws-scrcpy and record why |
| S2 | Are v4 artifacts listable and downloadable through the REST API/`gh run download` **while the run is in progress**? Does `gh workflow run` print the run URL? | Both confirmed from a test repo. Otherwise fall back to a check-run summary carrying the ciphertext (`checks: write`) and update plan §6 |
| S3 | Does ADBKeyBoard input Arabic into a Flutter `TextField` on API 34? | Arabic text appears. Otherwise use clipboard + paste through the viewer and document ASCII-only `type` |
| S4 | Does `screenrecord` work under `swiftshader_indirect` on API 30/34/35? | Playable mp4. Otherwise scrcpy v5 `--record --no-window` |
| S5 | Snapshot caching: is the time saved worth the cache size? | Warm boot < 60 s vs cold; otherwise cache only the system image |

Other risks: the quick-tunnel URL can go down mid-session (no SLA), so the CLI `status` shows the run state and the user restarts. GitHub runner image changes (the emulator binary moves): the emulator version is included in the cache key.

## 16. Phases

1. **P0 Foundations + spikes** (repo scaffold, CI lint, S1–S5).
2. **P1 Emulator + app** (sub-action `setup-emulator`, build/fetch/install).
3. **P2 Interactive session** (secrets, gate, viewer, tunnel, publish, hold, teardown, root action, reusable workflow).
4. **P3 E2E** (record, runners, collect, summary, `actions/e2e`).
5. **P4 Agent API** (adb wrapper, API, IME).
6. **P5 CLI** (init, doctor, up, status, down, mcp).
7. **P6 Docs + release**.
8. **P7 Verification** (external test repo, browser, E2E, agent smoke, leak scan, Roquia dry run).

---

## Appendix A — Roquia integration

Facts from `roquia-app` (read 2026-11-05):
- Private monorepo; the Flutter app lives in `mobile/`. Flavors `development` (`com.rouqia.app.dev`, entry `lib/main_development.dart`) and `production` (`com.rouqia.app`, `lib/main_production.dart`) via flutter_flavorizr.
- `envied` reads `.env.development` and `.env.production` at `build_runner` time. Both files are gitignored, as is `lib/core/env/*.g.dart`. `mobile.yml` CI writes both files and then runs `dart run build_runner build -d`. The Flutter version is pinned to 3.47.5 in CI; Java 17.
- Firebase client config is committed. FCM works on the `google_apis` image.

### A.1 One-time setup (done by the developer; the CLI never commits)

```bash
cd roquia-app
npx android-remote init --app-dir mobile --app-source flutter \
  --flavor development --target lib/main_development.dart \
  --pre-build mobile/tool/android_remote_prebuild.sh \
  --dotenv-path mobile/.env.development,mobile/.env.production
# then the developer:
#   add mobile/tool/android_remote_prebuild.sh  →  cd mobile && dart run build_runner build -d
#   gh secret set ANDROID_REMOTE_DOTENV   (content: API_URL=https://<staging-host>\nSENTRY_DSN=)
#   commit .github/workflows/android-remote.yml + the script on a feature branch → PR → develop/default branch
```

`flutter_version` comes from `config` (`"flutter_version":"3.47.5"`), matching `mobile.yml`, or from `pubspec.yaml` if it declares `environment.flutter`.

### A.2 Daily use

- Interactive: `npx android-remote up --minutes 20` (defaults from init: development flavor). Arabic UI check: `--config '{"locale":"ar-EG"}'`.
- Agents (Claude Code / GLM on Hermes / Orca workers): `npx android-remote up --agent --minutes 30 --json`, then the MCP tools. This replaces the local Genymotion device for UI verification packets, and the Orca browser can open the viewer link.
- E2E PR check (Maestro flows in `mobile/.maestro/`, selectors by `Semantics(identifier:)`): a `mobile-e2e.yml` job using `melbanna40/android-remote@v1` with `mode: e2e`, triggered on pushes to `develop` and on PRs labelled `run-e2e`, to save private-repo minutes.

### A.3 Roquia-specific notes

- **The API must be reachable from GitHub's network.** `10.0.2.2:8080` (the laptop's local backend) does not work on a runner. Sessions use the staging URL from `ANDROID_REMOTE_DOTENV`. Until staging exists, use a cloudflared tunnel to the local API in that secret, or the mock proxy.
- **Minutes**: private repo, so sessions are billed. The recommended defaults are `minutes 20`, `idle_minutes 10`, and E2E not on every PR.
- `workflow_dispatch` only works after the caller file is on the default branch. Under Gitflow it reaches `develop`/default through the normal PR.
- Test accounts: `*@roquia.test` seeded on staging. Never production credentials (T6).
- Maestro `inputText` with Arabic is limited (**[SPIKE S3]** covers the agent path). Flows should tap by identifier and type ASCII, or use pre-seeded data.
