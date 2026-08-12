---
fileClass: Project
Category: Claude
Status: Draft
Authors: Claude (draft) · John-Michael (owner) · Mason Bendixen
Last Updated: 8/12/2026
Version: 0.1
tags:
---

# Bucket PL1 — CI + branch protection

**Track:** PL — Platform & ops · **Size:** S (~a day from the template) · **Stream:** Phase 0 (pulled forward as the merge arbiter) · **Owner:** John-Michael
**Needs:** nothing (the repo + both build systems already exist). **Feeds:** every stream — Interface Contract 6 ("CI is the arbiter") is this bucket; branch protection is what lets 3–4 committers fan out safely.

# Context

Implements [[Initial Project Implementation Outline]] Phase 0.4 / bucket PL1 and supersedes the CI half of [[Setting up the project]] Phase 15. Two GitHub Actions jobs — a **server job cloned from `server_components/.github/workflows/ci.yml`** (gcc container + postgres service + Conan cache + the test-count floor) and a **ui job** (npm ci + vitest + both builds) — plus **branch protection on `master`** requiring both checks. After this bucket: all work lands through PRs, each stream owner still runs their local docker gate per slice, and destructive DB ops (`--recreate_database`) stay human-run.

**Source templates (read these, then adapt):**

- `C:\Users\mason\source\repos\server_components\.github\workflows\ci.yml` — the server-job template (245 lines, heavily commented; the comments explain *why* each step is shaped the way it is — keep them in spirit).
- `C:\Users\mason\source\repos\server_components\docker\build_and_test.sh` and CF's own `server/docker/build_and_test.sh` + `build_common.sh` — the local twins. **The conan flags in CI must stay identical to CF's docker scripts** (same `build_type`, same `compiler.cppstd`, same `--build=missing`); if they drift, CI and the local gate build different binaries and disagreements become undebuggable. CF's docker scripts are the source of truth at implementation time.
- `C:\Users\mason\source\repos\honuware-web-components\.github\workflows\ci.yml` — the `verify` job there is the ui-job model (npm ci → lint → test → build); CF's version drops lint (not configured yet) and pack/publish (nothing is published).

# Scope

**In:** `.github/workflows/ci.yml` with the two jobs · test-count floor in CI **and** aligning the docker script's floor with it · branch protection rule on `master` requiring both checks · the team PR-flow convention written down (README section) · CI badge.

**Out (explicit non-goals):** deployment/release packaging, Docker images, AWS (**PL2**) · rate limiting, analytics, runbooks (**PL3**) · Windows CI (Mason's VS spot-checks stay manual, per the division of labor) · coverage/lint gates (add later if wanted — don't block Phase 0 on new tooling) · honuware's own CI (already exists in its repo).

# Design pin-downs

- **Job names are protected identities.** Branch protection matches on the job `name:` string — pin them now (`Server build + tests (Linux)`, `UI build + tests`) and never rename casually (the honuware template documents this trap; toolchain versions live in `container:`, not the name).
- **The test-count floor is load-bearing, in both places.** `communityfinder_test_cases` and `honuware_tests` are static libraries of self-registering `TEST()`s that `test/src/main.cpp` never references by symbol — a linker is entitled to silently drop them, and the volatile-anchor convention means a dropped endpoint TU 404s every route in Release while the exit code stays 0. The floor is the alarm. Policy: **floor = (measured actual − ~10), kept identical in CI and `server/docker/build_and_test.sh`, raised at every bucket's exit** (EV1 §7.2, GD2 §8.2 already say so). Suite was 1517 at Phase 9 — measure a fresh local run and set both floors accordingly (≈1500). If the floor ever trips on a real link regression, the fix is `$<LINK_LIBRARY:WHOLE_ARCHIVE,…>` — never lowering the floor.
- **Postgres service: leave `POSTGRES_DB` unset.** The image then names the default database after `POSTGRES_USER` (`docker`), which the framework's no-database bootstrap connection requires. Setting it "helpfully" breaks bootstrap.
- **FetchContent needs network + git in the container** — the configure step clones `github.com/honuware/server_components` at the pinned `GIT_TAG` (public repo, no token). The apt step already installs git.
- **Conan cold-start is real:** ConanCenter has no prebuilt gcc-14 binaries, so the first run compiles all ~16 packages (≈60–90 min, inside the 120-min timeout). The `actions/cache` step is what makes every later run 10–20 min. Cache key hashes the **app's** `conanfile.py`.
- **Don't "fix" the inert `if(!WIN32)` block** in `server/communityfinder_server/CMakeLists.txt` while you're in there — correcting it to `if(NOT WIN32)` activates a `CMAKE_C_COMPILER=/usr/bin/gcc` line that breaks the gcc:14.2.0 container (gcc lives under `/usr/local`). Known, deliberate, documented in SUTP Phase 2.
- **`.gitattributes` already pins `*.yml` to LF** (Phase 1.1) — no CRLF shebang risk from Windows checkouts.
- **vitest in CI:** GitHub sets `CI=true`, so `ng test` runs once and exits. If it ever hangs in watch mode, pass `--watch=false` explicitly.

# The workflow (drop-in, then verify against the templates)

`.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches: [ master ]
  pull_request:
  workflow_dispatch:

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  server:
    # Branch protection matches on this name — keep it stable.
    name: Server build + tests (Linux)
    runs-on: ubuntu-latest
    timeout-minutes: 120
    defaults:
      run:
        shell: bash            # container jobs otherwise fall back to dash (no pipefail)
        working-directory: server/communityfinder_server
    container:
      image: gcc:14.2.0
    services:
      postgres:
        image: postgres:13.1   # matches dev container + production
        env:
          POSTGRES_USER: docker
          POSTGRES_PASSWORD: docker
          # POSTGRES_DB intentionally unset -> default DB named `docker` (bootstrap needs it)
        options: >-
          --health-cmd "pg_isready -U docker"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 12
    env:
      HONUWARE_DB_HOST: postgres
      HONUWARE_DB_PORT: "5432"
      HONUWARE_DB_USER: docker
      HONUWARE_DB_PASSWORD: docker
      HONUWARE_DB_SSLMODE: disable   # Release/NDEBUG defaults to "prefer" otherwise
      CONAN_HOME: /root/.conan2
      # Floor policy: (measured actual - ~10), identical to server/docker/build_and_test.sh,
      # raised at every bucket exit. Set from a fresh local run before merging (~1500 today).
      MIN_EXPECTED_TESTS: "1500"
    steps:
      - uses: actions/checkout@v4

      - name: Install build tools
        # libkrb5-dev is not optional: static libpq auto-detects GSSAPI
        run: |
          set -euo pipefail
          apt-get update -qq
          apt-get install -y -qq --no-install-recommends cmake git python3-pip libkrb5-dev
          cmake --version

      - name: Install Conan 2.x
        run: |
          set -euo pipefail
          pip3 install --break-system-packages --quiet 'conan>=2.0,<3.0'
          conan profile detect --force
          conan --version

      - name: Cache Conan packages
        uses: actions/cache@v4
        with:
          path: /root/.conan2
          key: conan-gcc14.2.0-${{ hashFiles('server/communityfinder_server/conanfile.py') }}
          restore-keys: |
            conan-gcc14.2.0-

      - name: Conan install
        # Flags MUST match server/docker/build_common.sh (the local gate) exactly.
        run: |
          set -euo pipefail
          conan install . \
              --output-folder=build \
              --build=missing \
              -s build_type=Release \
              -s compiler.cppstd=17

      - name: CMake configure
        # Conan 2.28+ nests generators under conan/; probe the same candidates the template does.
        run: |
          set -euo pipefail
          TOOLCHAIN=""
          for candidate in \
              build/conan_toolchain.cmake \
              build/conan/conan_toolchain.cmake \
              build/build/conan_toolchain.cmake; do
              if [ -f "$candidate" ]; then
                  TOOLCHAIN="$(pwd)/$candidate"
                  break
              fi
          done
          if [ -z "$TOOLCHAIN" ]; then
              echo "ERROR: conan_toolchain.cmake not found under build/" >&2
              find build -name 'conan_toolchain.cmake' >&2 || true
              exit 1
          fi
          echo "Using toolchain: $TOOLCHAIN"
          cmake -S . -B build \
              -DCMAKE_BUILD_TYPE=Release \
              -DCMAKE_TOOLCHAIN_FILE="$TOOLCHAIN" \
              -DCMAKE_POLICY_DEFAULT_CMP0091=NEW

      - name: Build
        run: |
          set -euo pipefail
          cmake --build build -j "$(nproc)"

      - name: Run tests
        # THE COUNT ASSERTION IS LOAD-BEARING (dead-strip alarm) — never remove or lower casually.
        run: |
          set -euo pipefail
          cd build
          ./communityfinder_tests 2>&1 | tee test-output.txt

          count=$(sed -n 's/^\[==========\] Running \([0-9][0-9]*\) test.*/\1/p' \
                  test-output.txt | head -n 1)

          if [ -z "$count" ]; then
              echo "ERROR: could not parse a test count from the runner output." >&2
              exit 1
          fi

          echo "Tests executed: $count (floor: $MIN_EXPECTED_TESTS)"

          if [ "$count" -lt "$MIN_EXPECTED_TESTS" ]; then
              echo "ERROR: only $count tests ran, expected >= $MIN_EXPECTED_TESTS." >&2
              echo "       The suite did not fully link — anchors or a test bag were" >&2
              echo "       likely dead-stripped. Fix the link (WHOLE_ARCHIVE), never" >&2
              echo "       lower the floor. See CLAUDE.md." >&2
              exit 1
          fi

      - name: Upload test output
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: server-test-output
          path: server/communityfinder_server/build/test-output.txt
          if-no-files-found: ignore

  ui:
    # Branch protection matches on this name — keep it stable.
    name: UI build + tests
    runs-on: ubuntu-latest
    timeout-minutes: 30
    defaults:
      run:
        working-directory: ui
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
          cache-dependency-path: ui/package-lock.json

      - name: Install dependencies
        run: npm ci

      - name: Unit tests (vitest)
        run: npm test

      - name: Build (local / mock config)
        run: npx ng build --configuration local

      - name: Build (development config)
        run: npx ng build --configuration development
```

# Work items

### 1. Preflight
- [ ] 1.1 **[Mason]** Confirm John-Michael has repo access with enough rights for `.github/` pushes (collaborator), and that an **admin** (Mason, or JM if made admin) is lined up for the Settings → Branches step (5.x).
- [ ] 1.2 Measure the current suite count: one fresh run of `server/docker/build_and_test.sh` — note the `[==========] Running N tests` number (≈1517 expected). One `cd ui && npm test` run — note it passes locally on the Node version CI will use (22).

### 2. Server job
- [ ] 2.1 Create `.github/workflows/ci.yml` from the block above; diff every conan/cmake flag against `server/docker/build_common.sh` + `build_and_test.sh` and reconcile toward the docker scripts (they are the local gate; CI must be their twin).
- [ ] 2.2 Set `MIN_EXPECTED_TESTS` to (1.2's measured N − ~10).

### 3. UI job
- [ ] 3.1 The `ui` job as above. Verify `npm test` runs-once under `CI=true` (add `--watch=false` if not) and that both `ng build` configurations pass on a clean `npm ci` (no local node_modules assumptions).

### 4. Floor alignment + convention
- [ ] 4.1 Raise `server/docker/build_and_test.sh`'s floor (currently 1000) to the same value as CI's, so the local gate and CI agree. One-line change; note it in the PR description for the other streams.
- [ ] 4.2 Add the standing rule to `CLAUDE.md` (one line under testing conventions): *"Raise MIN_EXPECTED_TESTS in BOTH build_and_test.sh and ci.yml to (actual − ~10) at every bucket exit."*

### 5. Verification + protection
- [ ] 5.1 Open the PR containing 2–4. `on: pull_request` runs both jobs on the PR itself — first server run is the slow cold-cache one (~60–90 min); confirm both green, then re-run (or push a trivial commit) to confirm the warm-cache time (~10–20 min).
- [ ] 5.2 *(Optional but recommended)* Push a commit that deliberately breaks one C++ test and one vitest spec → confirm both jobs go red → revert. The arbiter has now been seen working.
- [ ] 5.3 Merge. Both checks now exist on `master` (they must have run at least once to appear in the protection picker).
- [ ] 5.4 **[Admin]** Settings → Branches → Add rule for `master`: ✅ Require a pull request before merging (approvals: **0** — OQ1) · ✅ Require status checks to pass, ✅ Require branches to be up to date, checks = `Server build + tests (Linux)` + `UI build + tests` · ✅ Do not allow bypassing the above settings (OQ2 — without this, every collaborator is an admin and the rule is advisory) · force pushes + deletions stay blocked (defaults).
- [ ] 5.5 Verify with a scratch PR: checks show as **Required**, merge is blocked until green, direct push to `master` is rejected.

### 6. Documentation
- [ ] 6.1 README: CI badge + a short "Contributing / how work lands" section — feature branches → PR → both checks green → merge (self-merge fine at 0 approvals); each stream owner still runs the local docker gate per slice; `--recreate_database` and other destructive DB ops stay human-run; the floor-raising rule.

# Gates

**Done =** a PR cannot merge with either job red; direct pushes to `master` bounce; `master` shows the green badge; docker + CI floors match; the convention lines exist in CLAUDE.md/README. (No docker-suite/ng-build gate of its own beyond the jobs themselves — this bucket ships the gate.)

# Open Questions

*(Numbered; recommendations included so "agreed" suffices.)*

1. **Required approvals: 0 or 1?** Three-to-four trusted committers, CI as the arbiter, speed matters for the 8/18 push. *Rec: 0 — checks required, review by request; raise to 1 later if churn ever warrants it.*
2. **"Do not allow bypassing" ON?** Everyone on the repo is currently an admin, so without it the protection is advisory. Emergency path with it on = an admin visibly toggles the rule off, acts, re-enables. *Rec: ON — that visible act is the right friction for a merge arbiter.*
3. **Cache `build/_deps` (the honuware FetchContent clone)?** Saves a shallow-ish clone per run; adds cache-invalidation surface keyed on the pinned SHA. *Rec: skip now; revisit only if the clone step ever gets slow or flaky.*
4. **Add `ng build --configuration production` to the ui job?** Nothing consumes a production bundle before deployment. *Rec: defer to PL2, where the real deploy artifact gets built.*
5. **Keep `concurrency: cancel-in-progress`?** Cancels superseded runs on force-push/rapid pushes — cheaper queues, but a canceled cold-cache Conan build wastes its progress (the cache only saves on completed runs of the cache step — note `actions/cache` saves post-job regardless of later step failures, but not on cancellation). *Rec: keep it (matches the template); the cold build happens once.*
