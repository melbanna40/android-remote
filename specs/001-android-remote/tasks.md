# 001 — android-remote: tasks

Read [spec.md](spec.md) and [plan.md](plan.md) first. Each packet is sized for one implementer run (GLM 5.3 / OpenCode). Each one names its exact files, its job and a **Done when** check that the implementer runs before handing back.

Conventions for every packet:
- Bash: `#!/usr/bin/env bash` + `set -euo pipefail`; source `runner/lib/common.sh`; must pass `shellcheck`.
- Node: ESM `.mjs`, Node 20 built-ins only (runner and `shared/`). The CLI may import `@modelcontextprotocol/sdk` only in `mcp.mjs`. Top line `// @ts-check`, JSDoc types.
- Never put `${{ inputs.* }}` or `${{ secrets.* }}` inside a `run:` script body. Pass them via `env:`.
- Never print a value from `$AR_STATE/secrets.json`, the tunnel URL/host, or any `dotenv` content.
- Third-party actions are pinned by full commit SHA with a `# vX` comment. Binaries are pinned by version plus a SHA-256 kept in `shared/protocol.mjs` (CLI) or `runner/lib/versions.sh` (runner).
- `[P]` = can run in parallel with the other `[P]` packets of the same phase once its dependencies are done.
- Test policy (owner): no unit-test files, **except** one trivial `node --test` self-check for the CLI (T055). Everything else is proven by the verification packets (P7).

---

## P0 — Foundations and spikes

### T001 Repo scaffold
- Files: `package.json`, `LICENSE` (MIT, "Mohamed ElBanna"), `.gitignore`, `README.md` (stub: one paragraph + "status: pre-release"), `CHANGELOG.md` (Keep a Changelog, `## [Unreleased]`), `.editorconfig`.
- Do: `package.json` → `name: android-remote`, `version: 0.0.0`, `type: module`, `bin: {"android-remote": "cli/bin/android-remote.mjs"}`, `files: ["cli", "shared"]`, `engines.node: ">=20"`, `dependencies: {"@modelcontextprotocol/sdk": "<latest 1.x>"}`, `devDependencies: {"typescript": "<latest 5.x>"}`, scripts `check` (`node scripts/check-syntax.mjs`), `typecheck` (`tsc -p jsconfig.json`), `selfcheck` (`node --test cli/test/`). Add `jsconfig.json` (`checkJs`, `noEmit`, `module: nodenext`, `include: ["cli","shared","runner/**/*.mjs","scripts"]`).
- Done when: `npm install` works and `npm run typecheck` runs (an empty project passes).

### T002 [P] Shared constants
- Files: `shared/protocol.mjs`.
- Do: export `PROTOCOL_VERSION = 1`, `TEMPLATE_VERSION = 1`, `RUN_NAME_PREFIX = 'android-remote'`, `artifactNames = { session: s => \`android-remote-session-${s}\`, logs: s => …, e2e: (runId, attempt) => … }`, `limits` (minutes 1–300, idle 0–120, text 2000, logcat lines 5000, upload 100 MB), `EXIT = {OK:0, FAIL:1, USAGE:2, PREREQ:3, REMOTE:4}`, `WORKFLOW_FILE = 'android-remote.yml'`.
- Done when: `node -e "import('./shared/protocol.mjs').then(m=>console.log(Object.keys(m)))"` prints them.

### T003 [P] Sealed box
- Files: `shared/sealed-box.mjs`.
- Do: implement plan §6 exactly: `generateClientKeyPair() → {publicKey: b64url raw32, privateKeyJwk}`, `seal(plainObj, clientPubB64u, session) → sealedObj`, `unseal(sealedObj, privateKeyJwk, session) → plainObj`. Use `crypto.generateKeyPairSync('x25519')`, export the raw public key from the JWK `x`, `crypto.diffieHellman`, `hkdfSync('sha256', …, 32)`, `aes-256-gcm`, 12-byte nonce, AAD = session. Throw `Error('session mismatch')` / `Error('decrypt failed')`.
- Done when: a scratch run of seal→unseal round-trips an object, and a wrong session throws. No stray files are committed (T055 makes the check permanent).

### T004 [P] CI lint workflow
- Files: `.github/workflows/ci.yml`, `scripts/check-syntax.mjs`, `scripts/check-forbidden.mjs`, `.github/dependabot.yml`.
- Do: jobs on `pull_request` + `push` to develop/main: `actionlint` (pinned release binary), `shellcheck` on `runner/**/*.sh scripts/*.sh`, `node scripts/check-syntax.mjs` (`node --check` on every `.mjs`), `npm run typecheck`, `npm run selfcheck`, `node scripts/check-forbidden.mjs`. `check-forbidden` scans `cli/**` and fails on `/git\s+(commit|push|tag)|gh\s+(repo\s+create|release\s+create|secret\s+set)|'(commit|push)'/` (spec AC8), and scans `runner/**`, `action.yml`, `actions/**`, `.github/workflows/android-remote.yml` for `\$\{\{\s*(inputs|secrets)\.` inside `run:` blocks (simple YAML line scan: track whether we are inside a `run: |` block).
- Done when: CI passes on the scaffold; adding `git push` to a dummy CLI file makes `check-forbidden` exit 1.

### T005 [P] Spike S2 — mid-run artifact and run URL
- Files: `spikes/s2/workflow.yml` (kept as documentation), notes appended to `docs/protocol.md` under "Spike S2 results".
- Do: in a scratch public test repo (lead provides it), dispatch a workflow that uploads a tiny artifact and then sleeps 5 min. While it runs: `gh workflow run … ` (record whether it prints the run URL), `gh api repos/{o}/{r}/actions/runs/{id}/artifacts`, `gh run download {id} -n <name>`. Record exact outputs and gh version.
- Done when: the notes say yes/no for (a) URL printed, (b) artifact listed mid-run, (c) artifact downloadable mid-run, with command output excerpts.

### T006 [P] Spike S1 — viewer through gate and tunnel
- Files: `spikes/s1/workflow.yml`, `spikes/s1/NOTES.md`.
- Do: on `ubuntu-latest`, with KVM: boot API 34 `google_apis` x86_64 (inline script, adapted from android-emulator-runner's README), run ws-scrcpy-web in docker bridge mode (plan §9) with a socat bridge to `127.0.0.1:5555`, run a 30-line throwaway Node proxy that preserves `Host`, and expose it with a cloudflared quick tunnel. Print the URL **only in the spike** (private scratch repo). Hold 15 min. Manually: open in Chrome and Firefox, check video and tap. Record: image digest, config path, how the device was added (API call or seeded config), stream URL format and size/bitrate/fps params, and every `/api/*` route seen in DevTools, classified as allow or deny. Repeat with NetrisTV/ws-scrcpy (pinned commit, `npm ci && npm run dist`) and API 35.
- Done when: NOTES.md has a table viewer × API → works/doesn't, plus latency impressions, a proposed `deny-paths.txt` for each viewer, and a recommendation (keep the D4 default or switch).

### T007 [P] Spike S3+S4 — Unicode typing and screenrecord
- Files: `spikes/s3s4/workflow.yml`, `spikes/s3s4/NOTES.md`.
- Do: on an API 30/34/35 emulator: install ADBKeyBoard (pinned release APK + SHA-256), `ime enable`/`ime set`, open the examples app (or the stock Settings search box), `am broadcast -a ADB_INPUT_B64 --es msg <b64("مرحبا Hello")>`, then screenshot. Run `screenrecord --time-limit 20`, pull the file, upload it as an artifact, and check it plays.
- Done when: NOTES.md gives pass/fail per API, with screenshot and video artifacts linked.

### T008 [P] Example Flutter app
- Files: `examples/flutter-sample/**`.
- Do: `flutter create --org dev.androidremote --platforms android flutter_sample`, moved to `examples/flutter-sample`. Add two flavors `development`/`production` (Gradle `productFlavors`, `applicationIdSuffix ".dev"`), entry points `lib/main_development.dart` / `lib/main_production.dart`. The UI has a counter button `Semantics(identifier: 'increment')`, a `TextField` `Semantics(identifier: 'name')` echoing its text, and a deep link `arsample://echo?text=…` shown on screen. The app reads `const String.fromEnvironment('API_URL')` and displays it. Add `.maestro/smoke.yaml` (launch, tap increment twice, assert "2", input "Hello", assert it is shown) and `.maestro/fail.yaml` (asserts text that never appears; tag `failing`). Add `integration_test/app_test.dart` (counter increments). Add `integration_test` to dev_dependencies.
- Done when: locally `flutter build apk --debug --flavor development -t lib/main_development.dart` succeeds and `flutter analyze` is clean.

---

## P1 — Emulator and app

### T010 Common shell library
- Files: `runner/lib/common.sh`, `runner/lib/versions.sh`.
- Do: `common.sh`: `AR_STATE="${RUNNER_TEMP:-/tmp}/android-remote"` (mkdir -p, chmod 700), `log()`, `warn()` (`::warning::`), `die()` (`::error::` + exit 1), `group()`/`endgroup()`, `retry <n> <sleep> cmd…`, `wait_port <port> <timeout_s>`, `wait_http <url> <timeout_s>`, `adb_s()` (= `adb -s "${ADB_SERIAL:-emulator-5554}"`). `versions.sh`: pinned versions + sha256 for cloudflared (linux-amd64), maestro, ADBKeyBoard, ws-scrcpy commit, ws-scrcpy-web image digest (fill from T006), junitreport.
- Done when: `shellcheck runner/lib/*.sh` passes, and `bash -c 'source runner/lib/common.sh; wait_port 1 1'` fails with a clear message.

### T011 [P] KVM + SDK + AVD
- Files: `runner/emulator/kvm.sh`, `runner/emulator/install.sh`.
- Do: `kvm.sh`: write the udev rule (plan §8), reload and trigger, `die` if `/dev/kvm` is missing or not rw. `install.sh` (env `API_LEVEL SYSTEM_IMAGE DEVICE`): `yes | sdkmanager --licenses >/dev/null`, `sdkmanager "emulator" "platform-tools" "system-images;android-$API_LEVEL;$SYSTEM_IMAGE;x86_64"` (skip the image if its directory exists); validate `DEVICE` against `avdmanager list device -c`; `echo no | avdmanager create avd -n ar -k <pkg> -d "$DEVICE" --force` only if `~/.android/avd/ar.avd` is missing; set `hw.keyboard=yes`, `disk.dataPartition.size=6G` in `config.ini`. Print `emulator -version | head -1` to `$AR_STATE/emulator-version`.
- Done when: shellcheck passes; logic is verified in T014's self-test run.

### T012 [P] Emulator start/wait/settings/snapshot
- Files: `runner/emulator/start.sh`, `runner/emulator/wait.sh`, `runner/emulator/settings.sh`, `runner/emulator/snapshot.sh`.
- Do: `start.sh` (env `CORES RAM_MB SNAPSHOT_HIT`): resolve `auto` (plan §8), then `nohup emulator -avd ar <flags from plan §8> [-snapshot ar-clean -no-snapshot-save] > $AR_STATE/emulator.log 2>&1 &`, write the pid to `$AR_STATE/emulator.pid`. `wait.sh` (env `BOOT_TIMEOUT`): plan §8 boot checks; on timeout, tail 200 lines of `emulator.log` inside a group, then `die`. `settings.sh` (env `LOCALE INTERACTIVE`): plan §8 settings, `logcat -G 16M`. `snapshot.sh`: `adb emu avd snapshot save ar-clean` with retry.
- Done when: shellcheck passes; verified in T014.

### T013 [P] App build/fetch/inspect/install
- Files: `runner/app/inject-env.sh`, `runner/app/fingerprint.sh`, `runner/app/build-flutter.sh`, `runner/app/build-gradle.sh`, `runner/app/fetch-apk.sh`, `runner/app/inspect-apk.sh`, `runner/app/install.sh`.
- Do:
  - `inject-env.sh` (env `AR_DOTENV AR_DOTENV_PATH AR_DART_DEFINES_JSON`): `umask 077`; for each comma path, `printf '%s' "$AR_DOTENV" > path`; write the JSON to `$AR_STATE/dart-defines.json`. Never echo values; log only "wrote N env file(s)".
  - `fingerprint.sh`: prints the sha256 per plan §11 (uses `sha256sum` of the secrets, not the secrets themselves).
  - `build-flutter.sh` (env `APP_DIR BUILD_MODE FLAVOR TARGET DART_DEFINES PRE_BUILD`): `cd APP_DIR`, `flutter pub get`, run `PRE_BUILD` if set (repo-relative, must exist, run with `bash`), build the args array (`--flavor`, `-t`, one `--dart-define` per non-empty line, `--dart-define-from-file` if the json exists), `flutter build apk`, copy the result to `$AR_STATE/app.apk`. On failure with `BUILD_MODE=release`, add the hint "release needs signing config; try build_mode: debug".
  - `build-gradle.sh` (env `APP_DIR GRADLE_TASK APK_GLOB`): `./gradlew "$GRADLE_TASK"`, take the newest file matching the glob, copy it to `app.apk`. `die` if there is none.
  - `fetch-apk.sh` (env `APK GH_TOKEN`): parse `url:` (https only, `curl -fsSL --max-filesize 500M`), `release:<tag>/<asset>` (`gh release download`), `artifact:<run>/<name>` (`gh run download`, take the first `*.apk`). Output to `app.apk`.
  - `inspect-apk.sh`: use `aapt2 dump badging` from the newest build-tools to write `$AR_STATE/app.json` `{package, versionName, launchable, abis}`; warn per plan §8 when `lib/` exists without `x86_64`.
  - `install.sh` (env `LAUNCH`): `adb install -r -t -g app.apk` with retry 3; launch with `monkey -p <pkg> -c android.intent.category.LAUNCHER 1` when `LAUNCH=true`.
- Done when: shellcheck passes; verified in T014/T024.

### T014 Sub-action `setup-emulator` + self-test (emulator part)
- Files: `actions/setup-emulator/action.yml`, `.github/workflows/self-test.yml` (job `emulator` only for now).
- Do: composite inputs `api_level system_image device locale cores ram_mb boot_timeout cache phase` (`all|start|wait`). Steps: kvm → restore caches (sysimg + avd keys, plan §11) → install → start → (if `phase != start`) wait → settings → snapshot + cache save on a miss. Scripts are reached via `${{ github.action_path }}/../../runner/...`. Outputs `serial`, `emulator-version`, `snapshot-hit`. Self-test: matrix api `[30, 34, 35]`, `uses: ./actions/setup-emulator`, then `adb shell getprop ro.build.version.sdk` equals the matrix api.
- Done when: the self-test passes on all three APIs; a second run shows cache hits and a faster boot (note timings in the PR description).

---

## P2 — Interactive session

### T020 [P] prepare.mjs
- Files: `runner/prepare.mjs`.
- Do: read every input from `process.env.INPUT_*`-style env names set by action.yml (`AR_MODE`, `AR_CONFIG`, …), parse `AR_CONFIG` JSON (an allowlist of the keys in plan §4.1; unknown key → `::error::` + exit 1), merge (config > input > default). Validate: mode, minutes 1–300, idle 0–120, api 30–36, `system_image` in an allowlist, flavor/target/paths `^[A-Za-z0-9_./-]+$` (no `..`), `apk` spec format, `e2e` enum, `e2e_args` without `;|&$\`` characters. Policy: interactive on `GITHUB_EVENT_NAME` starting with `pull_request` → fail; interactive with an empty client key and a public repo (read `GITHUB_EVENT_PATH` → `.repository.private`) → fail with the plan §6.6 message. Write the normalized values to `$GITHUB_OUTPUT` and a non-secret table to the job summary.
- Done when: `AR_MODE=interactive AR_CONFIG='{"bogus":1}' node runner/prepare.mjs` exits 1 with a clear message, and a valid local env run prints the outputs (point `GITHUB_OUTPUT` at a temp file).

### T021 [P] secrets.mjs + redact.mjs
- Files: `runner/secrets.mjs`, `runner/session/redact.mjs`.
- Do: `secrets.mjs` per plan §6.2 (`--agent` flag → also an agentToken). It prints only the `::add-mask::` lines. `redact.mjs <in> <out>`: load `secrets.json` + `$AR_STATE/url`; replace every occurrence of those values and of the URL host, plus the regexes `([?&]k=)[^&\s]+`, `(Authorization:\s*Bearer\s+)\S+`, `(Cookie:\s*)[^\n]+`, with `***`.
- Done when: a local run produces `secrets.json` with mode 600, and `redact.mjs` on a sample file containing a fake key and URL outputs it with `***`.

### T022 Gate
- Files: `runner/session/gate.mjs`.
- Do: plan §9 "gate.mjs" and §7.4, exactly. Env: `GATE_PORT=3199`, `VIEWER_PORT`, `AGENT_PORT=4310`, `AR_STATE`, `DENY_FILE`. Load secrets from `$AR_STATE/secrets.json`. Routing: `/__ar/healthz` → 200 `ok`; `/agent/*` → Bearer check → strip the `/agent` prefix → agent upstream with `X-AR-Internal`; everything else → viewer auth (`k` query or cookie) → viewer upstream, Host preserved. WebSocket upgrades follow the same auth rules (cookie for the viewer). 503 "starting" page until the upstream port answers. In-memory token buckets and per-IP failure counters (`cf-connecting-ip` header, else the socket address). Write `$AR_STATE/stop` at 200 failures. Touch `last-activity`. Log one line per request: time, method, path (no query), status, ms. Graceful SIGTERM.
- Done when: running locally with a dummy upstream (`python3 -m http.server`) behaves like this: `curl /` → 403; `curl '/?k=<key>'` → 302 with Set-Cookie and no `k` in Location; with the cookie → 200 from the upstream; `/agent/v1/device` without a token → 401; the 11th bad key in a minute → 429; the log file contains no key.

### T023 [P] Viewer adapters
- Files: `runner/viewers/ws-scrcpy-web/start.sh`, `runner/viewers/ws-scrcpy-web/deny-paths.txt`, `runner/viewers/ws-scrcpy/start.sh`, `runner/viewers/ws-scrcpy/deny-paths.txt`.
- Do: implement the plan §9 adapter contract using T006's NOTES (image digest, device add, stream params, deny paths). Both scripts write `$AR_STATE/viewer-entry` and log to `$AR_STATE/viewer.log`, and `wait_http` until ready (120 s).
- Done when: shellcheck passes; verified in T027.

### T024 [P] Tunnel, publish, hold, teardown
- Files: `runner/session/tunnel.sh`, `runner/session/publish.mjs`, `runner/session/hold.sh`, `runner/session/teardown.sh`.
- Do: `tunnel.sh` per plan §9 (download pinned with sha256 check, cache dir `$AR_STATE/tools`, mask the URL+host, write `$AR_STATE/url`, `cloudflared.pid`). `publish.mjs` (env `AR_SESSION AR_CLIENT_PUBLIC_KEY AR_MINUTES GITHUB_RUN_ID`): build the plaintext, `seal()` from `shared/sealed-box.mjs` (relative import `../../shared/sealed-box.mjs`), write `$AR_STATE/publish/session.sealed.json`; with no client key in a private repo, append the link to `$GITHUB_STEP_SUMMARY` (plan §6.6); write `expiresAt` to `$AR_STATE/deadline`. `hold.sh` per plan §5.2 (print "live until HH:MM UTC" and remaining minutes every 5 min; never the URL). `teardown.sh`: kill cloudflared first, then the viewer (docker rm -f), gate, agent, and the emulator (`adb emu kill`, then the pid); redact every log into `$AR_STATE/logs-redacted/`.
- Done when: shellcheck and `node --check` pass; verified in T027.

### T025 [P] Agent IME helper
- Files: `runner/agent/ime.sh`.
- Do: download the pinned ADBKeyBoard APK (sha256 check), install it, `ime enable com.android.adbkeyboard/.AdbIME`. Do **not** set it as default at boot. `agent-api.mjs` switches IME only around non-ASCII typing (T031). If S3 failed, the script exits 0 with a warning and writes `$AR_STATE/unicode-unsupported`.
- Done when: shellcheck passes.

### T026 Root composite action
- Files: `action.yml`.
- Do: inputs and outputs exactly as in plan §4.1. Steps, named with `[ar:<stage>]` prefixes in the order of plan §5.2: prepare (setup-node 20 SHA-pinned, `node prepare.mjs`, id `cfg`) → secrets → kvm/caches/install/start (background) → setup-java 17 + `subosito/flutter-action` (flutter only) + `gradle/actions/setup-gradle` (gradle/flutter) → APK cache restore → inject-env → build-flutter | build-gradle | fetch-apk (by `app_source`, skipped on a cache hit) → APK cache save → wait/settings/snapshot → inspect + install → interactive branch: [agent: ime.sh + `nohup node agent-api.mjs`] → `nohup node gate.mjs` → tunnel.sh → viewer start → publish.mjs → `actions/upload-artifact` (name from `cfg` outputs, path `$AR_STATE/publish`, retention 1, only if the client key is set) → hold.sh; e2e branch: `uses` the same steps as `actions/e2e` inline (T044 refactors to share) → teardown (`if: always()`) → upload logs artifact (`if: always()`, retention 3). Every `run:` uses `env:` only.
- Done when: `actionlint` and `check-forbidden` pass.

### T027 Reusable workflow + interactive self-test
- Files: `.github/workflows/android-remote.yml`, `.github/workflows/self-test.yml` (add job `interactive`).
- Do: reusable workflow per plan §4.2 (`uses: melbanna40/android-remote@develop`). Self-test job: `uses: ./` with `mode: interactive, minutes: 5, agent: true, app_source: flutter, app_dir: examples/flutter-sample, flavor: development, target: lib/main_development.dart, client_public_key: <a fixed test public key stored as the repo variable AR_SELFTEST_PUBKEY>`. Add a parallel step before hold, in the same job: `node scripts/selftest-probe.mjs`, which reads `$AR_STATE/url` and secrets **locally** (never prints them) and checks: `/` → 403; `?k=` → 302; the cookie → 200; `/agent/v1/device` with the token → 200 JSON. Fail the job if any check fails. Create `scripts/selftest-probe.mjs`.
- Done when: the self-test passes on develop, and `scripts/log-leak-scan.sh` (T062) over its logs finds nothing.

---

## P3 — E2E mode

### T040 [P] Recorder
- Files: `runner/e2e/record.sh`.
- Do: `record.sh start|stop|discard`. `start`: background loop `screenrecord --time-limit 180 --bit-rate 4000000 /sdcard/ar-rec-$(printf %03d $n).mp4`, pid in `$AR_STATE/record.pid`. `stop`: `adb shell pkill -INT screenrecord`, kill the loop, sleep 2, pull `/sdcard/ar-rec-*.mp4` → `$AR_OUT/recording/`. `discard`: stop and delete. Fallback from S4 if needed (scrcpy v5 `--record --no-window --no-audio`).
- Done when: shellcheck passes.

### T041 [P] Maestro runner
- Files: `runner/e2e/maestro.sh`.
- Do: install the pinned Maestro release zip (sha256) into `$AR_STATE/tools/maestro`, unless cached. Run `maestro test "$E2E_PATH" --device "$ADB_SERIAL" --format junit --output "$AR_OUT/junit/maestro.xml" --test-output-dir "$AR_OUT/screenshots" --debug-output "$AR_OUT/maestro" $E2E_ARGS` (pass `-e APP_ID=<package>` from `app.json`). Store the exit code in `$AR_STATE/e2e-exit`; do not exit non-zero yet.
- Done when: shellcheck passes; verified in T045.

### T042 [P] Flutter integration_test runner + custom
- Files: `runner/e2e/flutter-it.sh`, `runner/e2e/custom.sh`.
- Do: `flutter-it.sh`: `cd APP_DIR; flutter test "$E2E_PATH" -d "$ADB_SERIAL" [--flavor] [-t is not used] [--dart-define…] [--dart-define-from-file] --file-reporter "json:$AR_OUT/flutter.json" $E2E_ARGS`, then `dart pub global activate junitreport <pinned>` and `tojunit --input $AR_OUT/flutter.json --output $AR_OUT/junit/flutter.xml`. `custom.sh`: run the repo-relative script `E2E_COMMAND` with `AR_JUNIT_DIR=$AR_OUT/junit ADB_SERIAL APP_PACKAGE` exported. Both store their exit code in `$AR_STATE/e2e-exit`.
- Done when: shellcheck passes.

### T043 [P] Collect + summary
- Files: `runner/e2e/collect.sh`, `runner/e2e/summary.mjs`.
- Do: `collect.sh`: final `screencap` → `screenshots/final.png`; `logcat -d -v threadtime > logcat/full.txt`; app pids (`pidof <pkg>` captured at start/end) → grep → `logcat/app.txt`; `device.json`; apply the record policy (on-failure + pass → discard). `summary.mjs`: parse every `junit/*.xml` (regex over `<testcase …>` with `<failure`/`<error` children), write `summary.md` and append it to `$GITHUB_STEP_SUMMARY`; write outputs `tests`, `failures`, `result`, `junit-path` to `$GITHUB_OUTPUT`; exit 1 if failures > 0 or `e2e-exit != 0`.
- Done when: `node runner/e2e/summary.mjs` on a hand-written sample JUnit with 1 failure prints the right counts and exits 1.

### T044 Sub-action `e2e` and wiring
- Files: `actions/e2e/action.yml`, `action.yml` (e2e branch → call the same scripts; keep both in sync by having `action.yml` steps mirror `actions/e2e` step-for-step, with a comment pointing to it).
- Do: inputs `e2e e2e_path e2e_args e2e_command app_dir flavor dart_defines record artifact_retention_days serial`. Steps: record start → runner → record stop → collect → summary (id `sum`) → upload-artifact (`if: always()`) → outputs.
- Done when: actionlint passes.

### T045 E2E self-test
- Files: `.github/workflows/self-test.yml` (add jobs `e2e-maestro`, `e2e-maestro-failing`, `e2e-flutter`).
- Do: `uses: ./` on `examples/flutter-sample`. The failing job uses `e2e_args: --include-tags failing` with `continue-on-error: true`, followed by a step asserting `steps.<id>.outputs.result == 'failed'` and that the artifact contains `recording/` and `junit/`.
- Done when: all three jobs behave as described on develop.

---

## P4 — Agent API

### T030 adb wrapper
- Files: `runner/agent/adb.mjs`.
- Do: plan §7.3, exactly: `shq(s)`, the validators (`isPackage`, `isActivity`, `isUrl`, `clamp`), `run(args, {timeoutMs, input?, binary?})` built on `execFile` (maxBuffer 64 MB, binary-safe for screencap), a serial queue so only one input command runs at a time, and the helpers `tap, swipe, text, keyevent, screencapPng, uiDump, install(pathOrBuffer), launch, stop, clear, openLink, logcat, clearLogcat, rotate, deviceInfo`. Non-ASCII `text()` → base64 broadcast with an IME switch and restore (read `settings get secure default_input_method` first).
- Done when: `node --check` passes, and a local run against an emulator, if one is available (or T027's self-test probe), succeeds for `deviceInfo()`.

### T031 Agent API server
- Files: `runner/agent/agent-api.mjs`, `runner/agent/ui-xml.mjs`.
- Do: `http.createServer` on `127.0.0.1:${AGENT_PORT}`; require `x-ar-internal` (constant-time). Routes and response shapes per plan §7.2 (paths without the `/agent` prefix: `/v1/...`). Body parsing with size caps (64 KB JSON; `/v1/install` streams to a temp file, ≤ 100 MB). Map errors to the plan §7.1 codes. `ui-xml.mjs`: parse `<node …/>` attributes with a regex, flatten, compute centers from `bounds="[l,t][r,b]"`. Log to `$AR_STATE/agent.log` (method, path, status, ms).
- Done when: `node --check` passes; verified in T082.

---

## P5 — CLI

### T050 CLI skeleton
- Files: `cli/bin/android-remote.mjs`, `cli/src/args.mjs`, `cli/src/ui.mjs`, `cli/src/errors.mjs`.
- Do: `util.parseArgs`-based parsing with subcommands `init doctor up status down mcp` (default `up`), plus `--help` and `--version` (from package.json). `CliError(message, fix, exitCode)`; the top-level handler prints `✗ message` + `Fix: …` and exits with the code (`ANDROID_REMOTE_DEBUG=1` → stack). `ui.mjs`: step/ok/warn/info with color only on a TTY, a spinner on a TTY, and `--json` mode (no colors, one JSON object at the end). Commands are lazy-imported.
- Done when: `node cli/bin/android-remote.mjs --help` lists the commands, and an unknown flag exits 2.

### T051 [P] gh/git wrappers
- Files: `cli/src/gh.mjs`, `cli/src/git.mjs`.
- Do: `gh.mjs` (all through `execFileSync('gh', …)`, JSON parse): `requireAuth()`, `repoNameWithOwner()`, `isPrivate()`, `defaultBranch()`, `fileExistsOnRef(path, ref)`, `dispatch(workflow, ref, inputs) → {runUrl?}` (`gh workflow run <file> --ref <ref> --json` with inputs on stdin; parse a URL from stdout), `findRunBySession(session, mode)`, `getRun(id)`, `listArtifacts(runId)`, `downloadArtifact(runId, name, dir)`, `cancelRun(id)`, `forceCancelRun(id)`, `inFlightRuns(workflow)`. `git.mjs` (read-only commands only): `gitCommonDir()`, `currentBranch()`, `headSha()`, `isDirty()`, `unpushedCount(ref)`, `remoteSha(ref)`.
- Done when: `node scripts/check-forbidden.mjs` passes and `node -e` calls of `repoNameWithOwner()` work inside this repo.

### T052 [P] Session store + states
- Files: `cli/src/session-store.mjs`, `cli/src/states.mjs`.
- Do: plan §5.3 (`save, load(id?), latest({agent?}), list(), remove(id)`). Create dirs 0700 and files 0600 (`fs.writeFileSync(…, {mode: 0o600})` + `chmodSync`). `states.mjs`: state constants and a `nextState(current, observation)` pure function per plan §5.1.
- Done when: saving a session creates a 0600 file under `.git/android-remote/sessions/`.

### T053 [P] Template + init
- Files: `cli/templates/caller-workflow.yml`, `cli/src/template.mjs`, `cli/src/commands/init.mjs`.
- Do: the template per plan §4.3 with `{{APP_DIR}} {{APP_SOURCE}} {{FLAVOR}} {{TARGET}} {{PRE_BUILD}} {{DOTENV_PATH}} {{REF}}` placeholders. `init` flags: `--app-dir --app-source --flavor --target --pre-build --dotenv-path --pin --force`. Auto-detect: `pubspec.yaml` in the app dir → flutter; `gradlew` → gradle. `--pin` resolves the latest release tag's SHA through `gh api repos/melbanna40/android-remote/git/ref/tags/<tag>`. If the file exists without `--force` → CliError. It writes only that one file, then prints the next steps (`git add …`, `git commit -m "ci: add android-remote"`, `git push`, `gh secret set ANDROID_REMOTE_DOTENV < .env.remote` if a dotenv path was given) **as text for the user**.
- Done when: `init` in a temp Flutter project writes a file that passes `actionlint`, and a re-run without `--force` exits 1.

### T054 doctor
- Files: `cli/src/commands/doctor.mjs`.
- Do: checks per spec FR-7.2, each `{name, ok, fix}`: node version, gh on PATH, `gh auth status`, repo detected, `gh api repos/{r}/actions/permissions` → enabled, workflow file on the default branch (`fileExistsOnRef`), template version header ≥ `TEMPLATE_VERSION`, and with `--ref R` the file on R. Exit 3 if any fail. `--json` support.
- Done when: running in this repo (no caller file) reports the missing workflow with the `init` fix.

### T055 [P] CLI self-check
- Files: `cli/test/self-check.test.mjs`.
- Do: three `node:test` cases: sealed-box round trip plus a wrong-session failure; `args.mjs` parses `up --minutes 20 --dart-define A=1 --dart-define B=2` into the expected object; the template renders with sample values and contains no `{{`.
- Done when: `npm run selfcheck` passes.

### T056 up
- Files: `cli/src/commands/up.mjs`.
- Do: flags per spec FR-7.3. Flow per plan §5.1: preflight (doctor subset), ref = `--ref` or the current branch (must exist on the remote: `remoteSha`), dirty/unpushed warnings, session id `randomBytes(4).hex`, keypair (interactive only), map flags to the ≤ 9 dispatch inputs + `config` JSON (`--api`, `--device`, `--locale`, `--idle-minutes`, `--agent`, `--build-mode`, `--dart-define` (warn "visible in logs"), `--e2e*`, viewer tuning), dispatch, find the run, save the session, poll (spinner shows the state + the run URL), on artifact → download to a temp dir → `unseal` → remove the private key → save → print the link `url + '/?k=' + key`, the expiry, and (if agent) the MCP hint → `open` the browser unless `--no-open` (`open`/`xdg-open`/`start` via execFile). E2E: `--wait` polls to completion, downloads the e2e artifact to `./android-remote-results/<runId>/`, prints the summary counts, exit 0/4. `--json`: final object `{session, runId, runUrl, state, url?, key?, agentToken?, expiresAt?}` (keys included only in `--json` because agents need them, documented).
- Done when: dispatching against the external test repo (T080) reaches LIVE and prints a working link.

### T057 [P] status + down
- Files: `cli/src/commands/status.mjs`, `cli/src/commands/down.mjs`.
- Do: `status`: for each stored session (or `--session`), refresh the run state and print state, remaining minutes, the URL (key masked as `k=…xyz9` unless `--show-key`), and the run URL. Remove sessions whose run completed more than 1 h ago. `down`: the latest (or `--session`) → `cancelRun`, wait ≤ 60 s for completion, then `forceCancelRun`; remove the session file. `--all` → every in-flight run of the workflow.
- Done when: in T080, `down` ends the run in under 60 s and the link stops answering.

### T058 mcp bridge
- Files: `cli/src/commands/mcp.mjs`.
- Do: plan §7.5. Each tool = a zod-free JSON schema (SDK `McpServer.tool` with input schema objects) → `fetch(url + '/agent/v1/...', {headers: {authorization: 'Bearer ' + token}})`. `screenshot` returns `{type:'image', data: base64, mimeType:'image/png'}`. `install_apk` reads a local path (≤ 100 MB) and posts it raw. Errors become MCP tool errors with the API's message. stdout is reserved for the protocol: logs go to stderr only.
- Done when: `claude mcp add android-remote -- node <repo>/cli/bin/android-remote.mjs mcp` lists the tools in Claude Code (`/mcp`), and `session_status` answers "no live session" when there is none.

---

## P6 — Docs and release

### T060 [P] User docs
- Files: `README.md` (quick start, how it works diagram, commands, security summary, third-party software + licenses, credit to flutter-remote), `docs/getting-started.md`, `docs/cli.md`, `docs/workflow-inputs.md` (every input + config key, with defaults), `docs/e2e.md` (Maestro / Flutter / custom-for-Patrol examples, PR-check YAML), `docs/troubleshooting.md` (the spec §9 edge cases as symptom → fix).
- Done when: every input in `action.yml` appears in `workflow-inputs.md` (check with a grep loop), and no secret-looking example values appear.

### T061 [P] Security, protocol, agent, cost docs
- Files: `docs/security.md` (plan §14 table + guidance), `docs/protocol.md` (plan §6 byte-level, the artifact names, the S2 results), `docs/agent-api.md` (plan §7 with curl examples + MCP setup for Claude Code / Hermes / OpenCode), `docs/costs.md` (plan §12).
- Done when: reviewed by the lead.

### T062 [P] Release tooling + leak scan + agent smoke
- Files: `scripts/release-pin.mjs`, `.github/workflows/release.yml`, `scripts/log-leak-scan.sh`, `scripts/agent-smoke.sh`.
- Do: `release-pin.mjs X.Y.Z`: rewrite `melbanna40/android-remote@develop` → `@vX.Y.Z` in `.github/workflows/android-remote.yml`, set the package.json version, print the diff. `release.yml` (on push tag `v*.*.*`): fail if `@develop` remains; `npm publish --provenance --access public` (`NPM_TOKEN` secret, `id-token: write`); move the `v<major>` tag (`git tag -f v1 && git push -f origin v1`, inside the release workflow only); create the GitHub release from the CHANGELOG section. `log-leak-scan.sh <run-id> <needle-file>`: `gh run view --log` + download the logs artifact + the summary API → grep each needle (key, token, host), exit 1 on a hit. `agent-smoke.sh` (env `AR_URL AR_TOKEN`): curl in order: device, tap center, type `مرحبا Hello`, ui (assert `Hello` appears), screenshot (PNG magic bytes), open-link `arsample://echo?text=smoke`, ui (assert `smoke`), logcat lines=50, then no token → 401, then 11 bad tokens → 429.
- Done when: `release-pin.mjs 1.0.0` on a scratch branch produces the expected diff; `bash -n` passes on both scripts.

---

## P7 — Verification (lead or implementer with repo access; record evidence in `specs/001-android-remote/verification.md`)

### T080 External test repo
- Do: the lead creates the public repo `melbanna40/android-remote-sample` (outside the CLI) with a copy of `examples/flutter-sample`. In it, run `npx github:melbanna40/android-remote#develop init --app-dir . --flavor development --target lib/main_development.dart`; the user commits and pushes. `doctor` is all green.
- Done when: `verification.md` has the doctor output.

### T081 Interactive session in the browser
- Do: `npx … up --minutes 15 --agent`. Open the link in the Orca browser: the app is visible, tap increment → 2, type in the field, rotate. Open the bare URL in a fresh tab → 403. Run `status`, then `down` → the run is cancelled < 60 s and the URL is dead. Record screenshots and timings (dispatch → live, cold and warm).
- Done when: AC1 and AC2 are evidenced.

### T082 Agent API + MCP smoke
- Do: during a live `--agent` session: `AR_URL=… AR_TOKEN=… scripts/agent-smoke.sh` passes. Then in Claude Code with the MCP bridge: screenshot, ui_tree, tap by the `increment` node center, `open_link`.
- Done when: AC5 is evidenced (script output + an MCP transcript excerpt).

### T083 E2E as a PR check
- Do: in the sample repo, add `.github/workflows/e2e.yml` (plan §4.4, `e2e: maestro`), open a PR → green. Push a commit that runs the `failing` tag → red, with an artifact containing junit, recording, logcat and screenshots. Repeat once with `e2e: flutter`.
- Done when: AC3 and AC4 are evidenced (run URLs + artifact listing).

### T084 Leak audit and public-repo policy
- Do: run `scripts/log-leak-scan.sh` for the T081 and T083 runs with the session's key, token and host as needles → no hits. Dispatch interactive from the GitHub UI with no client key → it fails at `[ar:prepare]` with the documented message.
- Done when: AC6 and AC7 are evidenced.

### T085 Roquia dry run (private repo)
- Do: on a roquia-app feature branch, follow plan Appendix A.1 with a staging/mock `API_URL`, then `up --minutes 15 --config '{"locale":"ar-EG","flutter_version":"3.47.5"}'`. Check the RTL Arabic UI renders and log in with a `*@roquia.test` account. Record the billed minutes from the run's timing.
- Done when: `verification.md` has screenshots and minute usage. Do not merge the Roquia branch without the owner.

---

## Dependency summary

```text
T001 → (T002,T003,T004,T005,T006,T007,T008) [P]
T010 → (T011,T012,T013) [P] → T014
T014 + T006 + T005 → (T020,T021,T023,T024,T025) [P], T022
T021 → T030 → T031
(T020..T025, T031) → T026 → T027   (T026 starts agent-api.mjs, so P4 lands before it)
T026 → (T040,T041,T042,T043) [P] → T044 → T045
T002 + T003 → T050 → (T051,T052,T053,T055) [P] → T054, T056 → (T057, T058) [P]
T027 + T045 + T056 → (T060,T061,T062) [P] → T080 → T081 → T082, T083 → T084 → T085
```
