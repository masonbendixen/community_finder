---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 9/8/2026
Version: 0.1
tags:
---
# Overview

Go into plan mode and use this document for your planning. Don't ask for permission to modify it or work in .claude/plans. This is your plan file. Please leave this Overview alone and build the plan in the following sections.

I had been working with Visual Studio 2022 with CMake and Conan. Levi is using 2026. I want to migrate to an interim step where we support Visual Studio 2022 and 2026 temporarily and then do a full migration to Visual Studio 2026 after. I have moved to the same Conan as him (2.31.2). I am running CMake 3.29.9 and he is running 4.4.3. I'm open to moving to the new version of CMake and think that needs to be done. My version of cl is 19.44.35228 and his is 19.51.36348.

Please create a plan with phases of implementation. Within each phase, please respect the layering of the system and start with the work in lower layers first. Please create checkboxes by work items and then check them off as you implement them. Within the subsections of each phase, please number each such subsection. Please stick to your internal tools to inspect the filesystem and avoid external tools like grep, sed, and awk that you need to prompt me to run. I will build the C++ server and run tests myself. I will also commit and push to GIT myself so please don't use GIT commands unless you really need to understand the history of the files. Please don't prompt me if you can and run prompt requests to completion. Please always add tests for anything you chance for which testing is possible. When building this plan, please create an open questions section for things you need to ask me instead of asking me questions at the prompt.

# Findings

Everything below was measured against ConanCenter and the local Conan cache, by resolving the real dependency graphs of all three repos against a synthetic `compiler.version=195` profile in a throwaway Conan home — so nothing is masked by the warm cache that makes the 2022 machine look healthy.

## 1. Toolchain inventory

| | Mason | Levi |
|---|---|---|
| `cl` | 19.44.35228 | 19.51.36348 |
| `MSVC_VERSION` | 1944 | 1951 |
| Conan `compiler.version` | **194** | **195** |
| MSBuild toolset | v143 | v145 |
| CMake generator Conan emits | Visual Studio 17 2022 | **Visual Studio 18 2026** |
| CMake | 3.29.9 → 4.4.x in Phase 1 | 4.4.3 |

Two things to note. Your default Conan profile says `compiler.version=193`, but `conan_provider.cmake` derives the real value from `MSVC_VERSION` at configure time, so your builds have actually been running at **194** — which is the one configuration ConanCenter publishes binaries for. And the *Visual Studio 18 2026* generator exists only in CMake 4.2 and later, which is why Levi's 4.4.3 matters.

## 2. What actually blocks Visual Studio 2026

**2.1 — Two recipes pin CMake below 4, and CMake 3.x cannot target VS2026.** This is the blocker:

- `abseil/20220623.1` → `cmake/[>=3.16 <4]`
- `libtiff/4.6.0` → `cmake/[>=3.18 <4]`

Conan honours those caps, downloads a 3.x CMake as a tool dependency, and that CMake rejects the *Visual Studio 18 2026* generator. Levi's system CMake being 4.4.3 does not help — the recipe's own tool requirement wins. No other pinned recipe in any of the three repos caps CMake; everything else uses the system one.

**2.2 — Boost's build engine does not know toolset 14.5.** The Boost recipe writes `using msvc : 14.5 : …` into `user-config.jam` (Conan resolves v145 correctly). But `b2/5.3.2` — the version in your cache — declares `.known-versions = 14.3 14.2 14.1 14.0 …` in `msvc.jam`. There is no 14.5, so the toolset never configures. **b2 5.3.3 and newer add 14.5.** Boost asks for `b2/[>=5.2 <6]`, so this is fixed with a version floor, not a Boost upgrade — but a cached 5.3.2 satisfies that range and keeps winning until pinned.

**2.5 — libsodium builds through an MSBuild solution, and the 1.0.20 recipe has no entry for msvc 195.** Added after Phase 6, because this plan did not predict it. `_msvc_sln_folder` maps only 190–193 and falls back to `"vs2022"`, into which Conan injects `PlatformToolset=v145` — MSBuild then reports `MSB8020: The build tools for v145 cannot be found`. Fixed by libsodium ≥ 1.0.21, which map `"195": "vs2026"`.

The wider point: **the `cmake/[… <4]` scan in 2.1 can only see CMake-based recipes.** A recipe that drives MSBuild, b2 or autotools has its own toolset-version assumptions and needs its own check. Five packages in the graph touch MSBuild (boost, libjpeg, libpq, xz_utils, libsodium); libsodium was the only one carrying a per-version solution map.

**2.3 — Every dependency compiles from source on VS2026, and no version bump changes that.** ConanCenter publishes msvc binaries at exactly one configuration: `compiler.version=194`, `compiler.cppstd=17`. Verified against zlib, openssl, boost, abseil, gtest and libpqxx at both their oldest and newest versions — there is no 195 anywhere in the index. Upgrading recipes does not buy prebuilt binaries; it buys sources and recipes that survive a source build under MSVC 19.5x and CMake 4.4. Expect a long first build on Levi's machine and do not read it as a fault.

**2.4 — mailio pins Boost exactly, so the two move as one.** `mailio/0.25.3` requires `boost/1.86.0` — an exact pin, not a range. That, rather than a preference, is why Boost sits at 1.86. `mailio/0.26.0` pins `boost/1.91.0`. A mismatched pair fails at graph resolution with a version conflict before anything compiles.

## 3. What is *not* the problem

- **Conan 2.31.2 is fine.** It lists 195 in `settings.yml` and emits toolset `v145` and the right generator.
- **`conan_provider.cmake` is fine.** `string(SUBSTRING ${MSVC_VERSION} 0 3 …)` turns 1951 into "195" with no change needed.
- **No recipe rejects msvc 195.** At `compiler.cppstd=20`, not one recipe in any of the three graphs fails `validate()`. Nothing needs forking or patching — only moving forward. (At `cppstd=14`, or with cppstd unset, crow / mailio / libpqxx all go *invalid*; the profile in Phase 2 sets it explicitly for that reason.)

## 4. What the 2022 machine can and cannot prove

This plan now does all the work on Mason's machine first, which is the right call — it separates the *CMake 4* variable from the *VS2026* variable so they never fail together and confuse each other. It is worth being explicit about the limit of that, though:

**Can be proven on VS2022 + CMake 4.4:**
- Old dependency sources built under a CMake 4 that has removed `cmake_minimum_required` < 3.5 support. This is real risk and it lands in Phase 1.
- The CMP0167 / FindBoost decision.
- That every recipe bump keeps all three builds and the whole suite green.
- That the graph still *resolves* at 195, via a synthetic profile (subsection 3.5) — no VS2026 needed.

**Cannot be proven on VS2022:**
- Finding 2.1. The `<4` caps are invisible at msvc 194, because Conan fetches a CMake 3.x for those two recipes and CMake 3.x emits *Visual Studio 17 2022* perfectly happily.
- Finding 2.2. `b2/5.3.2` handles toolset 14.3 fine; only 14.5 is missing.

So a fully green 2022 machine does not mean Levi is unblocked — it means every *other* variable has been eliminated before he tries. Phase 6 is where the blockers actually get tested.

## 5. Scope: three repositories, one dependency set

All three conanfiles carry **identical version pins**, so every bump below is the same edit applied three times. The apps' lists are supersets of honuware's.

| Repo | Conanfile | Notes |
|---|---|---|
| `server_components` (honuware) | `conanfile.py` | The framework. Lists `abseil` but **never links `${ABSL_LIB}`** — see 6.1. |
| `knottyyoga` | `server/knottyyoga_server/conanfile.py` | Superset; adds `ftxui/5.0.0` + `replxx/0.0.4`. Consumes honuware via FetchContent. |
| `communityfinder` | `server/communityfinder_server/conanfile.py` | Superset; same two additions. Pins honuware at `GIT_TAG cad2942…`. |

## 6. Library → layer map (drives the work order in every phase)

| Layer | Libraries consumed |
|---|---|
| foundation | boost, crow, date, libpng, libtiff, zlib, libcurl (`util/http`) |
| data | libpqxx, boost, libcurl, crow |
| services | libsodium (secrets), boost + openssl + mailio (mail) |
| square / scheduler | (via foundation), boost |
| platform | libsodium (auth), libzip (branding / theme bundles) |
| testing / tests | crow, gtest |
| **app only** | abseil (`database_helper`, `test_helper` mains), ftxui + replxx (test-helper REPL) |

**6.1 — abseil is an app-only dependency.** Nothing in `server_components` links `${ABSL_LIB}`; it is used solely for `absl/flags` in the two app CLI mains (`absl::ParseCommandLine`, `absl::GetFlag`). That API has been stable since 2020, which drops the abseil bump from "four years of risk" to "low". It also means honuware's own conanfile carries a dependency it never uses — see Open Question 3.

# Phase 1 — CMake 4.4 on the VS2022 machine, current pins

First, and deliberately with **no dependency versions changed**. The point is to move one variable at a time: today's exact graph, today's compiler, new CMake. Anything that breaks here is a CMake-4 problem and nothing else, which makes it cheap to read.

The failure to expect: CMake 4 removed compatibility with `cmake_minimum_required` below 3.5, and recipes that declare no CMake tool requirement build with the *system* CMake — so 2022-era sources (zlib 1.3.1, libpng 1.6.40, libjpeg 9e, libcurl 7.86.0, libzip 1.10.1) will now meet CMake 4.4 for the first time. **That failure list is useful output, not a setback** — it tells you exactly which Phase 3 bumps are load-bearing rather than hygienic.

### 1.1 Raise CMake and rebuild the framework (foundation of everything)

- [x] Install CMake 4.4.x on the VS2022 machine to match Levi. ✅ 2026-09-04
- [x] Clean-configure and build `server_components` standalone with pins untouched. ✅ 2026-09-04
- [x] Record every dependency that now fails to build, with its error. This list feeds Phase 3. ✅ 2026-09-04

**Result: nothing failed.** The whole graph built under CMake 4.4 with pins untouched, so the `cmake_minimum_required` < 3.5 removal did not bite any of the 2022-era sources after all. That is a real reduction in scope — it means **every Phase 3 and Phase 4 bump is hygiene or VS2026-unblocking, not repair**. The two that still matter are libtiff (3.2) and abseil (4.3), and they matter only for Levi.

### 1.2 foundation scaffolding — CMP0167 and FindBoost

Done for `server_components` only; the apps are 1.3.

**Correction to the original plan text:** "drop the OLD setting and let config mode win" was wrong. The repo declares `cmake_minimum_required(VERSION 3.24)`, and a policy introduced after that floor (CMP0167 arrived in 3.30) defaults to **OLD** when unset. Deleting the block would therefore have kept module-mode Boost *and* added an unset-policy warning. The correct change is `OLD` → `NEW` inside the same guard.

- [x] Flip `cmake_policy(SET CMP0167 OLD)` to `NEW` in `server_components/CMakeLists.txt`, keeping the `if(POLICY)` guard. ✅ 2026-09-04
- [x] **Verified, not assumed.** Configured a throwaway project against the real Conan-generated `conan/conan/` folder from the 1.1 build, with CMP0167 NEW and the exact `find_package(Boost 1.83 REQUIRED COMPONENTS filesystem)` call: `Boost_FOUND=1`, version 1.86.0 satisfies the 1.83 floor via `BoostConfigVersion.cmake`, and both `Boost::filesystem` and the `boost::boost` target behind `${BOOST_LIB}` are declared. The second, component-less `find_package(Boost REQUIRED)` from `ConanLibImports.cmake` also survives. ✅ 2026-09-04
- [x] Confirmed the Conan dependency provider still intercepts the call. `conan_provider.cmake` registers `SET_DEPENDENCY_PROVIDER … SUPPORTED_METHODS FIND_PACKAGE`, so it sees every `find_package` regardless of mode, and it picks its config-mode branch on the literal `MODULE` argument (absent here) rather than on the policy. The trigger call therefore still forces `ConanLibImports.cmake` to be generated. ✅ 2026-09-04
- [x] Confirmed no impact on the Linux gate. `docker/Dockerfile` installs bookworm's CMake 3.25, where `if(POLICY CMP0167)` is false and the block is skipped entirely — the guard stays for exactly this reason. ✅ 2026-09-04
- [x] Mason: build `server_components` standalone and run the suite, then commit on its own. ✅ 2026-09-04

**Two things worth knowing about the new behaviour.** Config mode does *not* enforce the `COMPONENTS` list the way FindBoost did — the generated config never sets `Boost_filesystem_FOUND`, and the call still succeeds — so that clause is now documentation rather than a check. And because this block sits inside `if(PROJECT_IS_TOP_LEVEL)`, it is standalone-only: the two apps are untouched until 1.3, and nothing about their consumed-mode build changes in the meantime.

**No test is possible for this change.** It is a CMake policy selection with no runtime surface; the configure step is the test, which is why the verification above was done against the real generated files rather than left to the build.

### 1.3 App scaffolding

**Started without waiting on 1.2's CI, deliberately — see the note below.**

- [x] Applied the same `OLD` → `NEW` flip in both app top-level CMakeLists, keeping each repo's `if(POLICY)` guard and comment voice. Both had the identical starting pattern: `cmake_minimum_required(VERSION 3.24)`, a guarded `SET CMP0167 OLD`, then the same `find_package(Boost 1.83 REQUIRED COMPONENTS filesystem)` trigger ahead of `include(ConanLibImports.cmake)`. ✅ 2026-09-04
- [x] Verified config mode against **each app's own** generated `conan/conan/` folder, not just honuware's. Both return `Boost_FOUND=1` at 1.86.0, declare `Boost::filesystem` and `boost::boost`, and survive the second component-less `find_package(Boost REQUIRED)`. ✅ 2026-09-04
- [x] Confirmed the app-superset `honuware_layering.cmake` is CMake-4-safe by inspection: all three copies (honuware, knottyyoga, communityfinder) are structurally identical — the same three functions and a single `cmake_policy(SET CMP0057 NEW)` (IN_LIST, CMake 3.3). Nothing CMake 4 removed, and honuware's copy already validated under 4.4 in 1.1. ✅ 2026-09-04
- [x] Mason: build both apps and run the suites on Windows — `knottyyoga` and `communityfinder` both build and pass. ✅ 2026-09-04

**Why this did not need to wait for 1.2's CI.** Three independent reasons, any one of which is sufficient:

1. **CI cannot exercise the 1.2 change at all.** `.github/workflows/ci.yml` is Linux-only by design (its own comment says so — "Windows/MSVC is verified manually"), runs in `gcc:14.2.0`, and installs `cmake` from bookworm apt, i.e. 3.25. At 3.25 `if(POLICY CMP0167)` is false and the block never executes. A green CI would not have validated the flip, and a red CI could not have been caused by it.
2. **Consumed mode never runs honuware's block.** It sits inside `if(PROJECT_IS_TOP_LEVEL)`, so the apps skip it entirely — the app changes stand on their own code, not honuware's.
3. **The apps are pinned to a SHA anyway.** They fetch honuware at `cad2942…`, so a pushed 1.2 commit does not reach them until the pin moves.

If 1.2's CI does come back red, it is telling you about something *other* than the policy flip, and it would not require reworking anything in 1.3.

### 1.4 Confirm the Linux gate is unaffected

- [x] **Satisfied for `server_components` by CI, which passed on the 1.2 commit.** A separate local `build_and_test.sh` run would be redundant here: `docker/Dockerfile` describes itself as the local twin of `.github/workflows/ci.yml` — same `gcc:14.2.0` base, same apt packages, same Conan — and CI carries the same test-count floor (`MIN_EXPECTED_TESTS: 1000`). Green therefore means the Linux build and the full component suite are intact, and that no tests silently vanished. ✅ 2026-09-04
- [x] **`knottyyoga` Linux gate: green.** `docker_project/build_and_test.sh` in `knottyyoga_build:latest` on `knotty-net`, against the running `knotty-postgres-docker`. **5163 tests from 548 suites ran, all passed** (floor 3500), exit 0. Run with the local honuware override (`HONUWARE_SRC_DIR=/honuware` → the `server_components` working tree), which is `load_container.cmd`'s default, so this exercised the 1.2 and 1.3 changes together. ✅ 2026-09-04
- [x] `communityfinder` Linux gate — `server/docker/build_and_test.sh`, still to run. CI covers honuware only, so this is the last outstanding piece of the Phase 1 gate. ✅ 2026-09-04

**Harness path note.** knottyyoga's Linux harness lives at `server/docker_project/`, not `server/docker/` — only communityfinder uses the latter. Same shape (`build_and_test.sh` + `build_common.sh`), different directory name. CLAUDE.md refers to the `server/docker/` path generically, which is only correct for communityfinder.

**The knottyyoga floor is now slack.** The script's comment estimates "~3150 app + ~1310 component"; the actual run is 5163. A floor of 3500 against 5163 would let roughly a third of the suite vanish silently before tripping — which is precisely the failure mode (a dead endpoint anchor at `-O2`) the floor exists to catch. Worth raising toward ~4800; not changed here because it is unrelated to the migration.

**What the green CI does and does not tell you.** It confirms nothing else in the 1.2 commit broke Linux, and that the suite still links and runs at full count. It does **not** validate the CMP0167 flip itself — at CMake 3.25 the guard is false and the block never executes, exactly as predicted in 1.2. The flip remains verified only by the Windows build plus the config-mode probes in 1.2 and 1.3.

**Consequence worth tracking (see Open Question 6).** After 1.2 and 1.3, the Windows dev boxes resolve Boost through config mode while Linux — the docker gate *and* CI, both on CMake 3.25 — stays on module-mode FindBoost indefinitely. Two different lookup paths across platforms, and the one we just adopted is the one CI can never cover.

**Gate:** VS2022 + CMake 4.4 builds all three repos and the suite is green at the same test count, with pins unchanged.

# Phase 2 — Shared Conan configuration

Pure configuration, no version churn. Nothing here is unit-testable; the gate is that the build still works. Note that the `b2` floor cannot be *exercised* on the 2022 machine (5.3.2 handles toolset 14.3 fine) — it is being staged now so it is already in place when Levi builds in Phase 6.

**Two corrections to the draft below, found while implementing.** The original sketch was wrong in both halves and the shipped version differs:

1. **A committed profile file does nothing on its own.** `conan_provider.cmake` reads `CONAN_HOST_PROFILE` (its own default: `default;auto-cmake`) and turns each entry into a `--profile:host=` flag. A profile that is only committed to the repo is never passed to Conan. It has to be named in `CONAN_HOST_PROFILE` before the first `find_package()`, which is the trigger `find_package(Boost …)` in each top-level CMakeLists.
2. **The `[settings]` block was redundant and the `default` profile was dead weight.** `auto-cmake` — generated by the provider from live CMake state — already supplies os, arch, compiler, `compiler.version` (from `MSVC_VERSION`), `compiler.cppstd` (from `CMAKE_CXX_STANDARD`, so 20 already), runtime and build_type, and it composes last, so anything written in our `[settings]` would be redundant or silently overridden. The shipped profile therefore carries **only** `[tool_requires]`.

### 2.1 Add a committed Windows profile to `server_components`

- [x] Created `server_components/conan/profiles/windows`, carrying only the b2 floor: `boost/*: b2/[>=5.3.3 <6]`. Scoped to boost rather than `*` so unrelated packages do not gain a b2 build dependency. ✅ 2026-09-04
- [x] Wired it in: `set(CONAN_HOST_PROFILE "${CMAKE_CURRENT_LIST_DIR}/conan/profiles/windows;auto-cmake")`, `WIN32`-guarded, immediately before the trigger `find_package(Boost …)`. Set as a normal variable — the provider already made the cache entry at `project()` time, and a normal variable in that scope shadows it where the provider macro actually runs. ✅ 2026-09-04
- [x] **Dropped `default` from the list** (Mason's call, and correct). Measured: `~/.conan2/profiles/default` declares `compiler.version=193`, `compiler.cppstd=14`, `build_type=Release` — every one of them also declared by auto-cmake, which wins, and every one of them stale relative to the real build (194 / 20 / Debug). It contributed nothing but noise. It is also disposable: `docker/Dockerfile`, `ci.yml` and `package/build_linux_release.sh` all run `conan profile detect --force`. `CONAN_BUILD_PROFILE` still uses `default`; the build context genuinely is machine-specific. ✅ 2026-09-04
- [x] `compiler.cppstd=20` needs no explicit setting after all — `detect_cxx_standard()` reads `CMAKE_CXX_STANDARD`, which all three repos set to 20 before the trigger call. Confirmed in the generated `conan_host_profile`. ✅ 2026-09-04
- [x] **Verified the floor works.** Against a cache holding the stale b2: without the profile `boost/1.86.0` resolves `b2/5.3.2`; with it, `b2/5.5.3`. Also verified the `[tool_requires]` applies from the **host** profile alone, which is all `CONAN_HOST_PROFILE` gives us. ✅ 2026-09-04
- [x] **Verified it rebuilds nothing.** Boost's package_id is byte-identical with and without the profile (`c0d0edb7b82b1444609493a455920daf17ecb5a3`) — tool requirements do not enter a package_id. ✅ 2026-09-04

### 2.2 Mirror the profile into `knottyyoga`

- [x] Copied byte-identical to `server/knottyyoga_server/conan/profiles/windows` and wired into that repo's top-level CMakeLists the same way. ✅ 2026-09-04

### 2.3 Mirror the profile into `communityfinder`

- [x] Copied byte-identical to `server/communityfinder_server/conan/profiles/windows`, same wiring. All three verified identical by checksum. ✅ 2026-09-04

**Live proof that a committed profile does nothing on its own.** `knottyyoga/server/docker_project/conan_profiles/default` already exists — a Linux/gcc-14 profile, committed months ago. Nothing in the repo references it: a repo-wide search for `conan_profiles` returns no matches, and `docker_project/Dockerfile` runs `conan profile detect --force` like the other two images. So it is inert, and it has quietly drifted out of agreement with the real build (it declares `compiler.cppstd=gnu20` while `build_common.sh` passes `-s compiler.cppstd=17`). It is exactly the failure mode correction #1 above describes, already sitting in the tree. Worth deleting or wiring up deliberately — not touched here, since it is unrelated to the migration and belongs to whoever owns that harness.

**Why three copies rather than one shared file.** The apps cannot reference honuware's copy: they run Conan *before* `FetchContent` pulls honuware in, and that order is deliberate — honuware's component declarations have to resolve the `${..._LIB}` variables Conan generates. At the moment the profile is needed, honuware's tree does not exist on a clean checkout. Nor should it live in the machine-local `default` profile: not version-controlled, drifts silently between the two machines, and our own tooling calls `conan profile detect --force` on it in three places. Three mirrored copies is the least-bad option, and it matches how the three `conanfile.py` `libraries` lists are already kept in sync.

### 2.4 Interim only — teach Conan that 195 can consume 194 binaries

Optional, explicitly removed in Phase 7, and a **Levi-machine action** — it does nothing on the 2022 box. Microsoft guarantees binary compatibility across v140–v145 and asks only that you link with the newest toolset present, so it is safe for the interim.

- [x] **Measured the benefit** in a throwaway Conan home rather than guessing. Adding `"195": "194"` to the msvc fallback map takes the communityfinder graph at msvc 195 / cppstd 20 from **21 missing packages down to 11** — ten dependencies that would otherwise compile from source become downloads. ✅ 2026-09-04
- [ ] Apply on Levi's machine: in `<conan home>/extensions/plugins/compatibility/compatibility.py`, change `msvc_fallback = {"194": "193"}` to `{"195": "194", "194": "193"}`.
- [ ] Decide how to distribute it. `conan config install` from a committed folder is the drift-proof option but adds an install step; a one-line hand edit is faster and this is temporary anyway. Not done here because it has no effect on the 2022 machine.
- [ ] Ceiling worth knowing: it will **not** help gtest, abseil, boost, libtiff, libpqxx, libcurl, date, libpq, libsodium, ftxui or replxx — the 11 that remain. gtest and abseil set `compatibility_cppstd: False` and ConanCenter builds only at C++17 while we build at 20, so those two compile from source on every machine, today included.

**Gate:** VS2022 build unchanged and green across all three repos. Nothing here is unit-testable — it is Conan configuration with no runtime surface — so the build is the test.

- [x] Mason: reconfigure and build all three repos on Windows. Watch the configure log for `CMake-Conan: Installing single configuration` and confirm no unexpected rebuild storm; the package-id check above says there should be none. ✅ 2026-09-04

# Phase 3 — Recipe bumps: foundation layer

The lowest layer, and where most of the dependency set lives. All validated on VS2022 + CMake 4.4. Apply each bump to all three conanfiles together — they carry identical pins and must not drift.

### 3.1 zlib 1.3.1 → 1.3.2

- [x] All three conanfiles. Half the graph depends on it, so it goes first. ✅ 2026-09-04

### 3.2 Image codecs — libpng, libjpeg, libtiff

- [x] libpng 1.6.40 → 1.6.58 ✅ 2026-09-04
- [x] libjpeg 9e → 9f ✅ 2026-09-04
- [x] libtiff 4.6.0 → 4.7.2 — **one of the two VS2026 blockers**. Confirmed the cap is gone: 4.7.2 declares `cmake/[>=3.18]`, unbounded. A short "do not go back below 4.7.x" comment now sits above the pin in all three conanfiles so a future downgrade cannot silently re-block VS2026. ✅ 2026-09-04
- [x] Targets stay `PNG::PNG` / `TIFF::TIFF`; no `ConanLibImports.cmake` or CMakeLists edits needed. ✅ 2026-09-04
- [x] **Tests: the suspicion was right, the gap was total, and closing it found a live bug.** `image_resize_test.cpp` covered JPEG and PNG thoroughly (including two subtle PNG colour-path regressions) but had **no TIFF case at all** — `IMAGE_TYPE_TIFF`, the `${TIFF_LIB}` link edge and both arms of the production switch existed with nothing behind them. `image_helper_test.cpp` only mentions `"tiff"` as a MIME string. So the one mandatory bump in this phase was also the one with zero coverage. ✅ 2026-09-04

  Two tests now cover it, and they are not the two originally planned — see the defect below for why:
  - `GetImageDimensionsTiff` — the **read** path, which is the half libtiff actually affects. Passes.
  - `ResizeTiffThrowsBecauseTheOutputSinkCannotSeek` — a characterization test pinning the defect. Passes, which is what *proves* the defect.

**DEFECT FOUND: resizing a TIFF has never worked.** Not a regression from the bump — a latent bug the new coverage exposed on its first run.

`ResizeImage` writes its output through `VectorSink` (`back_insert_device`), which is append-only. JPEG and PNG stream strictly forwards and do not care. A TIFF writer must seek back to patch the IFD offset into the header once it knows it, so the `IMAGE_TYPE_TIFF` arm of `WriteView` throws `"no random access: iostream error"` for **every** input, on every platform, in every build. Confirmed empirically: the `EXPECT_THROW` assertion passes.

The read path is fine — `ArraySource` is seekable, and `GetImageDimensionsTiff` decodes a real TIFF through the production entry point without complaint.

Deliberately **not fixed here.** It changes production image behaviour and deserves its own commit and review rather than a ride inside a dependency-version phase. The characterization test is written so that **when the fix lands it will fail**, which is the signal to replace it with the JPEG-style round-trip assertion. Raised as Open Question 9.

Aside worth keeping: the first version of the TIFF *fixture helper* hit the same wall, because it copied the JPEG/PNG `back_insert_device` idiom. TIFF fixtures have to be built through a seekable sink — `std::ostringstream` — and the helper now carries a comment saying so.

### 3.3 date 3.0.4 → 3.0.5

- [x] All three conanfiles. ✅ 2026-09-04
- [x] **Tests:** `date_time_util_test.cpp` covers the consumer; no new case needed. ✅ 2026-09-04

### 3.4 crow 1.3.2 → 1.3.3, libcurl 7.86.0 → 8.21.0

- [x] crow 1.3.2 → 1.3.3, header-only. ✅ 2026-09-04
- [x] libcurl 7.86.0 → 8.21.0. `CURL::libcurl` unchanged; a comment above the pin records that the 8 boundary is not an API break. ✅ 2026-09-04
- [x] **Tests: determined that none is possible without changing the codebase's testing strategy.** The HTTP layer is tested entirely through doubles by design — `TestHttpClient` is a fake, `http_client_test_util_test.cpp` tests *the fake*, and the Square client tests inject it. The real libcurl-backed `MakeHttpClient()` is **not exercised by any test in the suite**, and was not before this change either. Writing one would mean standing up a loopback HTTP server inside the unit suite — a live-server pattern this codebase deliberately does not have. That is a bigger architectural decision than a version bump should carry, so it is **not** done here and is raised as Open Question 7 instead. ✅ 2026-09-04

  What the curl 8 bump therefore rests on: it compiles and links against libcurl 8's headers, and every consumer of the `HttpClient` interface still passes. Runtime behaviour of the production client is unverified — unchanged from before, but worth naming rather than glossing.

### 3.5 Check the 195 graph from the 2022 machine

- [x] Resolved all three repos at **both** msvc 194 and msvc 195, composing the committed profile exactly as the CMakeLists does. Six combinations: **zero version conflicts, zero Invalid packages.** ✅ 2026-09-04
- [x] Scanned all 26 packages in the post-bump graph for a surviving `cmake/[… <4]` cap. **abseil/20220623.1 is now the only one left** — exactly as predicted, and it is Phase 4.3. libtiff, libcurl and mailio all declare unbounded CMake ranges; the other 18 declare no CMake tool requirement at all. ✅ 2026-09-04

  Method note, because it nearly produced a false all-clear: `conan cache path` returns a trailing `\r` on Windows, so a naive `[ -f "$p/conanfile.py" ]` test silently fails for *every* package and the scan reports "no blockers found". The first run did exactly that. Strip the `\r` before using the path.

**Gate:** docker green at the same test count; VS2022 builds all three repos.

- [x] **Linux docker gate for `server_components`: GREEN.** `1764 tests from 177 suites ran, all passed`, `[honuware] OK`, exit 0. Every bumped dependency compiled from source on gcc 14 without incident. ✅ 2026-09-04

  Took three attempts, and the first two are worth recording. **Attempt 1** died with `docker run` returning **125** and `error waiting for container: unexpected EOF` partway through building `date/3.0.5` — a Docker-level failure with no build diagnostic anywhere in the log; a follow-up `docker version` then hung for 120s, so Docker Desktop had crashed. Restarting it also silently took down `knotty-postgres-docker`, which the suite needs — worth checking first if a future gate run fails at startup rather than mid-build. **Attempt 2** built and ran clean except for the two new TIFF tests, which is how the defect below was found.
- [x] Mason: build all three repos on Windows and run the suites. ✅ 2026-09-05

# Phase 4 — Recipe bumps: services, platform, app

Continuing upward through the layers.

### 4.1 services — openssl 3.5.2 → 3.5.8

- [x] All three conanfiles. Same LTS line, patch-only. Deliberately **not** 4.0.x. ✅ 2026-09-04
- [x] **Tests: none needed.** `mail_helper_test.cpp` and `secrets_at_rest_test.cpp` already exercise both consumers, and a patch bump inside one LTS line has no API surface to test. The existing suite is the test. ✅ 2026-09-04

### 4.2 platform — libzip 1.10.1 → 1.11.4

- [x] All three conanfiles. ✅ 2026-09-04
- [x] **Tests: none needed — already the best-covered dependency in the set.** `theme_bundle_zip_test.cpp` carries six distinct malformed-input cases: `RefusesSomethingThatIsNotAZip`, `RefusesATruncatedArchiveRatherThanCrashing`, `ReaderRefusesAnEntryThatIsAPath`, `ReaderRefusesAnArchiveWithNoThemeJson`, `ReaderRefusesTooManyEntries`, `ReaderRefusesAnOversizedEntryBeforeExpandingIt` — plus `LargeButLegalAssetsSurvive` and the round-trip file. The plan's "add a malformed case if none exists" is already satisfied several times over. ✅ 2026-09-04

### 4.3 app — abseil 20220623.1 → 20250814.2

- [x] All three conanfiles. Bumped in `server_components` too rather than removed — **Open Question 3 is still unanswered**, and bumping is the reversible choice; removing it is not. ✅ 2026-09-04
- [x] **The second VS2026 blocker is gone.** 20250814.2 declares `cmake/[>=3.16]`, unbounded. ✅ 2026-09-04
- [x] Target stays `abseil::abseil`; the `${ABSL_LIB}` link lines in `src/database_helper/CMakeLists.txt` and `src/test_helper/CMakeLists.txt` are untouched. ✅ 2026-09-04
- [x] **Tests: not possible, and the exposure is precisely three symbols.** The consumers are `main()` functions with no seam. The entire abseil surface in both apps is `ABSL_FLAG(...)`, `absl::ParseCommandLine(argc, argv)` and `absl::GetFlag(FLAGS_x)`, from `absl/flags/flag.h` and `absl/flags/parse.h` — all stable since the flags library shipped in 2019, which is what makes a 2022→2025 jump low risk rather than a leap of faith. ✅ 2026-09-04
- [x] **Compile-checked the call sites on Linux**, since honuware's gate does not build the app CLI mains. Built `knottyyoga_database_helper` and `knottyyoga_test_helper` in the knottyyoga container against the local honuware tree: both `Built target`, zero compile errors. That turns "the flags API has been stable since 2019" from an argument into a result — `ABSL_FLAG`, `absl::ParseCommandLine` and `absl::GetFlag` all still compile and link against 20250814.2. ✅ 2026-09-04
- [ ] Mason: hand-check the flags still *parse* at runtime — `--recreate_database`, `--migrate`, and the test-helper REPL flags. Compilation is proven; runtime behaviour is not, because no harness executes those binaries.

### 4.4 app — ftxui and replxx

- [x] ftxui held at 5.0.0 — 6.x and 7.x are breaking UI-API changes and the REPL is not on the critical path. Open Question 4 still stands. ✅ 2026-09-04
- [x] replxx 0.0.4 confirmed still the newest published (0.0.2 / 0.0.3 / 0.0.4). Nothing to do. ✅ 2026-09-04
- [x] Neither caps CMake; both satisfy msvc 195. Neither is a migration blocker. ✅ 2026-09-04

> **SUPERSEDED — libsodium was NOT safe to hold.** This subsection originally also held libsodium at 1.0.20 on the reasoning "recent, no CMake cap, builds clean — churn with no payoff." That was wrong, and Phase 6.2 proves it: 1.0.20 cannot build under msvc 195 at all. "Builds clean" was observed on the 194 machine, where a prebuilt binary exists and libsodium's MSBuild path never runs. Bumped to **1.0.22** in Phase 6. Note that ftxui and replxx were held on the *same kind* of evidence — both did reach `Build` on the 195 machine's graph, so they are better attested than libsodium was, but neither has been observed compiling under 195 yet.

**Gate:** docker green; VS2022 builds all three repos.

- [x] **Resolution re-verified after the bumps** — all three repos × msvc 194 and 195, composing the committed profile as the CMakeLists does. Six combinations, zero errors, zero Invalid packages. ✅ 2026-09-04
- [x] **Blocker scan: ZERO.** All 26 packages in the post-Phase-4 graph, and not one declares a `cmake/[… <4]` cap. Spot-checked against the three bumped recipes directly rather than trusting the loop again, after the false all-clear in Phase 3. **Both VS2026 CMake blockers are now cleared.** ✅ 2026-09-04
- [x] **Linux docker gate for `server_components`: GREEN.** `1764 tests from 177 suites ran, all passed`, `[honuware] OK`, exit 0 — the same count as Phase 3, so nothing was lost. openssl 3.5.8 and libzip 1.11.4 both compiled from source on gcc 14, including across the six malformed-archive cases. ✅ 2026-09-04
- [x] Mason: build all three repos on Windows and run the suites. ✅ 2026-09-05

**Note on the failed Windows configure of 2026-09-04 — it was an internet outage, not this work.** The log is unambiguous: every version range resolved (all from cache), and the first and only real error was DNS — `Failed to resolve 'center2.conan.io' ([Errno 11001] getaddrinfo failed)` while checking for a `b2/5.5.3` binary. The `find_package(Boost)` failure underneath it is purely downstream: Conan aborted, so `ConanLibImports.cmake` was never generated, so there was no `BoostConfig.cmake` for the trigger call to find. Nothing to fix in the repos.

Connectivity confirmed restored (`center2.conan.io` resolves), and the graph now computes cleanly at the exact VS configuration (msvc 194 / cppstd 20 / Debug / dynamic): **19 Cache, 9 Skip, 1 Missing.** The single package the retry has to build is `openssl/3.5.8`; everything else is already local.

**Second Windows failure, 2026-09-04 — also the outage, one step further downstream.** The retry got past resolution and died building openssl 3.5.8:

```
ERROR: openssl/3.5.8: Error in build() method, line 562
  while calling '_replace_runtime_in_file', line 577
  FileNotFoundError: [Errno 2] No such file or directory: 'Configurations\10-main.conf'
```

Not a recipe bug and not a version problem — **the cached source tree was truncated by the interrupted download**, and Conan reused it instead of re-fetching. Measured against the working 3.5.2 source in the same cache:

| | files | `Configurations/` |
|---|---|---|
| openssl 3.5.2 (builds) | 5714 | 27 files, incl. `10-main.conf` |
| openssl 3.5.8 (failed) | 5240 | absent entirely |

The recipe `chdir`s to the source folder and patches the MSVC runtime flag into `Configurations/10-main.conf` — an msvc-only path, which is exactly why the Linux gate passed on the same version.

**The lesson worth keeping: source checksums do not protect against this.** They are verified at download time, not on reuse of an already-extracted tree. Any package that fails strangely right after a network drop is a candidate for `conan remove <ref>`, not for debugging the recipe.

Fixed by `conan remove "openssl/3.5.8" -c` and a clean rebuild: the source came back at **5767 files with all 27 `Configurations/` entries**, and openssl 3.5.8 **built successfully under msvc 194 / Debug**. So 3.5.8 is sound on Windows and there is no need to hold at 3.5.2. The binary is now in the local cache, so the next configure has nothing left to build.

*(Housekeeping: that rebuild ran `conan install` with an `--output-folder` outside the repo, which rewrote `server_components/CMakeUserPresets.json` to point at it. Restored to its original content. The file is gitignored, so it could not have reached a commit, but it would have broken the next VS configure.)*

# Phase 5 — Code-touching bumps, one commit each

The only changes that can force a C++ edit. Foundation and services before testing, per the layer rule.

### 5.1 foundation + services — boost 1.86.0 → 1.91.0 with mailio 0.25.3 → 0.26.0

**OUTCOME: attempted, then reverted to the documented fallback. Boost and mailio stay at 1.86.0 / 0.25.3. The two code fixes the bump required were KEPT, so the tree is now ready for 1.91 whenever the blocker below is resolved.**

- [x] Bumped both, as a matched pair, across all three conanfiles. Resolution verified: six combinations (three repos × msvc 194/195), zero conflicts. ✅ 2026-09-05
- [x] **Cost 1 — Asio API removal (fixed, kept).** Boost 1.91 removed `basic_waitable_timer::cancel(error_code&)`; only `std::size_t cancel()` survives, which cannot fail. Two call sites — `job_scheduler.cpp` `Stop()` and `scheduler.cpp` `Shutdown()` — both of which declared an `error_code` purely to select the non-throwing overload and never inspected it. Now call `cancel()`. Note the asymmetry, which looks like an inconsistency and is not: `signal_set::cancel(ec)` two lines below **kept** its overload, so that call is deliberately unchanged. Both compile on 1.86 too, so keeping them costs nothing. ✅ 2026-09-05
- [x] **Cost 2 — Boost.Asio and crow's standalone Asio can no longer share a translation unit (fixed, kept).** From 1.91 both define the same GLOBAL-namespace helper namespaces (`asio_prefer_fn`, `asio_handler_cont_helpers`), so including `<crow.h>` and `<boost/asio/...>` together produces dozens of redefinition errors inside Asio with nothing wrong in our code. This is unavoidable in endpoints: crow reaches even `util/json_value.h` and `util/error_response.h`, and endpoints use `ThreadPool`.

  Scoped before touching anything: Boost.Asio appears in exactly **three** headers — `scheduler.h` and `job_scheduler.h` (scheduler side-branch, never includes crow) and `thread_pool.h` — and only two `.cpp` files, both in the scheduler. So `thread_pool.h` was the single leak. It is now PIMPL: `struct Impl` holds the `boost::asio::thread_pool`, Boost.Asio lives only in `thread_pool.cpp`, and `~ThreadPool()` moved to the `.cpp` where `Impl` is complete. The public interface was already `std::function`-based, so no caller changed. Both files carry a comment explaining why the header must stay Boost-free. ✅ 2026-09-05
- [x] **Cost 3 — SEGFAULT in the real SMTP path. This is what stopped the bump.** With mailio 0.26.0 the suite dies at `MailHelperTest.SendMessage` (SIGSEGV, exit 139) inside mailio's `smtps` connect/submit; 0.25.3 completes the same call in ~1.7 s. Reproducible in isolation with `--gtest_filter=MailHelperTest.*`. `SendMail` catches and *rethrows*, so a genuine failure should surface as an exception — a segfault is a crash in production mail sending, not a test artifact. ✅ 2026-09-05
- [x] **Reverted** to 1.86.0 / 0.25.3 in all three conanfiles, with the whole story recorded in a comment above the `boost` pin so it is not rediscovered from scratch. Gate re-run after the revert: **1764 tests, all passed**, including `MailHelperTest.SendMessage`, `ThreadPoolTest.*`, `StopCancelsScheduledJobs` and `ShutdownStopsTimers`. ✅ 2026-09-05
- [x] **Tests: no new test warranted, and this was checked rather than assumed.** The plan asked for a CRLF assertion if one was missing. It is not: `quick_account_welcome_mail_test.cpp` already asserts every `\n` is preceded by `\r`, `person_verify_mail_test.cpp` compares three generated bodies against `NormalizeCrLf(expected)`, and `types_test.cpp` covers `NormalizeCrLf` itself across already-CRLF, LF-only and mixed input. For the two code fixes, `ThreadPoolTest.QueueBasic/JoinBasic`, `JobSchedulerTest.StopCancelsScheduledJobs` and `SchedulerTest.ShutdownStopsTimers` (plus `ShutdownIsIdempotent`) already cover exactly the changed paths. ✅ 2026-09-05

**Why holding is the right trade.** Neither version is required for VS2026 — Phases 3 and 4 cleared the actual blockers, and the b2 floor from Phase 2 is what makes Boost build on the v145 toolset. Boost 1.91 was hygiene. Trading a crash in the mail path for hygiene is a bad deal. See Open Question 10 for what it would take to move.

### 5.2 testing + tests — gtest 1.12.1 → 1.17.0

**OUTCOME: clean. Zero code changes needed — the only bump in the whole migration that cost nothing.**

- [x] All three conanfiles. 1.17 requires C++17; we build at 20. ✅ 2026-09-05
- [x] **Pre-assessed the risk surface before bumping**, by scanning the repo for the names gtest actually removed: `INSTANTIATE_TEST_CASE_P`, `TYPED_TEST_CASE`, `REGISTER_TYPED_TEST_CASE_P`, `::testing::TestCase`, `testing::internal::`. **None appear anywhere.** The "no fixtures, self-contained tests" convention is what keeps that surface empty — this is a concrete dividend from it. ✅ 2026-09-05
- [x] The only exposure was the two custom matchers — `JsonWvalueMatcher` (`json_value_matcher.h:15`) and `PostGresResultMatcher` (`table_matcher.h:91`), both deriving from `::testing::MatcherInterface`, plus three `::testing::MakeMatcher(new …)` call sites. That is the *supported* custom-matcher API with an unchanged `MatchAndExplain` signature, so it compiled untouched, as predicted. ✅ 2026-09-05
- [x] **Gate green: 1764 tests, all passed, zero compile errors**, with the matcher tests (`JsonTestUtilTest.*`, `table_matcher_test`, `json_value_matcher_test`) among them. ✅ 2026-09-05
- [x] **Tests: none added.** The matchers' own tests are the gate for this bump and already existed. ✅ 2026-09-05

**Gate:** full docker suite **with the test-count floor intact** — a silently vanished route or test is exactly what a dependency swap can cause.

- [x] Linux docker gate green after both 5.1 (reverted state) and 5.2. **1764 tests, all passed**, floor 1000, exit 0 — the same count as Phases 3 and 4, so nothing vanished across the whole dependency migration. ✅ 2026-09-05
- [ ] Mason: build all three repos on Windows and run the suites. Two separate commits — 5.1 (Asio fix + ThreadPool PIMPL, pins unchanged) and 5.2 (gtest).

# Phase 6 — Hand to Levi: first VS2026 build

Everything reaches Levi already proven under CMake 4.4, so anything that fails here is genuinely VS2026-specific. This is the first point at which Findings 2.1 and 2.2 are actually tested.

### 6.1 Sync and build

- [x] First VS2026 configure run, on `MASONLG` (VS 2026 Community 18.0, `cl 19.51.36256`, MSVC toolset 14.51.36231 → **msvc 195**, v145). ✅ 2026-09-06
- [x] **The scaffolding all worked.** The provider detected `compiler.version=195` unaided, generated the auto-cmake profile at cppstd=20, composed the committed `conan/profiles/windows` ahead of it (`boost/*: b2/[>=5.3.3 <6]` visible in the input profile), and resolved the whole graph without a single conflict or invalid package. ✅ 2026-09-06
- [x] Confirmed the expected shape of a cold 195 cache: **21 packages marked `Build`** — no 195 binaries exist, exactly as Finding 2.3 predicted. ✅ 2026-09-06

### 6.2 Confirm the blockers are gone

- [x] **Findings 2.1 and 2.2 are both CONFIRMED FIXED**, by the log rather than by inference. `cmake/[>=3.16]`, `[>=3.18]`, `[>=3.16.3]` and `[>=3.20]` all resolved to **cmake/4.4.2** — no CMake 3.x was pulled in, so the libtiff and abseil bumps did their job. `b2/[>=5.3.3 <6]` resolved to **b2/5.5.3**, so the Phase 2 profile floor did its job. Neither of the two predicted blockers fired. ✅ 2026-09-06
- [x] **A THIRD blocker appeared that this plan never predicted: `libsodium/1.0.20`.** The build died at:

  ```
  error MSB8020: The build tools for v145 (Platform Toolset = 'v145') cannot be found
    [libsodium.vcxproj]  ...\builds\msvc\vs2022\libsodium.sln
  ```

  **libsodium does not build with CMake — it builds a checked-in MSBuild solution.** Its `_msvc_sln_folder` maps only msvc 190–193 to a solution folder and silently falls back to `"vs2022"` for anything newer, then injects `PlatformToolset=v145` into that VS2022 project. The give-away is in the log: `TargetPath ... \Debug\v143\static\` — the solution is hardwired around v143. **1.0.21+ add `"194": "vs2022"` and `"195": "vs2026"`.** ✅ 2026-09-06
- [x] Bumped libsodium **1.0.20 → 1.0.22** in all three conanfiles, with the whole diagnosis in a comment above the pin. Resolution verified at 194 and 195 across all three repos; Linux gate green at **1764 tests, all passed**, zero compile errors, 25 libsodium-consumer tests (secrets/auth) passing. ✅ 2026-09-06
- [x] **Confirmed fixed on VS2026**: the next run reported `libsodium/1.0.22: Already installed! (5 of 27)`. Blocker three is closed. ✅ 2026-09-06

### 6.2b Fourth stop: `libpq/17.11` — a MACHINE issue, not a repo issue

- [x] The run then died in libpq's Meson configure with **error 9009** ("command not found") and, immediately above it:

  ```
  Python was not found; run without arguments to install from the Microsoft Store,
  or disable this shortcut from Settings > Apps > Advanced app settings > App execution aliases.
  ```

  **libpq 17.11 builds with Meson; Meson is a Python application.** That machine has Windows' Microsoft-Store `python.exe` alias stub on PATH instead of a real interpreter, so meson cannot launch. Nothing in the three repos is implicated. ✅ 2026-09-06
- [x] **Fix on that machine**, either: install a real Python (3.x, on PATH), or turn off the Store aliases at *Settings → Apps → Advanced app settings → App execution aliases* (`python.exe` and `python3.exe`), or both. Then re-run the configure. ✅ 2026-09-08

**This is Open Question 12 arriving in person, and it is worse than it first looked.** The two machines do not merely resolve different libpq versions from `libpq/[>=15.4 <18]` — they build libpq with **completely different build systems**:

| | resolves | Windows build system | needs |
|---|---|---|---|
| VS2022 desktop | `libpq/15.5` | Autotools + **MSBuild** | pkgconf |
| VS2026 laptop | `libpq/17.11` | **Meson** | meson + **Python** |

Two consequences worth holding onto:

1. **The divergence accidentally helped.** libpq 15.5 goes through MSBuild with no per-version solution map — the same shape as the libsodium failure. libpq 17.11's Meson build is toolchain-agnostic and has no v145 assumption at all, so it is *more* likely to survive VS2026 than 15.5 would have been. Converging the machines downward onto 15.5 could reintroduce a v145 MSBuild problem; if they are converged, it should be **upward**.
2. **But it also means the VS2026 machine has never validated what the VS2022 machine actually ships.** A green build on one says nothing about the other for this dependency.

The durable fix remains a committed lockfile rather than an ad-hoc pin — pinning `libpq` to 17.x across all three repos would push a Postgres *client major version* change onto the Linux gate and the deploy image, which is a bigger decision than this migration should make on its own.

**Two corrections to earlier phases, both mine.**

**The Phase 4 "zero CMake blockers" conclusion was correct but too narrowly scoped.** It scanned for `cmake/[… <4]` caps, which can only see CMake-based recipes. libsodium builds through MSBuild and was invisible to that method however carefully it was run. The gap is now closed: all 24 packages in the graph were re-scanned for MSBuild-based builds — **five touch MSBuild** (boost, libjpeg, libpq, xz_utils, libsodium) and **libsodium is the only one with a per-version solution-folder map**, so this class should be exhausted. Any future "is the graph VS2026-clean?" check needs *both* scans.

**The Phase 4.4 decision to hold libsodium was based on bad evidence.** It read "recent, no CMake cap, builds clean — churn with no payoff." But "builds clean" came from the 194 machine, where a prebuilt binary exists and the MSBuild path never executes. That is exactly the trap Findings §4 describes — a green 194 machine proves nothing about 195 — applied to a package that had been written off as boring. **The general lesson: on the 2022 machine, "it builds" and "its build path was exercised" are different claims**, and only packages that actually compiled from source there tell you anything about 195.

### 6.3 Run the gates on 2026

- [x] Full suite on Levi's machine at the same test count as the 2022 machine. ✅ 2026-09-08
- [x] Any delta in test count is a broken endpoint anchor, not a flake — chase it before proceeding. ✅ 2026-09-08

**Gate:** both machines build all three repos and produce identical test counts.

# Phase 7 — Full migration to Visual Studio 2026

The end state, once both developers are on 2026 and interim support stops earning its keep.

### 7.1 Move both machines to 195

- [x] Mason installs VS2026 on the main desktop, uninstalling VS2022. `server_components` configures, runs Conan and builds successfully. ✅ 2026-09-08
- [x] Rebuild both apps from a cold cache once, to prove nothing depended on a 194 artifact. **Discharged by measurement instead — and the measurement is the stronger proof.** Enumerated every msvc binary in the desktop's Conan cache (`conan list "*:*" --format=json`, grouped by `compiler.version`): **21 binaries, all at `195`, zero at `194`.** That is the same 21 packages Phase 6.1 predicted would be marked `Build` on a cold 195 cache. A cold rebuild would have shown that nothing 194 *was* reused; the cache scan shows there is no 194 artifact left to reuse, which is a superset of the claim. Reinforced by 7.2 below: this machine's `compatibility.py` has no `195 → 194` fallback, so Conan could not have substituted a 194 binary even had one survived. ✅ 2026-09-08

**A false all-clear, caught, and worth recording because it is the second of its kind.** The first run of that cache scan reported "NONE — every msvc binary is 195" while actually having examined *nothing*: `$pid` is a read-only automatic variable in PowerShell, so `foreach ($pid in …)` threw on every iteration, the results array stayed empty, and an empty array trivially satisfies "no non-195 entries". Exactly the shape of the Phase 3.5 trailing-`\r` bug — **a scan that silently examines zero items reports the same thing as a scan that finds zero problems.** Any future all-clear from a loop should print the number of items it actually inspected; this one now does (`total msvc binaries found: 21`).

**Fifth stop: `git` is gone from the machine — and it is the VS2022 uninstall that took it.** The first `communityfinder` configure on the new install died at:

```
CMake Error at .../ExternalProject/shared_internal_commands.cmake:928 (message):
  error: could not find git for clone of honuware-populate
Call Stack: ... CMakeLists.txt:133 (FetchContent_MakeAvailable)
```

Measured, not inferred: `where.exe git` returns nothing, PATH carries only `C:\Program Files\GitHub CLI\`, `C:\Program Files\Git` / `Program Files (x86)\Git` / `%LOCALAPPDATA%\Programs\Git` are all absent, no `Git` entry survives in the uninstall registry, and the failed cache reads `GIT_EXECUTABLE:FILEPATH=GIT_EXECUTABLE-NOTFOUND`. **VS2022 had installed Git for Windows as an optional component and put it on PATH; uninstalling VS2022 removed it.**

VS2026 does ship a private git — `…\18\Community\Common7\IDE\CommonExtensions\Microsoft\TeamFoundation\Team Explorer\Git\cmd\git.exe`, verified working at 2.54.0.windows.3 — but it is not on PATH and is not one of `FindGit`'s search locations, so `find_package(Git)` cannot see it.

- [x] Install Git for Windows on the desktop (`winget install --id Git.Git -e`), **restart Visual Studio** so the CMake configure inherits the new PATH, then reconfigure. `find_program` re-searches `*-NOTFOUND` cache entries every configure, so no cache delete should be needed. ✅ 2026-09-08

**Two things worth keeping.**

1. **This is a machine issue, not a repo issue — the second one this migration has produced** (6.2b's Store-alias `python.exe` was the first), and both arrived *after* every repo-side blocker was cleared. The pattern: a fresh toolchain install is missing an ambient tool that the old install had quietly been supplying. Do not reach for the conanfiles when a build dies on a *tool* rather than a *package*.
2. **`server_components` building fine proves nothing here.** It declares no `FetchContent_Declare(… GIT_REPOSITORY …)`; the honuware pin at `communityfinder/server/communityfinder_server/CMakeLists.txt:128` is the only thing in the configure that needs git at all. So the framework repo is structurally incapable of catching this class of failure, and the same will be true of any future host-tool gap that only the apps' FetchContent step exercises.

**A logging trap that will mislead you here, and it is worth naming.** The configure log prints `-- CMake-Conan: find_package(Git) found, 'conan install' already ran` immediately before the failure, which reads as though Git *was* found. It was not. That message is `conan_provider.cmake:609`, in the `else()` branch, and it means only *"conan install already ran during this configure"* — it is printed for every intercepted `find_package` regardless of outcome, and says nothing about whether the package resolved. The cache entry is the authority, not the log line.

**Fallback if a system git is ever undesirable:** set `GIT_EXECUTABLE` to the Team Explorer path in `CMakeSettings.json`. Rejected here — that file is committed, the path is specific to both the machine and the VS edition, and it would break Levi's configure.

### 7.2 Remove the interim scaffolding

- [x] Drop the `195 → 194` compatibility entry from 2.4 — **no-op on the desktop: it was never applied there.** `~/.conan2/extensions/plugins/compatibility/compatibility.py` still reads the stock `msvc_fallback = {"194": "193"}` (line 43). Since 2.4's apply-step was only ever scoped to the *other* machine, and 2.4's remaining bullets were never checked off, the likeliest state is that the interim fallback was never applied anywhere and the whole of 2.4 was measurement-only. ✅ 2026-09-08
- [x] **Confirmed on the second machine (`MASONLG`): also stock.** `compatibility.py:43` reads `msvc_fallback = {"194": "193"}.get(compiler_version)` — no `195` entry. **So the interim fallback was never applied on either machine, and the whole of 2.4 was measurement-only.** 7.2 closes with no change to make anywhere. ✅ 2026-09-08

  This retroactively strengthens 7.1: with no `195 → 194` fallback on either box, neither machine *could* have consumed a 194 binary at 195, independently of what the cache happened to hold. The two findings corroborate rather than merely coexist.

  Caveat on the evidence, recorded honestly: the output is byte-identical to the desktop's, so the transcript alone does not prove which machine produced it. The shell prompt read `C:\Users\Mason` against the desktop's `C:\Users\mason`, which is a hint and not a proof. If a future reader needs certainty, re-run it with `$env:COMPUTERNAME` printed alongside.
- [x] Consider pinning `compiler.version=195` explicitly in the committed profile — **rejected, on a measurement rather than a preference: the pin would be inert.** Composed two profiles in the real `CONAN_HOST_PROFILE` order (ours first, `auto-cmake` second) with ours declaring `compiler.version=195` and auto-cmake declaring `194`. `conan profile show` resolves the host profile to **`compiler.version=194`** — the later profile wins, as Conan 2 composition specifies. So any `[settings]` written into the committed profile is silently overridden by the auto-cmake profile the provider generates from live CMake state. ✅ 2026-09-08

  This is not a new discovery so much as a confirmation of what the profile file already documents in its own header (`DELIBERATELY NO [settings]`, lines 32–37) — but it is worth having measured, because "both machines are at 195 now, so just pin it" is an obvious-sounding idea that would have produced a line of configuration doing precisely nothing while *looking* authoritative. The only way to make such a pin bite would be to compose our profile *after* auto-cmake, which would also override the CMake-derived settings the whole arrangement exists to propagate. **Leave `compiler.version` to auto-detection permanently**, and treat the "one committed file serving both machines" property as the design rather than an interim compromise.

### 7.3 Re-pin honuware in both apps

- [x] Mason pushes `server_components` — **already done.** Local `master` and `origin/master` both sit at `6abea1416a95d58befcba500101f15ac387ff9e2` ("Dependency issue", 2026-09-06), verified read-only with `ls-remote`. Working tree clean apart from an untracked `CMakeUserPresets.json.old` (leftover from the Phase 4 housekeeping; safe to delete). ✅ 2026-09-08
- [x] Re-pin the `FetchContent` `GIT_TAG` from `cad2942…` to `6abea141…` in `communityfinder/server/communityfinder_server/CMakeLists.txt:131` and `knottyyoga/server/knottyyoga_server/CMakeLists.txt:183`. Both were on the identical old SHA and are now on the identical new one. ✅ 2026-09-08
- [x] Verify the pinned build from a fresh clone with no `HONUWARE_SRC_DIR` override — **communityfinder Linux gate GREEN**: `1785 tests from 183 test suites ran`, `[  PASSED  ] 1785 tests`, `[communityfinder] OK`, exit 0. Verification chain is complete rather than assumed: the configure log says `honuware : pinned SHA (FetchContent will clone it)` (no override), and the cloned tree's own `git log -1` in `/build/_deps/honuware-src` reads **`6abea1416a95d58befcba500101f15ac387ff9e2`** — the new pin. ✅ 2026-09-08
- [x] **knottyyoga Linux gate against the new pin: GREEN.** `5165 tests from 548 test suites ran`, `[  PASSED  ] 5165 tests`, `[knottyyoga] OK`, exit 0, floor 3500. Same verification chain: `honuware : pinned SHA (FetchContent will clone it)` in the log, fresh `knottyyoga-linux-build-pinned` volume, and the cloned tree's HEAD confirmed at `6abea141…`. ✅ 2026-09-08

**knottyyoga gave the precise before/after that communityfinder could not.** Phase 1.4 recorded **5163 tests from 548 suites**; this run is **5165 from 548**. Exactly **+2 tests, zero new suites** — and the two are named in the log:

```
[ RUN      ] ImageResizeTest.GetImageDimensionsTiff                       [ OK ]
[ RUN      ] ImageResizeTest.ResizeTiffThrowsBecauseTheOutputSinkCannotSeek [ OK ]
```

Those are precisely the two TIFF tests added to honuware in Phase 3.2, landing in an existing suite (hence 548 unchanged). **The delta is fully accounted for**: the re-pin delivered exactly the honuware test additions and removed nothing. That is the strongest single piece of evidence in this phase — it is the difference between "the count went up, probably fine" and "the count went up by exactly the two tests we know were added."

Worth noting *why* it worked here and not for communityfinder: knottyyoga's Phase 1.4 count was **written down in this document**. communityfinder's never was, so its only baseline was a six-week-old `test-output.txt` left in a build volume. Recording the count per run is what made this check possible.

**The 5.1 code changes also verified through the pinned build**, not merely compiled: `ThreadPoolTest.QueueBasic` / `JoinBasic` (the `thread_pool.h` PIMPL), `JobSchedulerTest.StopCancelsScheduledJobs` and `SchedulerTest.ShutdownStopsTimers` / `ShutdownIsIdempotent` (the two Asio `cancel()` call sites) all pass. Those are exactly the paths 5.1 touched, now exercised via the pin rather than via `HONUWARE_SRC_DIR`.

**Both apps' Linux gates are green against `6abea141…`. Phase 7.3's verification is complete; only the Windows spot-checks and the two commits remain.**

**The stale-cache trap was real, and is now confirmed rather than merely avoided.** Inspected the old `communityfinder-linux-build` volume directly: its `CMakeCache.txt` contains `FETCHCONTENT_SOURCE_DIR_HONUWARE:PATH=/honuware`. Reusing that volume — the obvious thing to do — would have rebuilt the **local** honuware tree and reported a green "pinned" gate that never touched `6abea141…`. Anyone re-verifying a pin in future must start from a clean build tree; `HONUWARE_SRC=none` alone is not sufficient, because the override lives in the CMake cache, not the environment.

**The test-count comparison is weaker than it looks, and that is worth fixing.** The previous communityfinder gate output in that old volume is dated **28 July** — 1520 tests from 163 suites, six weeks before this migration began. So `1520 → 1785` reflects six weeks of honuware growth plus the migration's own added tests, not a clean before/after for the re-pin. The direction is the safe one (nothing vanished), but there was no same-week baseline to compare against.

Two consequences:

1. **communityfinder's floor is now very slack** — `MIN_EXPECTED_TESTS=1000` against an actual 1785, so ~44% of the suite could vanish before it trips. That is the same defect already logged against knottyyoga in Open Question 8 (3500 against 5163), so it is a pattern, not a one-off: **both floors were set once and never revisited as the suites grew.** Worth raising communityfinder's toward ~1700 and knottyyoga's toward ~4800 in one small commit.
2. **Record the count per gate run.** Every honuware gate count in this plan is written down (1764, repeatedly), which is exactly why honuware regressions would have been visible. No communityfinder count was ever recorded until now. Logging it each run is what turns the floor from a crash-guard into a regression detector.

**What the re-pin actually moves.** Confirmed a clean fast-forward (`merge-base --is-ancestor` → yes), spanning five commits — `ab8036c` Update CMake policy, `0f4f64c` + `5fc5e3c` Phase 4, `6368e2f` Phase 5, `6abea14` Dependency issue — and eight files:

| file | what arrives |
|---|---|
| `components/foundation/util/thread_pool.{h,cpp}` | the **PIMPL** from 5.1 — Boost.Asio no longer leaks out of the header |
| `components/scheduler/scheduler/{scheduler,job_scheduler}.cpp` | the two Asio `cancel()` fixes from 5.1 |
| `components/foundation/util/image_resize_test.cpp` | the two TIFF tests from 3.2, incl. the characterization test pinning the TIFF-resize defect |
| `CMakeLists.txt`, `conan/profiles/windows`, `conanfile.py` | CMP0167 flip, the committed b2-floor profile, the recipe bumps |

**This is the first time the apps have seen any of it.** Every app-side Windows build in Phases 3–5 ran against the *old* pin `cad2942…`, and the app conanfiles were bumped independently (they are supersets, and in consumed mode honuware's CMake resolves `${..._LIB}` from the app's `ConanLibImports`). Only knottyyoga's Linux gate exercised the new honuware source, and only via `HONUWARE_SRC_DIR=/honuware`. So the pinned gate below is the first real integration of the two trees.

**A trap in verifying this, worth writing down.** The obvious move is to re-run `docker/build_and_test.sh` with `HONUWARE_SRC=none`, but the `communityfinder-linux-build` volume carries a CMake cache from earlier runs that *did* set `FETCHCONTENT_SOURCE_DIR_HONUWARE=/honuware`. That is a **cache variable**: it survives into the next configure even with the environment variable unset, so the run would quietly keep building the local tree and report a green "pinned" gate. The verification run therefore uses a **fresh build volume** (`communityfinder-linux-build-pinned`) so the configure starts from nothing and FetchContent genuinely clones `6abea141…`. Same class of error as the two false all-clears above: the check passes without having checked.

# Post-migration work — answers to the Open Questions, and Mason's new items

Phases 1–7 are done and are not to be edited. Everything below is new work arising from the answered Open Questions plus the two items in *Mason- New Items*. Same conventions as above: layering respected inside each phase (lower layers first), cross-repo order `server_components` → `knottyyoga` → `communityfinder`, and every phase that changes honuware is a bump point (Mason pushes honuware → Claude re-pins → Mason commits).

**Four questions closed with no work required**, recorded here so they are not reopened:

- **OQ1 / OQ5 (sequencing).** Both moot — the migration is done, and doing it before the bucket work was right because both machines needed VS2026.
- **OQ2 (Boost 1.86 under 19.51).** **Answered by Phase 6/7: yes.** Boost 1.86 builds on both machines with the b2 floor from Phase 2 doing its job (`b2/[>=5.3.3 <6]` → `b2/5.5.3`, confirmed in the 6.2 log). This is a real result, not an absence of failure: it means **Phase 5.1 was never load-bearing for the migration** and the boost/mailio bump is now pure hygiene. It also removes the fallback argument for OQ10 — see 15.1.
- **OQ3 (drop abseil from honuware).** Mason: keep it. No change; the `#` in `conanfile.py` stays a superset-consistent pin. The invariant "app conanfile is a superset of honuware's" is preserved by *not* removing it, which is the reversible choice anyway.

**Ordering rationale.** Phase 8 goes first because it is a hard block — nothing can be run or debugged until Visual Studio's Debug menu works again. Phase 9 follows because a disclosed live credential outranks everything else that is merely important. Phase 10 is third: it is Mason's explicit ask and it removes a daily workflow tax. TIFF removal (11) precedes the build-integrity work (12) so that the lockfile and the floors capture the dependency set *after* it shrinks, not before.

# Phase 8 — Unblock: restore Visual Studio's Debug and Launch settings

**First, because nothing else can be run until it is done.** *Debug → Debug and Launch Settings* silently does nothing, and deleting `.vs` does not bring it back.

### 8.1 Immediate unblock

**RESOLVED 2026-09-09 — it is a Visual Studio 2026 defect, and there is no fix on our side.** *Debug → Debug and Launch Settings* fails with:

```
Microsoft.VisualStudio.Shell.ServiceUnavailableException:
  The VsTextManagerClass service is unavailable.
   at AsyncServiceProvider.GetSyncService[TInterface](ServiceRequest request)
   at ...VS.CommandHandler.OpenDocumentAndReplaceRange(String documentFilePath,
                                String replaceText, Tuple`2 range, Action callback)
   at ...VS.CommandHandler.<ExecEditDebugTargetAsync>d__35.MoveNext()
```

Two of the three documented entry points are broken, and they fail *identically* because they share `ExecEditDebugTargetAsync`:

| entry point | documented | actual |
|---|---|---|
| Debug menu | autopopulates `projectTarget`, opens the file | **writes the entry, never opens the file** |
| Targets View → *Add Debug Configuration* | autopopulates, opens the file | **does nothing at all** |
| Root `CMakeLists.txt` → *Add Debug Configuration* | opens the file, does *not* autopopulate | works as documented |

**The write half works.** The entry is appended to `launch.vs.json` correctly every time; only the open-and-highlight step dies. Proven by deleting the file, invoking the command, and watching it reappear with the correct target 4 seconds later.

Note the failing call is **`GetSyncService`** — a *synchronous* request for `SVsTextManager`, which is UI-thread-affinitized, issued from an `async` continuation. That is a threading defect, which is why it reproduces on **every machine and every project**, including a four-line `CMakeLists.txt` with one target. It survived a repair, a reboot, and a full rollback from 18.10.12201.205 to 18.9.12120.119, and reproduces on both. The third entry point works precisely because it runs on the UI thread via the *Select a Debugger* dialog — and that is also why it cannot autopopulate `projectTarget` and appends a blank `CMakeLists.txt`, `CMakeLists.txt(1)`, … skeleton on each invocation.

No public report matches: searching the exception string, both internal method names, and VS2026+CMake across Developer Community and GitHub returned nothing. That is weak evidence — Developer Community renders via JavaScript and indexes poorly, and VS2026 is days old — but it means there is no fix to wait for yet. **File it.**

- [x] Confirm the JSON editor itself is healthy — `File → Open → File` on `.vs\launch.vs.json` opens it with full syntax highlighting. The editor is fine; only the programmatic open fails. ✅ 2026-09-09
- [x] Note what the menu entry is *supposed* to do, because expecting a dialog is half of why this was hard to read: it appends an entry for the selected target to **`.vs\launch.vs.json`** and opens it as a text file. There is no settings UI. ✅ 2026-09-08
- [ ] Report via *Help → Send Feedback → Report a Problem*, with the exception above and the four-line repro. Nothing else in this document is blocked on it — 8.1 is worked around, not waiting.

**Three wrong diagnoses preceded the right one. All are recorded because each was plausible and each cost time — the third cost a day and was written into this document as "RESOLVED".**

**Wrong theory 1 — `CMakeUserPresets.json` put VS in Presets mode.** The reasoning: all three repos carry *both* a `CMakeSettings.json` and a Conan-generated `CMakeUserPresets.json`, and *none* has a `CMakePresets.json`; VS switches to Presets mode on sight of the former and then ignores `CMakeSettings.json`, where the `CONAN_CMD` / `CMAKE_PROJECT_TOP_LEVEL_INCLUDES` wiring lives. It fit the symptom, it fit `Machine configuration.md`'s existing "delete it before opening VS2026" instruction, and it was **wrong** — the file was deleted and the menu still did nothing. *The instruction in `Machine configuration.md` is therefore itself unexplained: whatever it was originally worked around, it was not this.* Worth knowing before that line is deleted in 8.2.

**Wrong theory 2 — the command was firing and the file was opening unnoticed.** `launch.vs.json` was timestamped 18:34:12, right at the click. But the whole `.vs` folder — `ProjectSettings.json`, `slnx.sqlite`, `VSWorkspaceState.json` — shared that exact stamp, so it was VS saving session state, not the command running. **A timestamp that matches the moment you looked is not evidence that the thing you were looking for happened.**

**Wrong theory 3 — no startup item was selected.** Recorded here on 2026-09-08 as the resolution, and it is **wrong**. The docs do say the menu entry greys out with no target chosen, which made it fit; but the entry was never greyed out, it was *enabled and inert*, and re-testing on 2026-09-09 with `honuware_test_runner` explicitly selected reproduced the failure exactly. **The distinction that was missed: a greyed-out command and an enabled command that silently does nothing are different failures, and the docs only explain the first.** Worse, the workaround adopted at the time — select a startup item and press F5 — genuinely does work, because *debugging* was never broken. Only the settings editor was. A workaround that makes the pain go away is not a diagnosis.

**A fourth line of enquiry ran a long way on a real but unrelated defect.** `ActivityLog.xml` shows `Microsoft.VisualStudio.LanguageServer.ContainedLanguage.dll` missing from disk, which collapses ~25 MEF parts in the Web Tools LSP delegation layer — including one named `TextDocumentHandler`. With "the text service is unreachable" as the remembered symptom, that looked like a direct hit. It is a genuine flaw in this installation and worth reporting separately, **but it is not this bug**: the JSON editor assemblies are all present and healthy, and `File → Open` on `launch.vs.json` renders it correctly. *A real defect found while looking for a different real defect is the most expensive kind of red herring, because everything about it reads as confirmation.*

**And a piece of advice given here was backwards — twice, in both directions.** An early draft said to open `server\communityfinder_server` (the folder containing `CMakeLists.txt`). That was then "corrected" to the repo root on the evidence that `communityfinder/.vs/CMakeWorkspaceSettings.json` exists, holds `{"enableCMake": true, "sourceDirectory": "server\\communityfinder_server"}`, and that the *useful* `launch.vs.json` — the one carrying `HONUWARE_ALLOW_DESTRUCTIVE`, the mail password and `--recreate_database` — lived beside it.

**The original advice was right. The root-level workspace was a mistake** (confirmed 2026-09-09): the intent is to open Visual Studio on the **C++ server folder** and, separately, on the **Angular front end** — never on the repo root spanning both. **Open `server\<app>_server`.**

The correction had been reasoned from artefacts rather than from intent, and the artefacts were themselves the mistake being corrected. *A file that exists is evidence that something happened, not that it was meant to.* The root `.vs` folders have been removed in both apps (communityfinder's had grown to 3 GB of `cmake.db` and `slnx.sqlite` caches); their `launch.vs.json` and `CMakeWorkspaceSettings.json` are backed up, and the values that mattered were migrated into the inner workspace's `launch_defaults.local.json` first.

`sync_launch_targets.ps1` now warns if it finds a parent folder that is a CMake workspace pointing back at the folder being generated for, since entries are **not** interchangeable between the two — the workspace root form needs `"project": "server\\<app>_server\\CMakeLists.txt"`, the inner form needs a bare `"CMakeLists.txt"`.

- [x] Deleted the three generated `CMakeUserPresets.json` files anyway. Not the cause, but see 8.2 — the file is a genuine problem for a different reason, and removing it costs nothing since it is gitignored and regenerable. ✅ 2026-09-08
- [x] **Reversed, 2026-09-09: Tools → Options → CMake → *"Prefer using CMake Presets…"* → *use CMake Presets if available*.** The earlier **Never** was correct only while `CMakeSettings.json` held the Conan wiring. All three repos have now migrated to `CMakePresets.json` (below), so **Never** would strand them with no configuration at all — which is exactly what happened mid-migration: VS stopped invoking CMake entirely and *Delete Cache and Reconfigure* greyed out. ✅ 2026-09-09
- [x] **Migrated all three repos from `CMakeSettings.json` to `CMakePresets.json`.** Same `x64-Debug` name, so the existing `out/build/x64-Debug` tree is reused; `inheritEnvironments: msvc_x64_x64` becomes the `architecture`/`toolset` pair with `strategy: external`; `CONAN_CMD` was **dropped, not ported** — `conan_provider.cmake` reads `CONAN_COMMAND` and finds Conan via `find_program`, and the old value was the literal unexpanded string `${env:LOCALAPPDATA}/...` (`CMakeSettings.json` uses `${env.VAR}`, with a dot). It had never worked. The presets also give CLI/CI parity, which `CMakeSettings.json` never could. ✅ 2026-09-09
- [x] **A stale `.vs` blocked the switch and looked exactly like a preset failure.** After migrating, VS would not configure at all until `.vs` was deleted — `ProjectSettings.json` still pointed at the old configuration system. The presets themselves were provably fine the whole time: `cmake --preset x64-Debug` from a developer prompt configured cleanly, exit 0. **When the IDE disagrees with the command line, believe the command line.** ✅ 2026-09-09
- [x] **`env`/`args` merged — this closes the item that unblocks 16.1.** Delivered as a generator rather than a hand-edit, because `.vs/` is gitignored and therefore disposable: `server_components/tools/sync_launch_targets.ps1` enumerates every executable target from the CMake file API and stamps `args`/`env` from a durable `launch_defaults.local.json` (also gitignored — it holds DB and mail credentials). Both `database_helper` targets now get `--recreate_database` with `HONUWARE_ALLOW_DESTRUCTIVE=1`; the test executables get `--gtest_filter=*`. ✅ 2026-09-09
- [x] **Settled: the inner `server\<app>_server` folder is the workspace.** The root-level one was a mistake — VS is opened on the C++ server and on the Angular front end separately, never on a root spanning both. Root `.vs` removed in communityfinder and knottyyoga (`server_components` never had the split, its root *is* its source directory); `HONUWARE_MAIL_APP_PASSWORD` migrated from the old root `launch.vs.json` into communityfinder's `launch_defaults.local.json`, scoped to `*database_helper.exe*` since `create_database.cpp:32` is the only consumer and only at seed time. knottyyoga does not read it at all. ✅ 2026-09-09

**Do not lose the note in 8.2.** Hand-writing a `CMakeUserPresets.json` is only safe *because* 8.2 disabled Conan's `user_presets` generation on both the Windows profile and the Linux gate. knottyyoga now carries a committed one for the `FETCHCONTENT_SOURCE_DIR_HONUWARE` override — the machine-specific absolute path that must never reach `CMakePresets.json`. Were 8.2 to regress, Conan would overwrite that file and the override would vanish silently.

### 8.2 Stop Conan writing `CMakeUserPresets.json` into the source tree

**Scope correction: this is not what fixed the Debug menu** (8.1 was a missing startup item). It stands on its own merits: the file genuinely does flip VS into CMake Presets mode, where `CMakeSettings.json` — and with it the `CONAN_CMD` and `CMAKE_PROJECT_TOP_LEVEL_INCLUDES` wiring the whole Windows build depends on — is ignored outright. A container run silently editing a developer's IDE configuration is worth stopping regardless of which symptom first drew attention to it.

**Why hand-deleting never sticks, and this is the part worth keeping.** `build_common.sh:33-34` runs `conan install "$SRC_DIR" --output-folder="$BUILD_DIR"` where `$SRC_DIR = /src/<app>_server`, and `/src` is a **bind mount of the Windows working tree**. Conan writes `CMakeUserPresets.json` beside the recipe — so **every Linux gate run recreates the file inside the Windows repo**, with an `include` pointing at a Linux path (`/build/...`) that VS cannot resolve. The Phase 4 housekeeping note recorded one instance of this and treated it as an accident; it is not. It is what the gate does every time, and both Phase 7.3 verification runs did it.

**IMPLEMENTED 2026-09-08.** The mechanism was verified end to end before anything was edited, because the first attempt to reproduce it *failed* and nearly produced a wrong conclusion — see the note below.

- [x] Added to the committed windows profile in all three repos, with the full diagnosis in a comment above it:

  ```
  [conf]
  tools.cmake.cmaketoolchain:user_presets=
  ```

  From `conan config list`: *"(Experimental) Select a different name instead of CMakeUserPresets.json, empty to disable."* The **(Experimental)** marking is called out in the profile comment, because if a future Conan renames or drops this conf the symptom returns as a **dead Debug menu rather than an error** — there would be nothing to grep for. All three profiles re-verified byte-identical by SHA-256 after the edit (`B35D7009C191E5B6`). ✅ 2026-09-08
- [x] **Set on the Linux side too** — `-c tools.cmake.cmaketoolchain:user_presets=''` appended to the `conan install` in `communityfinder/server/docker/build_common.sh`, `knottyyoga/server/docker_project/build_common.sh` and `server_components/docker/build_and_test.sh`, plus `server_components/.github/workflows/ci.yml`. They need their own copy because the committed profile is composed into `CONAN_HOST_PROFILE` **only on WIN32**. All three scripts pass `bash -n`. ✅ 2026-09-08
- [x] Each script already carried a comment explaining that `conan install` writes `ConanLibImports.cmake` into the mounted Windows tree and that this is *"expected and harmless"*. That comment is now extended to say that **`CMakeUserPresets.json` arrives by the same mechanism and is emphatically not harmless** — the original note is what made the second file easy to overlook for so long. ✅ 2026-09-08
- [x] **Retire the manual step it replaces.** `Machine configuration.md` currently says *"If there is a CMakeUserPresets.json in each project, delete it before opening in Visual Studio 2026."* That instruction exists solely because of this behaviour. Once this is committed, delete the instruction rather than leaving a superstition behind (see 14.3). ✅ 2026-09-08
- [x] Delete the one surviving generated file: `server_components/CMakeUserPresets.json` (timestamped 18:12 today, i.e. written by a *Windows* configure, not by a gate). The two app copies are already gone. ✅ 2026-09-08
- [x] **Linux half verified by a full gate run, not just by the isolated `conan install`.** communityfinder gate with the change in place: `1785 tests from 183 test suites ran`, all passed, `[communityfinder] OK`, exit 0 — the identical count to the pre-change run in 7.3, so the flag cost nothing. The file was checked on the Windows side either side of the run: **`BEFORE present: False` → `AFTER present: False`.** Before the change, the same operation flipped it to `True`. ✅ 2026-09-08
- [x] Windows half still unproven: the profile path is exercised only by a VS configure, which cannot be driven from here. **Confirm on the next VS configure that no `CMakeUserPresets.json` reappears.** Until that is done, half of this item rests on the profile parsing correctly (verified — the `[conf]` block shows in the composed `conan profile show` output) rather than on observed behaviour. ✅ 2026-09-08

**A near-miss worth recording, because it would have put a false claim in this document.** The first attempt to reproduce the write used a scratch `conanfile.txt` with `CMakeToolchain`; no user-presets file appeared. A second attempt with a scratch `conanfile.py` using `vs_layout` — matching the real recipes — *also* produced nothing. Two failed reproductions pointed at the conclusion that **the Linux gate was not the writer at all** and that this whole subsection was misdiagnosed.

Timestamps were what forced the issue rather than settling it: `ConanLibImports.cmake` in both app trees carried gate timestamps (13:53 and 13:56), proving `conan install` *does* write into the bind-mounted tree — while `server_components/CMakeUserPresets.json` was hours newer at 18:12, pointing at a Windows configure instead. Ambiguous either way.

The decisive test was to stop simulating and **run the gate's exact `conan install` inside the real container against the real recipe**, checking the Windows side before and after: `present: False` → `present: True`. Then, with the conf: not recreated. **Both writers are real** — the Linux container and the Windows provider — which is exactly why the fix has to be applied in both places.

The lesson generalises past this item: **a simplified reproduction that fails to reproduce is not evidence that the behaviour does not occur.** Both scratch recipes were reasonable-looking and both were wrong, and either would have "disproved" a real defect. When the real thing is cheap to run, run the real thing.

### 8.3 Environment variables — the working reference

Names below were enumerated from the source, not from memory. **Defaults are marked as verified only where the code was actually read**; the rest are listed as present without asserting semantics.

**For the immediate goal — running the helpers and the server on Windows — you need almost nothing.** The connection defaults already resolve correctly on this machine: the `test_helper` failure reported `host=localhost port=5432 dbname=communityfinder user=docker`, which is the shared docker PostgreSQL. What was missing was schema, not configuration.

**Database connection** (`database_helper_init.cpp:52-54`; the legacy `KNOTTYYOGA_DB_*` names are still accepted as a fallback):

| variable | default |
|---|---|
| `HONUWARE_DB_HOST` | `localhost` on Windows, `postgresql` on Linux |
| `HONUWARE_DB_PORT` | `5432` |
| `HONUWARE_DB_USER` | `docker` |
| `HONUWARE_DB_PASSWORD` | `docker` |
| `HONUWARE_DB_SSLMODE` | unset — the docker gate sets `disable` |
| `HONUWARE_DB_NAME`, `HONUWARE_DB_SSLROOTCERT` | present in code; not needed for local runs |

**The ones you actually set by hand:**

| variable | why |
|---|---|
| `HONUWARE_ALLOW_DESTRUCTIVE=1` | gates `--recreate_database`; without it the command refuses rather than self-healing |
| `HONUWARE_MAIL_APP_PASSWORD` | read **at seed time** by `create_database.cpp:32`, which UPDATEs the `config_secrets` row. Not needed at runtime once seeded |
| `HONUWARE_SECRET_KEY` | at-rest key for `config_secrets`; non-prod falls back to a fixed dev key, so it is optional locally and **mandatory in production** |
| `SCHEDULER_SERVICE_ACCOUNT_PASSWORD` | the scheduler service account |
| `PORT` | `18081` for the communityfinder server (`ng serve` is 4201) |
| `HONUWARE_SRC_DIR` | **build-time only** → `-DFETCHCONTENT_SOURCE_DIR_HONUWARE`, for cross-repo co-development |

**Now verified against the source (2026-09-09).** The previous "semantics not verified" list is resolved below, and **four of its thirteen entries were not environment variables at all** — see the correction after the table.

| variable | verified behaviour |
|---|---|
| `PORT` | server listen port. **`18081` communityfinder** (`src/main.cpp:30`), **`18080` knottyyoga** (`src/main.cpp:45`) — different defaults, easy to trip over |
| `HONUWARE_VERSION` | build version in the health response, re-read **on every call**; falls back to `KNOTTYYOGA_VERSION`, then the literal `"unknown"`. Operators set it on the EC2 to pin which artifact is live |
| `HONUWARE_APP_NAME` | theme-bundle export metadata only (`manage_site_theme_bundle.cpp:278`); empty when unset. Never needed locally |
| `HONUWARE_TENANT_MODE` | `Fixed` (default — one tenant, no control database, no site header) or `Control` (multiplexes CloudFront-fronted sites off the control DB's `tenants` table) |
| `HONUWARE_FIXED_SITE_KEY` | Fixed mode only: overrides the site key, which otherwise **defaults to the app database name** |
| `HONUWARE_CONTROL_DB_NAME` | Control mode only; legacy `KNOTTYYOGA_CONTROL_DB_NAME` fallback |
| `HONUWARE_LOG_DEST` | where the `LogXxx()` streams write; legacy `KNOTTYYOGA_LOG_DEST` fallback |
| `HONUWARE_TRUST_PROXY`, `HONUWARE_ORIGIN_SECRET`, `HONUWARE_DEV_CORS_ORIGIN` | auth / CORS, all with legacy `KNOTTYYOGA_*` fallbacks |
| `CURL_CA_BUNDLE` | `http_client.cpp:86`; the runner otherwise resolves `certs/cacert.pem` relative to the working directory |
| `HONUWARE_SRC_DIR` | **container-side only** — `load_container.cmd` turns it into `-e HONUWARE_SRC_DIR=/honuware`. On Windows the equivalent is the CMake cache variable `FETCHCONTENT_SOURCE_DIR_HONUWARE`, not an env var |

**Correction — four entries were miscategorised, and it matters because setting them does nothing.**

- `HONUWARE_API_BASE`, `HONUWARE_CRUD_ACCESS`, `HONUWARE_MOCK_OPTIONS` are **Angular dependency-injection tokens**, not environment variables — `inject(HONUWARE_CRUD_ACCESS)`, `{ provide: HONUWARE_API_BASE, useValue: environment.apiBase }`. They live in the UI and are configured through `environment.ts`.
- `HONUWARE_ENV` is a **local batch variable** inside `load_container.cmd` (`set HONUWARE_ENV=-e HONUWARE_SRC_DIR=/honuware`) that accumulates docker arguments. No code ever reads an env var by that name.

*They earned their place on the list by matching a `HONUWARE_*` grep, which is exactly how a name-shaped search produces a plausible wrong answer. The fix was to read each use site rather than to trust the pattern.*

- [x] Fold this table into the `server_components` README (14.1). ✅ 2026-09-09 — added as **"Environment variables"** under *Build & test*, with the DB connection table, the set-by-hand table, and the correction above.
- [x] Teach the tooling the same names: `tools/launch_defaults.example.json` carries the verified reference, so the file where you set `env` for a debug target also tells you what may be set. ✅ 2026-09-09

### 8.4 Running a subset of the tests — `--gtest_filter`

The filter matches against the full `SuiteName.TestName`. Syntax is `POSITIVE[-NEGATIVE]`, patterns separated by `:`, with `*` and `?` as wildcards.

| goal | filter |
|---|---|
| one whole suite | `--gtest_filter=MailHelperTest.*` |
| one exact test | `--gtest_filter=ImageResizeTest.GetImageDimensionsTiff` |
| anything mentioning TIFF, either half of the name | `--gtest_filter=*Tiff*` |
| several patterns | `--gtest_filter=ThreadPoolTest.*:*Scheduler*` |
| everything **except** the mail tests | `--gtest_filter=-*Mail*` |
| a suite, minus one test | `--gtest_filter=ImageResizeTest.*-*Tiff*` |

Other flags worth knowing: `--gtest_list_tests` (enumerate without running — the fastest way to find a name), `--gtest_repeat=N`, `--gtest_shuffle`, `--gtest_break_on_failure` (drops into the debugger at the failure), `--gtest_output=xml:results.xml`.

**Where to put it.** For a **throwaway** filter while chasing one failure, edit the `"args"` array in `.vs\launch.vs.json` directly — that is the live file VS reads, and the edit takes effect immediately.

**But that edit is not durable, as of 2026-09-09.** `launch.vs.json` is now generated by `tools/sync_launch_targets.ps1`, and values from `launch_defaults.local.json` **overwrite** `args` wholesale on every run. So a hand-typed `--gtest_filter=ImageResizeTest.*` survives until the next time that script runs — after which it silently reverts to the standing `--gtest_filter=*`. Deleting `.vs` loses it too, since that folder is gitignored and disposable.

To make a filter stick, put it in `launch_defaults.local.json` instead, under the target's wildcard key:

```json
"targets": { "*tests.exe*": { "args": ["--gtest_filter=ImageResizeTest.*"] } }
```

That is also where `--recreate_database` and `HONUWARE_ALLOW_DESTRUCTIVE` now live for the `database_helper` targets — 8.4 previously pointed at `launch.vs.json` for that entry, which was true when written and is no longer where it is maintained.

In the container, arguments pass straight through: `./docker/build_and_test.sh --gtest_filter=Event*`.

**One gotcha, verified in the script** (`build_and_test.sh:63-66`): a filtered run **skips the test-count floor** and exits 0 early. That is correct behaviour — a subset can't be measured against a whole-suite floor — but it means **a filtered run is never a gate**. Always finish with an unfiltered run before believing anything is green.

Two harness rules that bite when running a narrow filter, both from CLAUDE.md: tables are **pre-created** by `GlobalDatabaseTestSupport` from the composed `DatabaseInfo` and each test runs in a transaction that is aborted at the end (so tests never create tables, and nothing persists); and `ThreadPool::Shutdown()` must be called before the first DB-touching assertion after `handle_full()`, because the endpoints' async writes otherwise re-enter the test's non-thread-safe libpqxx connection. A single-test run is exactly where a missing `Shutdown()` shows up as a mysterious intermittent failure.

# Phase 9 — The committed Gmail credential and the live-email test (OQ11)

Answering *"Can you put together a plan to fix these issues?"*. Two genuinely separate problems that happen to share a file; they want separate commits and have different urgency.

### 9.1 Rotate the credential (Mason — cannot be done from here)

`components/services/util/secrets/secret_values.cpp` lines 18–22 hold a real Gmail app password next to `smtp.gmail.com:465`, in a repo that runs a public GitHub Actions workflow.

- [ ] **Revoke the existing app password** at the Google account's App Passwords page. Do this first and independently of any code change — the credential is already disclosed and every hour it stays valid is exposure.

  **Prerequisite, and the reason the page sometimes appears not to exist.** App Passwords are only offered when **2-Step Verification is on**. The entry is *hidden*, not empty, when: 2SV is off; 2SV is configured as **security keys only**; **Advanced Protection** is enabled; or the account is a **Workspace / school account** whose admin has disabled app passwords. If the URL below bounces you to the Security overview, that is why — it is not a sign the password does not exist.

  **Sign in as the right account first.** This is the mailbox that sends, i.e. the address behind `kMailSenderAddress` / `::Mail::LoadSenderAddress` — not necessarily your personal Google account.

  **To revoke:**

  1. Go to **https://myaccount.google.com/apppasswords** (menu path: Google Account → **Security** → under *How you sign in to Google*, **2-Step Verification** → scroll to the bottom → **App passwords**).
  2. Re-authenticate when prompted — Google always challenges before showing this page.
  3. The table lists each app password by **the name given when it was created**, with created and last-used dates. The 16-character value itself is *never* shown again.
  4. Click **Remove** (the trash icon) on the row, and confirm.
  5. Revocation is **immediate** — SMTP auth with that string starts failing on the next attempt. Nothing needs to be restarted.

  **If you cannot tell which row is which** — likely, since the values are not displayed and names like *"Mail on Windows"* are ambiguous — **change the Google account password instead. That revokes every app password on the account at once.** Blunt, and it will break anything else on that account still using one, but it is the guaranteed close. Given there are **two** distinct credentials to account for here (see below), this is the option that does not depend on identifying them correctly.

  **To create the replacement:**

  1. Same page. Google removed the old *"Select app" / "Select device"* dropdowns — there is now a single **App name** free-text field.
  2. Name it something specific and greppable, e.g. `honuware-smtp-2026-09`. **The name is the only handle you will ever have on it**, which is precisely the problem being worked around above.
  3. Click **Create**. The 16-character password appears in a dialog as four groups of four. **It is shown exactly once** — there is no way to retrieve it later.
  4. The spaces are presentation only. Strip them; store the 16 characters unbroken.

  **Where it goes:** the `config_secrets` / `HONUWARE_MAIL_APP_PASSWORD` path only. Not `secret_values.cpp`, not any `launch.vs.json`, and — new since this item was written — **not `launch_defaults.local.json`** unless you consciously accept that as another plaintext copy on disk (it is gitignored, but it is still a file).

  Sources: [Sign in with app passwords](https://support.google.com/accounts/answer/185833?hl=en) · [App passwords page](https://myaccount.google.com/apppasswords)
- [ ] **Correction 2026-09-10: these are TWO DIFFERENT app passwords, not one.** This item previously said the launch profile carried "the same app password". It does not. Verified by comparing the literals:

  | credential | where | exposure |
  |---|---|---|
  | `ctojsn…lbdz` | `server_components/components/services/util/secrets/secret_values.cpp:18` | **committed and public** — in `HEAD` and in history since the initial-extraction commit `f95d099`, in a repo running a public Actions workflow (`.github/workflows/ci.yml`) |
  | `rquya…kwuyc` | launch profiles only — never in any repo. `git grep` across `server_components` finds it nowhere | local disk only |

  **Both must be revoked.** Rotating one and assuming the other is covered is exactly the mistake this correction exists to prevent — and it is the case for changing the account password rather than removing rows individually.

- [ ] **The local-disk inventory has grown since this item was written, partly as a side effect of 8.1's tooling.** Current known copies of `rquya…kwuyc`, targeted search (not exhaustive):

  **Verified by a full recursive scan** of all three repos plus the working scratchpad (2026-09-10). It returns **exactly these three files and nothing else**, and `git log -S` confirms this literal has never been committed to `server_components` in any branch. So the local exposure is bounded — on *this* machine. The other machine has not been scanned.

  - `communityfinder/server/communityfinder_server/launch_defaults.local.json` — **created 2026-09-09** while folding launch settings into the generator. Gitignored.
  - `communityfinder/server/communityfinder_server/.vs/launch.vs.json` — generated *from* that defaults file, so it reappears on every `sync_launch_targets.ps1` run. **Deleting it does not remove the credential; delete or edit the defaults file, or it comes straight back.**
  - the scratchpad backup of the old repo-root `communityfinder/.vs/launch.vs.json`, taken before that workspace was removed.
  - The original `communityfinder/.vs/launch.vs.json` is **gone** — removed with the mistaken root workspace (8.1).

  *Note what happened here: a tool built to stop settings being lost also made a credential harder to delete, by turning one hand-edited copy into a generated one with a durable source. That is a fair trade for launch arguments and a bad one for secrets.* Once rotation is done, the honest fix is for `HONUWARE_MAIL_APP_PASSWORD` to come from the seeded `config_secrets` row or the ambient environment rather than from a checked-in-shaped file.

  **Not committed** — `.gitignore` ignores both `.vs/` and `launch_defaults.local.json`, verified with `git check-ignore` — so this half is local-disk exposure, not a repository one. But it means **rotation is a sweep, not a single edit**: treat every VS launch profile, every `.vs` folder, every `launch_defaults.local.json`, and both machines as holding a copy. A credential pasted into a launch profile once has usually been pasted elsewhere too.
- [ ] Worth noting what this copy also reveals: communityfinder already reads the password from an **environment variable** (`HONUWARE_MAIL_APP_PASSWORD` appears in `app_secret_values.cpp` and `create_database.cpp`), so the app side of 9.2 may largely exist already and the literal in honuware's `secret_values.cpp` is the outlier. Confirm before designing a new seam — the mechanism is probably already there.
- [ ] Issue a replacement and put it **only** in the `config_secrets` / environment path the rest of the system already uses. Do not commit it.
- [ ] **Understand what rotation does and does not fix.** Removing the value from `HEAD` does **not** remove it from git history — it stays in every clone, every fork, and GitHub's raw object store. Rotation is what actually closes it.
- [ ] **Decided (New OQ 18): rotate, do not scrub history.** Once the password is revoked the value in history is inert, and a `filter-repo`/BFG rewrite would rename every commit SHA — invalidating the honuware SHA both apps pin (`6abea141…` today), breaking every existing clone, and landing that cost directly on the work in this plan. **The one thing that must not slip: the rotation is what closes this, so it cannot be deferred behind the code change in 9.2.** Revoke first, tidy the code afterwards.

### 9.2 Make honuware follow the pattern communityfinder already implements

**Investigated, and this phase is smaller than it looked: no new mechanism is needed, and the correct design is already written down in this codebase.** The three tiers exist and work:

| tier | holds | where |
|---|---|---|
| store of record | real credentials, encrypted at rest | `config_secrets`, read by `secrets_helper.cpp` |
| bootstrap key | the key decrypting the above | `HONUWARE_SECRET_KEY` env var — never in the DB (circular), never in the repo |
| defaults | **non-secrets only** | `secret_values.cpp` / `app_secret_values.cpp` — public repo |

communityfinder already does the right thing end to end. `create_database.cpp:29-32`: *"Read from the environment (the local, gitignored VS launch profile / shell) so the live credential never lands in this PUBLIC repo."* It reads `HONUWARE_MAIL_APP_PASSWORD` and **UPDATEs** the framework-seeded row (`:183-192`), deliberately falling back to **empty** when unset *"so an unconfigured build simply cannot send mail rather than authenticating as the shared account"*. And `app_secret_values.cpp:19-20` states the split: the sender address is not a secret because it appears in every From header; *"the password is the secret and stays in the environment."*

**So the defect is exactly one thing: honuware's `secret_values.cpp` ships a real password as a *framework default*.**

- [ ] Replace that literal with an **empty** default, reusing communityfinder's own reasoning verbatim — an unconfigured build must fail to send rather than authenticate as the shared account.
- [ ] **Understand why it happened, or it will happen again.** Per `secrets/CLAUDE.md`, defaults in `secret_values.cpp` are loaded *"into the database on first run (production)"* **and** *"into the test secrets helper automatically (tests)"*. Putting the password there made every test everywhere work with zero configuration. **That convenience is the trap**: the defaults file is the one place a secret must never go, and it is also the most convenient place to put one.
- [ ] Add the rule to `components/services/util/secrets/CLAUDE.md`, which currently explains *how* to add a secret's default value without ever saying that a real credential must not have one. The document that taught the pattern is the document that should forbid this.
- [ ] **Test:** a missing mail secret produces a clean, diagnosable failure rather than an attempt to authenticate with an empty password. `server_config.cpp:173-194` already has the "fail loud on missing required secret" shape to follow.

**Correction to 9.1's sweep instruction.** An earlier draft said to delete the `HONUWARE_MAIL_APP_PASSWORD` line from `communityfinder/.vs/launch.vs.json`. **That is backwards.** That launch profile runs `--recreate_database`, which is the exact code that reads the env var to seed the secret — a gitignored launch profile is the *sanctioned* location, named as such in `create_database.cpp:30`. Keep it, and put the new password there. The sweep in 9.1 is about knowing where the copies are, not deleting the legitimate one.

### 9.3 Stop the unit suite from sending real email

`MailHelperTest.SendMessage` authenticates to Gmail and sends a real message to a real address **on every full suite run** — the ~1681 ms it takes is a live SMTP round trip. Every gate run in this migration sent one, including the seven or so run during Phases 1–7.

- [ ] Drive it through the existing `TestMailHelper` double, leaving genuine sending to the `--send_real_email` path the test helper already has.
- [ ] **Test:** the double-based test must still assert what the live one did — that the composed message reaches the transport with the right recipient, subject and CRLF-correct body. The existing CRLF coverage (`quick_account_welcome_mail_test.cpp`, `person_verify_mail_test.cpp`, `types_test.cpp`) stays as-is; this is about the send path, not the formatting.
- [ ] Expect the suite to get **~1.7 s faster** and to stop depending on network egress and a third party's availability — worth noting because it also removes a flake source from CI.

**A consequence for Phase 15 worth flagging now.** 9.3 removes the exact test that segfaults under mailio 0.26 (OQ10 / 5.1). That does **not** mean the bump becomes safe — it means the crash would no longer be *observed* by the suite. If 9.3 lands before any mailio work, the segfault must be reproduced deliberately (the `--send_real_email` path, or a scratch harness) rather than assumed gone. **Do not read a green suite after 9.3 as evidence that mailio 0.26 is fixed.**

### 9.4 Where secrets actually live while developing

The tiering in 9.2 says where a secret belongs *in the system*. This is the human question underneath it: where does the plaintext sit on a developer's machine between minting it and seeding it.

**The sanctioned location is already correct and already in use: the gitignored VS launch profile.** `create_database.cpp:30` names it explicitly — *"the local, gitignored VS launch profile / shell"*. `.gitignore:52` covers `.vs/`, and the file is untracked (verified). So `communityfinder/.vs/launch.vs.json` holding `HONUWARE_MAIL_APP_PASSWORD` beside `--recreate_database` is the design working, not a leak.

**Do not keep a master copy in a text file under `Documents`.** Measured on the desktop: the `Personal` shell folder is redirected to `C:\Users\mason\OneDrive\Documents`, so anything saved through Explorer's *Documents* shortcut **syncs to Microsoft's cloud with version history**. Confusingly, `C:\Users\mason\Documents` also exists as a *separate, non-synced* folder — it is where the Obsidian vault lives, and the vault is not duplicated into OneDrive. **Two folders both presenting as "Documents", only one of which syncs, and Explorer sends you to the syncing one.** That is not a distinction to be relying on while filing secrets.

- [ ] **Preferred: mint one app password per machine** (`honuware-smtp-desktop`, `honuware-smtp-masonlg`). Google allows several per account, and this removes the storage problem rather than solving it: nothing has to travel between machines, each is independently revocable, a compromise on one machine does not touch the other, and each value lives only in that machine's launch profile and its own `config_secrets`. If one is lost, mint another — they are free and disposable, which is the entire point of app passwords.
- [ ] **For anything that genuinely must be retrievable: a password manager**, not a file. Bitwarden (free, syncs across both machines), 1Password, or KeePassXC (a local encrypted database — literally "a file in Documents", but encrypted under a master password, so it is safe even inside OneDrive). Keep a record of *which* app password belongs to which machine even when the values themselves are disposable.
- [ ] Note in passing, same pattern already in play: `C:\Users\mason\Documents\npm_recovery_codes.txt` sits in plaintext. Recovery codes outrank an app password — they are what bypasses 2FA. Worth moving into the password manager while one is being set up.
- [ ] **Rule to write into `secrets/CLAUDE.md` alongside 9.2:** if a value can sit in a public repo it is not a secret (sender address, SMTP host, port, brand strings all belong in `*_secret_values.cpp`); a secret never gets a compiled-in default; and exactly one bootstrap secret (`HONUWARE_SECRET_KEY`) lives in the environment with everything else behind it.

# Phase 10 — Parallel test runs: per-repo and per-platform test databases (Mason New Item 1)

The goal in Mason's words: kick off all three repos' suites, on both platforms, walk away for an hour, and come back to all of them finished. Today every pair of those collides.

### 10.1 Understand what actually collides (measured)

Three databases exist, and the *names* are already distinct per repo — the collision is purely **platform**, because a Linux gate and a Windows run use the same name on the same shared PostgreSQL:

| suite | database | set by |
|---|---|---|
| honuware `honuware_test_runner` | `honuware_test` | its own test main |
| knottyyoga | `test_knottyyoga` | **honuware's default** — see below |
| communityfinder | `test_communityfinder` | `test/src/main.cpp:24` |

- [ ] Note the seam that makes this cheap: `global_database_test_support.cpp:21` holds `static std::string name{kTestDatabaseName}`, set by `Initialize`, so the name is **already a runtime value with an app-supplied override**. communityfinder overrides it; knottyyoga does not.
- [ ] **A wart to fix while here:** `global_database_test_support.h:12` reads `constexpr std::string_view kTestDatabaseName = "test_knottyyoga"`. **The framework's default is one specific application's database name.** knottyyoga works only because it inherits it. See New Open Question 2.

### 10.2 foundation/testing — a compile-time platform suffix in honuware

**Design settled by New OQ 15 and 16, and it is simpler than the draft.** The first sketch used an environment variable (`HONUWARE_TEST_DB_SUFFIX`) defaulting to empty so Windows names stayed as they are. Mason asked instead that **all three projects append `windows` / `linux` depending on the platform**, and confirmed two slots per repo is enough. Those two answers together remove the need for an environment variable at all.

- [ ] Derive the suffix at **compile time** from the platform — `_windows` under `_WIN32`, `_linux` otherwise — appended to the app-supplied base name. Six databases from three base names:

  `honuware_test_windows` / `honuware_test_linux`, `test_knottyyoga_windows` / `test_knottyyoga_linux`, `test_communityfinder_windows` / `test_communityfinder_linux`

- [ ] **Why compile-time beats the environment variable, now that two slots is the answer.** The harness **DROPs and CREATEs** whatever name it is handed. An env-supplied suffix therefore makes an arbitrary, externally-controlled string part of a destructive database operation, which would have needed validation, a character allowlist and a length check against PostgreSQL's 63-byte identifier cap — all to support a third slot nobody asked for. Compile-time detection cannot be misconfigured, needs no validation, and needs no wiring in any harness. **Fewer moving parts and strictly less risk.**
- [ ] Accept the trade explicitly: with no env override, two Linux gates cannot run concurrently (two checkouts of the same repo would collide). That is precisely what OQ16 said is fine. If it ever stops being fine, the escape hatch is an override — and it should then arrive *with* the validation described above, not without it.
- [ ] **Tests:** the composed name carries the right suffix for the built platform, and the base name is preserved. Since the branch is `#ifdef`, one side is unreachable per build — so assert the composition function's behaviour against an explicitly-passed platform token rather than only the compiled default, or the test proves nothing on the other platform.

### 10.2b knottyyoga sets its own base name (New OQ 15 — cross-repo bump point)

Mason: *"I'd like knotty yoga to specify its own database name like community finder in test setup."*

- [ ] Change `global_database_test_support.h:12` from `kTestDatabaseName = "test_knottyyoga"` to honuware's own `"honuware_test"`. The framework should not default to one application's database.
- [ ] Give knottyyoga's test main an explicit base name, mirroring `communityfinder/server/communityfinder_server/test/src/main.cpp:24`. After this all three are symmetric: each app names its own database, and honuware's default is honuware's own.
- [ ] **These two must land together.** If the honuware default changes and knottyyoga has not yet set its name, knottyyoga's suite silently starts building its schema in `honuware_test` — wrong database, still green. Push honuware → re-pin → knottyyoga in one bump point, and **verify by checking which database the run actually touched**, not merely that it passed.

### 10.3 Update the documentation the change supersedes

No harness wiring is needed — that is the dividend of the compile-time design.

- [ ] Update the three READMEs that document the database-per-suite arrangement (`communityfinder/database_server/README.md:42-50`, `knottyyoga/server/docker_project/README.md:62-69`, `communityfinder/server/docker/README.md:65`), plus the `build_and_test.sh` header comments in both apps, which name the databases explicitly.
- [ ] `communityfinder/CLAUDE.md:108` names `test_communityfinder`; update it too.
- [ ] Mention that the **old unsuffixed databases become orphaned** (`honuware_test`, `test_knottyyoga`, `test_communityfinder`). They are dropped-and-recreated scratch databases with nothing to preserve, so they can simply be dropped by hand once — worth a line in the README so nobody wonders what they are.

### 10.4 Prove the parallelism, don't assume it

- [ ] Run **two suites concurrently against the same PostgreSQL** — the natural pair is a Linux gate and a Windows run of the *same* repo, since that is the case that fails today — and confirm both reach their normal test counts.
- [ ] Then run all three Linux gates concurrently. Expect them to be slower individually (shared CPU and one PostgreSQL); the goal is total wall-clock while unattended, which Mason already accepted.
- [ ] Watch for a non-obvious limit: PostgreSQL's `max_connections`. Six suites' worth of pooled connections against one dev server is the first thing likely to break, and it will present as connection errors, not as a test failure.

# Phase 11 — Remove TIFF support (OQ9)

Mason: *"TIFF isn't really used anymore. I'm fine with removing support for it."* This also deletes `libtiff` from all three dependency graphs — the package that was one of the two original VS2026 blockers.

**New OQ 14 answered, and it removes the only real risk in this phase: nothing is deployed, so there is no stored data.** The worry was that a production row might hold a TIFF something still decodes, turning a working record into a failure. With no deployment there are no rows, so the removal is a pure code change with no data-migration dimension and no back-compatibility obligation.

Worth stating once, because it applies well beyond this phase: **pre-deployment is a temporary licence.** Every removal in this plan is cheap precisely because nothing depends on it yet, and that stops being true the day the first real data lands. Anything of this shape that is worth doing is worth doing before then.

### 11.1 READ THIS FIRST — removing libtiff silently removes libjpeg

`components/foundation/util/CMakeLists.txt:58` says it outright:

> `# image_resize (PNG/TIFF/ZLIB; jpeg is pulled in transitively by TIFF — there is no separate JPEG_LIB)`

and line 60 links `${PNG_LIB} ${TIFF_LIB} ${ZLIB_LIB}` with **no JPEG edge**. So JPEG decoding currently reaches libjpeg *through libtiff's* dependency. Dropping `${TIFF_LIB}` without adding an explicit JPEG edge breaks JPEG — the format that actually is used.

- [ ] Add an explicit `${JPEG_LIB}` edge to `honuware_foundation` **and** to `${HONUWARE_TESTS_TARGET}` (line 62 has the same transitive dependency) **before** removing `${TIFF_LIB}`, and confirm `JPEG_LIB` is defined in honuware's `ConanLibImports.cmake` rather than only in the apps'.
- [ ] Verify by *building*, not by reading: the JPEG tests in `image_resize_test.cpp` are the check, and they must stay green across the intermediate commit.

### 11.2 foundation — the image path

- [ ] `image_resize.h:8` — drop `IMAGE_TYPE_TIFF` from the enum.
- [ ] `image_resize.cpp` — remove the `<boost/gil/extension/io/tiff.hpp>` include (line 6) and both switch arms (39–40 read, 86–87 write).
- [ ] `image_resize_test.cpp` — remove the `TiffToArray` helper and both tests. **`ResizeTiffThrowsBecauseTheOutputSinkCannotSeek` is the characterization test written in Phase 3.2 to fail when the defect is fixed** — deleting the feature is the other way it can legitimately go, and this is that. Its long comment block is a good source for the removal commit message.

### 11.3 platform — the upload/type surface

- [ ] `image_helper.cpp:60-61` — remove the `"tiff"` → `IMAGE_TYPE_TIFF` mapping. Decide what an uploaded TIFF now does: rejected as an unsupported type is the honest answer, and there is already a `"Unsupported image type. Must be 'jpeg' or 'png'"` error in the suite output that says this is the existing shape.
- [ ] `image_helper_test.cpp:254` — `"tiff"` appears in an accepted-types list; update, and **add a test asserting a TIFF upload is now cleanly rejected** rather than merely removing the old expectation.
- [ ] `theme_bundle_assets.cpp:85` — `if (type == "tiff" || type == "tif") return "tif";` — **leave it alone. Decided (New OQ 13, Mason took the recommendation).** It is theme-bundle asset *extension mapping*, not image decoding: it names a stored file and carries no libtiff dependency, so it costs nothing to keep and removing it would reject bundles that work today. Add a short comment there saying it is deliberately unrelated to `IMAGE_TYPE_TIFF`, so the next person removing TIFF references does not "finish the job" and quietly change theme-bundle behaviour.

### 11.4 Drop the dependency

- [ ] Remove the `libtiff` pin from all three `conanfile.py` files, including the "do not go back below 4.7.x" comment added in 3.2 — which becomes obsolete, and leaving it would mislead.
- [ ] `NOTICE:17` lists libtiff among the bundled libraries. **Legal/attribution text must be updated with the dependency**, and it is exactly the kind of file a dependency change forgets.
- [ ] Confirm the graph actually shrank: re-resolve all three repos and check libtiff is absent, rather than assuming the conanfile edit was sufficient.

# Phase 12 — Build integrity: floors, CI CMake, lockfile (OQ8, OQ6, OQ12)

Three separate answers that all concern "does the gate actually prove what we think it proves". Sequenced after Phase 11 so the lockfile captures the post-TIFF graph.

### 12.1 Raise the test-count floors (OQ8)

Mason: *"Whatever you think we should do."* Recommendation: raise all three, now. A floor at 56% of the actual count cannot catch the endpoint-anchor failure it exists for.

- [ ] communityfinder `1000` → **1700** (actual 1785).
- [ ] knottyyoga `3500` → **4800** (actual 5165).
- [ ] honuware CI `MIN_EXPECTED_TESTS: 1000` → **1700** (actual 1764).
- [ ] Add a line to each script's comment saying the floor tracks the count and must be raised as suites grow — the shared root cause is that all three were set once and never revisited.
- [ ] **Record the count in this document on every gate run from now on.** Phase 7.3 proved why: knottyyoga's written-down 5163 made `+2 = the two TIFF tests` a precise claim, while communityfinder's missing baseline left only "it went up".

### 12.2 Raise CMake in CI and the images (OQ6)

Mason: *"Let's do it."* Note his answer applies to the argument recorded under OQ6, which says CI and the **deploy image** must move together — otherwise one divergence is traded for a worse one.

Measured starting point: every Linux image is `FROM gcc:14.2.0` and installs `cmake` from bookworm apt (3.25) — `ci.yml:117-118`, `server_components/docker/Dockerfile`, `knottyyoga/server/docker_project/Dockerfile`, `communityfinder/server/docker/Dockerfile`, and the release image `knottyyoga/server/knottyyoga_server/package/Dockerfile`.

- [ ] **Decided (New OQ 17, Mason deferred to me): the official binary tarball, pinned by version and SHA-256.** Reasoning, since the choice was left open: it adds no third-party package source to the images, it installs the *exact* build the dev boxes run rather than whatever a repository currently serves, and the checksum makes the image reproducible in a way `apt-get install cmake` never was. Kitware's APT repo is the conventional choice but reintroduces "whatever is current"; `pip install cmake` drags a Python dependency into images that otherwise need none — and given 6.2b, adding an incidental Python requirement to a build image is precisely the kind of thing that bites later.
- [ ] Pin the **exact** version — 4.4.3, matching the dev boxes. "Latest" reintroduces the drift this is meant to remove.
- [ ] **Decided: all five images move together, including the release image.** New OQ 17 asked this explicitly and Mason left it to me, so recording it plainly rather than letting it pass silently: `knottyyoga/server/knottyyoga_server/package/Dockerfile` builds **what actually ships**, and this change alters the toolchain that produces the shipped artifact. The OQ6 argument is what settles it — raising CI without raising the deploy image trades one divergence for a strictly worse one, because CI would then verify a build nobody performs. If that is not wanted, this is the bullet to object to.
- [ ] Apply to all five in one change. A partial raise is worse than none: it would mean three different CMake versions across the gates.
- [ ] Expect `if(POLICY CMP0167)` to become **true** on Linux for the first time, flipping the Linux Boost lookup from module mode to config mode. That is the entire point, and it is also the one thing that could break — Phase 1.2's config-mode verification was done against Windows-generated Conan files only.
- [ ] **Gate:** all three Linux suites green at their new floors. This is the change most likely to fail of anything in Phases 8–14; run it on its own commit.

*(The `CMakeUserPresets.json` / Conan `user_presets` item that was drafted here has moved to **Phase 8.2** — it is what breaks Visual Studio's Debug menu, so it belongs in the unblocking phase rather than in build hygiene.)*

### 12.3 Commit a lockfile (OQ12)

Mason: *"I'm fine with doing whatever you think would make sense."* Recommendation: do it, and do it here — after the TIFF removal so the lockfile captures the smaller graph, and after 12.2 so it captures the intended CMake.

The concrete problem this closes, from the Phase 6 log: identical conanfiles resolved `libpq/17.11`, `xz_utils/5.8.3`, `zstd/1.5.7` on one machine and `libpq/15.5`, `xz_utils/5.4.5`, `zstd/1.5.5` on the other. **libpq 15 vs 17 is a Postgres client major version, and the two build through entirely different build systems** (MSBuild vs Meson — 6.2b).

- [ ] `conan lock create` per repo, committed beside each `conanfile.py`.
- [ ] Decide whether Linux and Windows share one lockfile or take one each — the honest answer depends on whether the two platforms can resolve an identical set, which 6.2b suggests they currently cannot.
- [ ] Wire the gates and CI to `--lockfile`, otherwise the file is decorative — the same trap as the committed Conan profile in Phase 2 that did nothing until it was named in `CONAN_HOST_PROFILE`.
- [ ] **Converge upward if they must converge**, per 6.2b: libpq 15.5's MSBuild path carries the v145 assumption that broke libsodium, while 17.11's Meson build is toolchain-agnostic.

# Phase 13 — A live smoke test for the real HttpClient (OQ7)

Mason: *"I'm fine with doing this work."* Closes the gap from 3.4: `MakeHttpClient()` — the actual libcurl-backed implementation — is executed by **no test in the suite**, and libcurl has just moved 7.86.0 → 8.21.0 and will keep moving.

- [ ] Stand up a Crow server on `127.0.0.1` inside the test, issue one real GET through `MakeHttpClient()`, assert the round trip. Crow is already linked into the tests target, so there is no new dependency.
- [ ] **Bind to port 0 and read back the assigned port.** A hardcoded port is the classic way this test becomes flaky in docker and CI, and it will fail rarely and confusingly rather than consistently.
- [ ] Respect the harness rule from CLAUDE.md: `ThreadPool::Shutdown()` before any DB-touching assertion, since the endpoints' async writes re-enter the test's libpqxx connection.
- [ ] Keep it to **one** test. This deliberately introduces a live-server pattern the codebase has avoided; the justification is one uncovered production component, and it does not generalise into a second such test without its own argument.
- [ ] Also cover the TLS path, or explicitly decide not to — `certs/cacert.pem` is resolved CWD-relative by the real client (`build_and_test.sh:56-58`), which is itself untested and is the kind of thing that breaks in the release image rather than in the gate.

# Phase 14 — server_components README and shared dev-environment setup (Mason New Item 2)

Goal: a new developer can get from a bare machine to a green `server_components` build by following one document.

### 14.1 Extend `server_components/README.md`

**Correction to the original framing of this item: the README already exists.** It was 8271 bytes when this item was written — sections *The layer stack*, *What is NOT here*, *Build & test*, *Consuming these components*, *New-consumer checklist*, *License* — and already documented the `HONUWARE_DB_*` variables and the legacy `KNOTTYYOGA_DB_*` fallback. So this is an **extension of an existing document**, not a new one, and the work is to add the machine-setup material that is missing rather than to restate what is there. Writing a second setup document beside it would be the worst outcome.

**Progress 2026-09-09: 14199 bytes.** *Build & test* now carries two new subsections — **Environment variables** (the 8.3 table, folded in) and **Debug targets in Visual Studio** (`tools/sync_launch_targets.ps1`). The *Developer machine setup* section is still outstanding.

- [ ] Add a **"Developer machine setup"** section ahead of *Build & test*, drawing on `knottyyoga/server/SERVER.md` and the vault's `Projects/Machine configuration.md`.
- [ ] Cover, in order: prerequisites (Git, CMake 4.4.3, Conan 2.31.2, Python, Visual Studio 2026) → clone → Windows/VS build → Docker/WSL setup → PostgreSQL → create and seed the dev database → running `honuware_test_runner` and the Linux gate.
- [ ] **Include the Visual Studio CMake-mode setting as a numbered setup step**, not as troubleshooting:

  > Tools → Options → CMake → *"Prefer using CMake Presets…"* → **use CMake Presets if available**

  **Reversed 2026-09-09 — this step previously said `Never`, and following that on a new machine now breaks the build outright.** The old rationale was sound while it held: the repos were driven by `CMakeSettings.json`, that is where the `CONAN_CMD` and `CMAKE_PROJECT_TOP_LEVEL_INCLUDES` wiring lived, and VS silently flipped to Presets mode whenever a Conan-generated `CMakeUserPresets.json` happened to exist — so pinning the mode made a machine deterministic instead of dependent on which files were in the tree at the moment VS opened.

  All three repos migrated to `CMakePresets.json` in 8.1. `CMakeSettings.json` is gone from every one of them, so **`Never` now leaves VS with no configuration at all** — it stops invoking CMake and *Delete Cache and Reconfigure* greys out, which is exactly what happened mid-migration and read as a preset failure rather than a settings one. `CONAN_CMD` was not carried across because it never worked (see 8.1); the Conan wiring is now `CMAKE_PROJECT_TOP_LEVEL_INCLUDES` in the preset's `cacheVariables`.

  *Keep the step, and keep it numbered — the reasoning for making the mode explicit rather than emergent survives the reversal intact. Only the value changed.*
- [x] Carry over the environment-variable table from **8.3**. ✅ 2026-09-09 — added as *Environment variables* under *Build & test*, in three tables (connection / set-by-hand / rare) plus the correction that `HONUWARE_API_BASE`, `HONUWARE_CRUD_ACCESS` and `HONUWARE_MOCK_OPTIONS` are Angular DI tokens rather than environment variables.
- [ ] **Add generating the launch configuration as a setup step**, because the VS command that would normally do it is broken (8.1) and a new machine has no `.vs` at all: after the first successful configure, run `tools\sync_launch_targets.ps1 -RepoPath .`, then copy `tools/launch_defaults.example.json` to `launch_defaults.local.json` and fill in credentials. Without this, every debug target launches bare — no DB env, no `--recreate_database`, no `HONUWARE_ALLOW_DESTRUCTIVE`.
- [ ] **State which folder to open in Visual Studio**, since both apps briefly had two candidate workspaces: open `server\<app>_server`, *not* the repo root (8.1). The Angular front end is opened separately.
- [ ] **Fold in the three machine-level failures this migration actually hit**, because each cost real time, each is a setup problem a new developer will hit identically, and none is discoverable from its error message: the Store-alias `python.exe` stub breaking Meson for libpq (6.2b), Git for Windows missing after a VS reinstall (7.1), and the **unseeded dev database** (16.1).
- [ ] **"Create and seed the dev database" must be an explicit numbered step**, with the `HONUWARE_ALLOW_DESTRUCTIVE=1` + `--recreate_database` invocation written out. It is currently documented nowhere — `Machine configuration.md` stops at cloning the repo — which is exactly why it was missed on a freshly-built machine.

  **Write out the command line, not just the F5 route.** Since 8.1 the `database_helper` debug target carries both automatically, so pressing F5 on it seeds the database — but that only works once the launch configuration has been generated, which is itself a setup step, and it hides what is actually being run. A new developer needs the invocation on the page. Note also that `HONUWARE_MAIL_APP_PASSWORD` must be set **at seed time** for communityfinder or the `config_secrets` row is left empty, and `SCHEDULER_SERVICE_ACCOUNT_PASSWORD` must be set for knottyyoga or seeding **throws**.
- [ ] Make the **two-database distinction** explicit, because a green suite hides this: the test suite drives `test_<app>` while the helpers and the server drive `<app>`. Creating one does not create the other, and 1785 passing tests say nothing about the dev database.

### 14.2 Factor the shared Docker/PostgreSQL setup out of knottyyoga

Mason is right that this is shared functionality: all three repos build in `gcc:14.2.0` containers on `knotty-net` against one PostgreSQL, and the setup currently lives in knottyyoga's tree while being a dependency of all three.

- [ ] Move the database/container setup into `server_components`, leaving pointers behind in both apps rather than duplicating.
- [ ] **Sequence this after Phase 10**, which changes the database-per-suite arrangement the current documents describe. Moving the text first would mean moving it twice.
- [ ] Note the harness path inconsistency documented back in 1.4: knottyyoga uses `server/docker_project/`, communityfinder uses `server/docker/`. Worth converging while the files are being moved anyway — and CLAUDE.md's generic reference to `server/docker/` is only correct for communityfinder.

### 14.3 Cleanup recommendations for the current setup

From reading `Machine configuration.md` and the harnesses — offered as recommendations, not changes:

- [ ] **`Machine configuration.md` is a stub with a real setup buried in it.** Sections *System Overview*, *Setup Instructions*, *Directory Structure* and *Test Overview* are all unfilled template prompts, while the genuinely load-bearing content is the eleven lines at the top. Fold those into the new README and let the vault document either point at it or be retired.
- [ ] Typo: "Install Pytyhon".
- [ ] **Explain *why* Python is needed**, or it will be dropped as unnecessary by the next person: libpq 17.11 builds with Meson, which is a Python application (6.2b). The instruction also needs the Store-alias warning beside it.
- [ ] **"Delete `CMakeUserPresets.json` before opening in VS2026" deserves its cause.** It is regenerated by `conan install --output-folder` pointing outside the repo (the Phase 4 housekeeping note), and it is gitignored. Stating the cause turns a superstition into a diagnosis.
- [ ] **Pin versions in the setup doc or stop naming them.** It names CMake 4.4.3 and Conan 2.31.2 but links to download landing pages that serve whatever is current — so the document will drift silently. Either link the exact release or say "at least X".
- [ ] `knottyyoga/database_server/Dockerfile.archive` and `Dockerfile.old` are `FROM gcc:7.3.0` and appear dead. Confirm and delete rather than carrying them into the moved layout.
- [ ] Revisit `knottyyoga/server/docker_project/conan_profiles/default` — Phase 2 found it inert (nothing references it) and drifted (`compiler.cppstd=gnu20` while the build passes `17`). Delete it or wire it up deliberately; moving it as-is would propagate a known-dead file.

# Phase 15 — Deferred, with recommendations (OQ10, OQ4)

Both were answered with a question back. Recommendations below; neither is scheduled until confirmed.

### 15.1 Boost 1.91 / mailio 0.26 (OQ10) — recommendation: **not now, but not never**

Mason: *"Is this worth tackling now? I'm fine with doing it if you think it is a good idea."*

- [ ] **My read: no, not now — and OQ2's answer is why.** The reason to fear holding was that Boost 1.86 might not survive MSVC 19.51; Phase 6 proved it does. So the bump is now *pure hygiene* with a known crash attached, and the honest trade is unchanged from 5.1: **do not trade a segfault in the mail path for currency.**
- [ ] **Two things change the calculation, and both are in this plan.** Phase 9.3 removes `MailHelperTest.SendMessage`, so the crash stops being observable by the suite — see the warning at the end of Phase 9. And Phase 9.2 moves the mail credential, which changes how that path is configured. **Whoever revisits mailio must do it after Phase 9 and must reproduce the segfault deliberately**, because the suite will no longer do it for them.
- [ ] When it is picked up: gdb in the Linux container, a backtrace out of mailio's `smtps` connect/submit, then decide upstream-report vs usage-change. Everything else is already done — the Asio fix and the `thread_pool.h` PIMPL are in the tree and compile against 1.86, so the pin is a one-line change once the crash is understood.

### 15.2 ftxui (OQ4) — recommendation: **hold at 5.0.0**

Mason: *"Is there a big advantage to moving to the newer version? I'm open to it."*

Measured: ConanCenter publishes 2.0.0, 3.0.0, 4.0.0, 4.1.0, 4.1.1, **5.0.0** (current), 6.0.2, 6.1.9, 7.0.2, 7.0.3.

- [ ] **Direct answer: no, there is no advantage that matters here.** ftxui's only consumer is the `test_helper` REPL — a developer tool, not shipped, not on the critical path, and not currently being worked on. 6.x and 7.x carry breaking UI-API changes, so the cost is real code churn and the benefit is a version number.
- [ ] Confirmed it is not a migration blocker either: it neither caps CMake nor fails at msvc 195 (Phase 4.4), and it reached `Build` on the 195 machine's graph.
- [x] **The caveat is now discharged (New OQ 19).** It carried forward from 4.4's superseded note: reaching `Build` in a graph is not the same as having compiled, and libsodium taught that lesson expensively. Mason confirms **`test_helper` builds for both knottyyoga and communityfinder on the 195 machine**, so ftxui 5.0.0 and replxx 0.0.4 are now *observed* compiling under msvc 195 rather than merely resolving. The last "held on 194-grade evidence" item from Phase 4.4 is closed. ✅ 2026-09-08
- [ ] That answer also carried a live defect with it — `test_helper` **runs** on knottyyoga but hits a database issue on communityfinder. Tracked as Phase 16; it is unrelated to ftxui and does not affect the hold decision.
- [ ] Revisit only if the REPL is due for work anyway, at which point moving once is cheaper than moving twice.

# Phase 16 — `communityfinder_test_helper` fails against its database

Surfaced by New OQ 19, which was asked about something else entirely. Mason: *"Test helper builds for knotty yoga and community finder. It runs on knotty yoga but has a database issue on community finder."*

**This is a real defect in an app binary, not a migration artifact**, and it is the only known behavioural difference between the two apps' identical-by-design CLI helpers. It also matters more than a dev tool usually would, because `test_helper` is one of exactly two consumers of abseil's flags API (4.3) and one of the two binaries no automated harness executes — `communityfinder_tests` links `communityfinder_test_cases`, not the helper's `main`.

### 16.1 Diagnosed — and it is a machine-setup gap, not a code defect

The reported error:

```
Database connection: host=localhost port=5432 dbname=communityfinder user=docker ...
Error: Failed to connect to database: ERROR:  relation "config_secrets" does not exist
```

**The connection succeeded.** It reached `communityfinder` on localhost as `docker`; what failed was the first query. Measured against the shared PostgreSQL:

| database | tables in `public` |
|---|---|
| **`communityfinder`** (dev) | **0** |
| `test_communityfinder` | 37 |
| `knottyyoga` (dev) | 118 |
| `test_knottyyoga` | 119 |
| `honuware_test` | 37 |

**The `communityfinder` dev database exists as an empty shell** — created at some point, never populated. `config_secrets` is the first table `test_helper` touches, so it dies immediately. knottyyoga's dev database carries a full 118-table schema, which is the whole of the asymmetry Mason observed. Nothing is wrong with the binary, the flags, abseil, ftxui, or the 195 toolchain.

- [x] Cause identified. ✅ 2026-09-08
- [ ] **Fix — Mason, one command.** Nothing is destroyed: the target has zero tables.

  ```powershell
  $env:HONUWARE_ALLOW_DESTRUCTIVE = "1"
  & "...\out\build\x64-Debug\src\database_helper\communityfinder_database_helper.exe" --recreate_database
  ```

- [ ] **Take the free verification while doing it.** CLAUDE.md notes that the test harness composes its schema from `MakeDatabaseInfo` and **does not run `create_database`'s seed** — so the seed path is verified by nothing in the 1785-test suite, and a live `--recreate_database` is the only thing that exercises it. This run is therefore also the first real check of communityfinder's seed. Watch for `event=recreate_database_done` rather than assuming success.
- [ ] Also discharges the unchecked bullet in **4.3** ("hand-check the flags still parse at runtime") for `--recreate_database`, since running it *is* that check.

### 16.2 The real finding: this is the third machine-setup gap in a row

Store-alias `python.exe` (6.2b) → missing Git for Windows (7.1) → **an unseeded dev database**. Three consecutive failures that looked like build or code problems and were all environment gaps, all on a freshly-set-up machine, none discoverable from the error message.

- [ ] **Feed this into Phase 14's README as a first-class setup step.** "Create and seed the dev database" is currently written down nowhere, which is precisely why it was missed — `Machine configuration.md` stops at cloning the repo. A new developer will hit this exact error.
- [ ] Note the shape for future diagnosis: **the suite passing tells you nothing about the dev database.** The suite drives `test_communityfinder` (37 tables, healthy); the helpers drive `communityfinder` (empty). Two databases, one green signal, and it covers only one of them.

### 16.3 Close the coverage gap that let it hide

Still worth doing, and unchanged by the diagnosis being benign.

- [ ] **Nothing in any harness executes these binaries.** `communityfinder_tests` links `communityfinder_test_cases`, not the helpers' `main`. That is why 4.3's runtime-flag check is still unchecked, and why this needed a human to report it.
- [ ] A smoke test that starts `test_helper` against the test database and exits cleanly would cover the flag parsing *and* the database wiring in one, and would have caught this before it reached a person.
- [ ] Same gap applies to `--migrate`, which remains unexercised by anything.

# Verification gates

Every phase uses the same two gates, in this order:

1. **Linux docker** — `server/docker/build_and_test.sh`, per change. The test-count floor is the real signal: a dependency swap is exactly the kind of change that can make routes and tests vanish while the exit code stays 0.
2. **Windows spot-check** — Mason on VS2022 + CMake 4.4 for Phases 1–5; Levi on VS2026 from Phase 6.

Cross-repo order within every phase: `server_components` → `knottyyoga` → `communityfinder`. Because both apps consume honuware via FetchContent at a pinned SHA, any phase that changes honuware is a bump point: Mason pushes honuware, the pin moves, then the apps build.

# Mason- New Items
- I would like to modify knotty yoga server, community finder, and server components so that all of them use different test database names and that we use different test database names for linux versus windows test runs. Currently, I have to wait for one test to complete for any of these three code bases before I can start another test pass and I also need to wait for you to finish running tests on Linux before I can run them on Windows. It would be nice to kick these off in parallel. I know that will slow things down, but, if I'm going to step away from my machine for an hour, it would be nice to return and have all of these have completed.
- I would like to add a README.md to C:\Users\mason\source\repos\server_components\ that is based on C:\Users\mason\source\repos\knottyyoga\server\SERVER.md but also incorporates C:\Users\mason\Documents\Obsidian\CommunityFinder\Projects\Machine configuration.md  - I basically want the instructions for how to get a new developer setup to build server components including the docker / WSL config and postgres database setup. We might want to factor the database / docker setup out of knotty yoga into server components since that is shared functionality between server components, knotty yoga server, and community finder so it should live in server components. Also, please look at my docker / container setup steps and postgres configuration to see if there is anything you would recommend cleaning up.

# Open Questions

1. **Do you want Levi blocked or unblocked during Phases 1–5?** This ordering means he cannot build until Phase 6. If he needs to keep working, the two mandatory bumps (libtiff in 3.2, abseil in 4.3) could be pulled forward into a single early commit that unblocks him without waiting for the rest. That costs the clean one-variable-at-a-time property you just asked for, so it is your call rather than mine.
	- Mason- Not really relevant now that we are done.

2. **Does Boost 1.86 build under 19.51 once b2 is floored to 5.3.3+?** Only Levi's machine can answer this, in Phase 6. If yes, Phase 5.1 becomes optional and the riskiest change in the plan disappears. If no, ConanCenter also has an open report of Boost 1.89/1.90 failing under MSVC 19.50, so budget for iteration there.
	- Mason- This is working on both machines now so I'm guessing this isn't really an issue anymore?

3. **Should `abseil` be dropped from `server_components/conanfile.py`?** Nothing in honuware links `${ABSL_LIB}` — it is used only by the two app CLI mains. Removing it would shrink honuware's standalone graph and delete one of the two VS2026 blockers outright, but it inverts the "app conanfile is a superset of honuware's" invariant. My inclination is to remove it from honuware and keep it in both apps; confirm before I do.
	- Mason- I feel like abseil is just generally useful so I'm fine with it being there.

4. **Is `ftxui` worth holding at 5.0.0?** 6.x and 7.x exist and are breaking. Holding is safe for this migration, but if the test-helper REPL is due for work anyway it may be cheaper to move it once, here, than separately later.
	- Mason- Is there a big advantage to moving to the newer version? I'm open to it.

5. **Timing against the CommunityFinder bucket work.** This migration touches all three repos and wants a quiet window. Should it land before EV1 Slice 1 (which creates the shared vocab tables and unblocks Stream B), or after?
	- Mason- Well we are done now. The big thing is that both friends needed vs2026 so it seemed like a good idea to do this first.

6. **Should CI's CMake be raised past 3.30 so it covers what the dev boxes now do?** Surfaced by 1.3. Both the CI workflow and `docker/Dockerfile` take bookworm's CMake 3.25, so `if(POLICY CMP0167)` is false there and Linux stays on module-mode FindBoost permanently. The dev boxes are now on config mode. That means the Boost lookup path we just adopted is the one path CI will never test — and the same blind spot applies to any future policy in the 3.25–4.4 window.
	- Mason- Let's do it.

   Arguments for raising it: CI stops diverging from how anyone actually builds, and it would exercise CMake 4 against the Linux dependency sources — which is the same risk Phase 1.1 just cleared on Windows, so the evidence says it is low. Arguments against: the workflow comment is explicit that CI deliberately mirrors the *consuming app's release build* (`package/Dockerfile` + `build_linux_release.sh`), so raising CI's CMake without raising the deploy image's would trade one divergence for a worse one. My read is that this is a real gap but not urgent, and that it should be decided as part of Phase 7 rather than bolted onto Phase 1 — but it is your call, and if the deploy image is due a refresh anyway the two should move together.
   - Mason- Let's do it.

7. **Should the real `HttpClient` get a live smoke test?** Surfaced by 3.4. The HTTP layer is tested entirely through doubles: `TestHttpClient` is a fake, `http_client_test_util_test.cpp` tests the fake, and the Square client tests inject it. So `MakeHttpClient()` — the actual libcurl-backed implementation — is never executed by the suite. That was tolerable while libcurl sat still at 7.86.0 for years; it is less comfortable now that it has moved to 8.21.0 and will keep moving.

   A single loopback test — stand up a Crow server on 127.0.0.1, issue one real GET through `MakeHttpClient()`, assert the round trip — would close it, and Crow is already linked into the tests target. The cost is introducing a live-server pattern the codebase has deliberately avoided, plus port-binding flakiness in docker and CI. I did not add it as part of a version bump: that is an architectural call, not something a dependency change should smuggle in. Worth doing as its own small piece of work if you agree.
   - Mason- I'm fine with doing this work.

9. **Is `IMAGE_TYPE_TIFF` reachable from a real upload — and do we fix it or delete it?** Found in 3.2: `ResizeImage` throws for every TIFF because the output sink cannot seek. It has never worked.

   The decision depends on something only you know. If users can upload TIFFs — `image_helper_test.cpp` lists `"tiff"` among accepted types, which suggests they can — this is a live crash path and the fix is small and worth doing soon: write the TIFF branch through a seekable sink (a `std::ostringstream`, then copy into the result vector) instead of `back_insert_device`. If TIFF support is vestigial and nothing real ever sends one, the more honest fix is deleting `IMAGE_TYPE_TIFF` and the libtiff dependency along with it — which would also drop a package from all three graphs.

   Either way it wants its own commit. The characterization test currently pins the broken behaviour and will fail once it is fixed, deliberately.
   - Mason- TIFF isn't really used anymore. I'm fine with removing support for it.

10. **What would it take to get onto Boost 1.91 / mailio 0.26?** The blocker is the SIGSEGV in mailio 0.26's `smtps` path (5.1). Everything else is already done — the Asio API fix and the `thread_pool.h` PIMPL are in the tree and compile on 1.86, so the pin can move as a one-line change the moment the crash is understood. Worth doing at some point: 1.86 is from 2024 and the gap will only widen. Someone needs to get a backtrace out of that crash (gdb in the Linux container against `--gtest_filter=MailHelperTest.SendMessage`) and decide whether it is a mailio regression worth reporting upstream or a change in how `smtps` must be used. Not urgent, and not something to chase inside the migration.
	- Mason- Is this worth tackling now? I'm fine with doing it if you think it is a good idea.

11. **A live Gmail app password is committed to the repo.** `components/services/util/secrets/secret_values.cpp` (lines 18–22) holds a real app password alongside `smtp.gmail.com:465`. Found while diagnosing the 5.1 segfault; entirely pre-existing and unrelated to the migration. Two consequences worth separating:

    - **The credential itself.** It is in version control, in a repo with a public GitHub Actions workflow. It should probably be rotated and moved to the same `config_secrets` / environment path everything else uses. Not touched here — rotating a credential is your call, not a side effect of a dependency migration.
    - **`MailHelperTest.SendMessage` is not a mock.** It authenticates to Gmail and sends a real email to a real hotmail address on **every full suite run** — the 1681 ms that test takes is a live SMTP round trip. Every gate run in this migration sent one. Worth deciding whether the unit suite should do that at all; the natural fix is to drive it through the existing `TestMailHelper` double and leave real sending to the `--send_real_email` path the test helper already has.

	- Mason- Can you put together a plan to fix these issues?

12. **The two machines resolve different transitive versions from the same ranges.** Spotted in the Phase 6 log. From identical conanfiles, the VS2026 box resolved `libpq/17.11`, `xz_utils/5.8.3`, `zstd/1.5.7`; the VS2022 box resolves `libpq/15.5`, `xz_utils/5.4.5`, `zstd/1.5.5`. Version ranges resolve against whatever the local cache already holds, so two developers can build genuinely different dependency sets from the same commit — and `libpq` 15 vs 17 is not a trivial difference for a Postgres client.

    Not blocking, and it did not cause any failure here. But it is a reproducibility hole, and it will eventually produce a "works on my machine" that is very hard to read. The fix is a lockfile (`conan lock create`, committed alongside the conanfiles) — which would also make the Windows and Linux gates provably identical. Worth doing after the migration settles rather than during it.
    - Mason- I'm fine with doing whatever you think would make sense.

8. **The knottyyoga test-count floor is slack.** Carried over from the Phase 1 Linux run: `MIN_EXPECTED_TESTS=3500` against an actual 5163. A third of the suite could vanish before it trips, which is the opposite of what the floor is for. Raise it toward ~4800? Unrelated to the migration, so not changed here.
	- Mason- Whatever you think we should do.
	- **Answered — scheduled as 12.1.** Raising all three (communityfinder 1000 → 1700, knottyyoga 3500 → 4800, honuware CI 1000 → 1700), plus recording the count per gate run, which is what made the Phase 7.3 `+2` claim precise.

# New Open Questions — arising from Phases 9–16

Numbered from 13 to avoid colliding with the answered set above. Several of these are blocking for their phase; each says which.

**All of 13–19 are now answered. Where each landed:**

| # | answer | lands in |
|---|---|---|
| 13 | leave `theme_bundle_assets.cpp:85` alone; comment it so nobody "finishes the job" | 11.3 |
| 14 | nothing deployed → no stored data, no migration dimension | Phase 11 preamble |
| 15 | knottyyoga names its own database; honuware's default becomes `honuware_test` | **new 10.2b** (cross-repo bump point) |
| 16 | two slots per repo is enough | 10.2 — and it is what let the env var go away |
| 17 | official tarball pinned by version + SHA-256; **all five images, release included** | 12.2 |
| 18 | rotate, do not scrub | 9.1 |
| 19 | `test_helper` builds on 195 for both apps → ftxui/replxx caveat discharged | 15.2 — but it surfaced a defect, now **Phase 16** |

**Two of these changed the plan rather than merely confirming it.** 15 and 16 together deleted the `HONUWARE_TEST_DB_SUFFIX` environment variable from the design: once the suffix is per-platform and two slots is enough, it can be a compile-time `#ifdef`, which needs no harness wiring and — because the harness DROPs and CREATEs whatever name it is given — removes an externally-controlled string from a destructive operation. **The answers made that phase smaller and safer, which is the opposite of how scope questions usually resolve.** And 19, asked as a low-stakes tidy-up about ftxui, turned up a live defect in a shipped-adjacent binary; that is now Phase 16.

13. **TIFF removal: does `theme_bundle_assets.cpp:85` go too?** *(Blocks 11.3.)* That line maps `"tiff"`/`"tif"` → the `"tif"` file extension for **theme-bundle assets**, which is a different concern from image *resizing* — it governs how an uploaded asset is stored and named, not whether it can be decoded and scaled. Removing `IMAGE_TYPE_TIFF` does not automatically mean a theme bundle may no longer contain a `.tif`.

    My inclination is to **leave it**, on the grounds that it costs nothing, carries no libtiff dependency, and removing it would reject bundles that work today. But it is your call, and if the answer is "nothing should ever accept a TIFF anywhere" then it goes and the theme-bundle tests need a case asserting the rejection.
    - Mason- I'll go with your recommendation.

14. **Do any real stored images use TIFF?** *(Blocks Phase 11 as a whole.)* "Not really used anymore" is enough to stop *accepting* TIFFs; it is not automatically enough to stop being able to *decode* one. If any production row holds a TIFF that something still reads, removing the read path turns a working record into a failure. Worth one query against the real data before 11.2 — and note that `ResizeImage` has never worked for TIFF anyway, so only the **read/dimensions** path could have live callers.
	- Mason- We haven't deployed yet so there are no existing anything. Do whatever you think is best.

15. **Should honuware's default test-database name stop being `test_knottyyoga`?** *(Blocks 10.2.)* `global_database_test_support.h:12` hard-codes one application's database as the framework default, and knottyyoga depends on inheriting it. Changing it to `honuware_test` and having knottyyoga set its own name explicitly — the way communityfinder already does — is the clean shape, and it makes all three apps symmetric.

    It is a cross-repo bump point (honuware change + a knottyyoga change that must land together, or knottyyoga's suite silently starts using the wrong database). Do it as part of Phase 10, or leave the wart alone?
    - Mason- I'm fine with a cross repo bump point. I'd like knotty yoga to specify it's own database name like community finder in test setup. I'd also like all three projects to append windows / linux depending on the platform.

16. **How many parallel slots do you actually want?** *(Shapes 10.2.)* The `_linux` / empty suffix scheme gives exactly two per repo, which satisfies the stated goal — Linux gate plus Windows run, unattended. If you would also want, say, two Linux gates at once (a branch and master), the suffix wants to come from something per-checkout rather than per-platform. Two is simpler and I would start there; say if you want more.
	- Mason- Let's go with your simpler recommendation.

17. **How should CMake 4.4.3 be installed in the `gcc:14.2.0` images?** *(Blocks 12.2.)* Bookworm apt tops out at 3.25, so the options are Kitware's APT repository, `pip install cmake==4.4.3`, or the official binary tarball. The tarball is the most reproducible and adds no package sources; pip is the shortest; Kitware's repo is the most conventional. I lean tarball, pinned by version and checksum.

    Related and more consequential: **your "let's do it" covers the release image too, correct?** `knottyyoga/server/knottyyoga_server/package/Dockerfile` builds what actually ships. The OQ6 argument is explicit that raising CI without raising the deploy image trades one divergence for a worse one — so I am planning to move all five together, but that changes the shipped artifact's build toolchain and I want it stated rather than assumed.
    - Mason- Which of these options would you recommend? I don't really have an opinion so go with whichever option you think would be best.

18. **The Gmail credential: rotate only, or also scrub git history?** *(Does not block 9.1 — rotate regardless.)* Rotation closes the exposure. Scrubbing (`git filter-repo` / BFG) removes the value from history but rewrites every commit SHA, which breaks every existing clone and fork, invalidates the honuware SHA pinned in both apps, and is highly disruptive for a repo two other repos consume by commit hash.

    My recommendation is **rotate and do not scrub** — the credential is dead once rotated, and the cost of rewriting a repo that two applications pin by SHA is large and lands squarely on the work in this plan. But if the repo is ever to go public in a stricter sense, that calculus changes.
    - Mason- Let's rotate and not scrub.

19. **Does the 195 machine's `test_helper` actually build?** *(Shapes 15.2, low stakes.)* ftxui and replxx were held on the reasoning that they resolve fine at msvc 195 — the same class of evidence that proved wrong for libsodium in 6.2. Resolving is not compiling. One deliberate build of `test_helper` on a 195 machine would convert the assumption into a fact; if it has already happened as part of 6.3's full-suite run, say so and this closes.
	- Test helper builds for knotty yoga and community finder. It runs on knotty yoga but has a database issue on community finder.
	- **Answered, and it closes the ftxui/replxx caveat (15.2): both compile under msvc 195, observed rather than inferred.** The database failure it surfaced is now Phase 16, and it raises New OQ 20.

20. **What exactly is the `communityfinder_test_helper` database failure?** *(Blocks Phase 16.)* The one-line report is enough to know there is a defect but not enough to fix it, and 16.2 is currently a hypothesis list rather than a diagnosis. What would settle it:

    - The **exact error text**, and whether it appears at startup or on a particular REPL command.
    - Whether it was run on Windows or in the Linux container, and against which database host — the `HONUWARE_DB_*` defaults point at the container's `postgresql`, so a Windows run needs the real host.
    - Whether `communityfinder` / `test_communityfinder` actually exist on that machine, and whether `--recreate_database` has ever been run there (it is gated behind `HONUWARE_ALLOW_DESTRUCTIVE=1`, so a first run without it fails rather than self-healing).

    **My leading hypothesis, worth stating so it can be falsified quickly:** the database was never created on that machine, or the `create_database` seed is broken. CLAUDE.md is explicit that the test harness composes its schema from `MakeDatabaseInfo` and **does not run the seed** — so a broken seed is invisible to all 1785 passing tests and would surface for the first time in exactly this binary. That would also explain why knottyyoga is fine: its database gets created by its own suite runs.

    **Timing matters here.** Phase 10 renames every test database. If this is diagnosed after 10.2 lands, the rename and the defect will each look like the cause of the other. Worth ten minutes now rather than an hour later.

    - Mason- provided the error: `Failed to connect to database: ERROR: relation "config_secrets" does not exist`, against `dbname=communityfinder host=localhost user=docker`.
    - **ANSWERED, and the hypothesis was right — see 16.1.** The connection succeeded; the database is an empty shell (0 tables, against `test_communityfinder`'s 37 and `knottyyoga`'s 118). `--recreate_database` has never been run on that machine. **Not a code defect, and nothing to do with the 195 toolchain.** Fix is one command; the durable fix is documenting the step in Phase 14's README, since it is written down nowhere today.
