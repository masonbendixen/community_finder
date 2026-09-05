---
fileClass: Project
Category: Claude
Status: Active
Authors: Mason Bendixen
Last Updated: 9/4/2026
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

- [ ] Levi pulls all three repos at the Phase 5 state and builds from a **cold Conan cache**, so no stale 194 artifact can mask a problem.
- [ ] Expect a long first build: with no 195 binaries and a C++20 profile, roughly twenty packages compile from source, Boost and OpenSSL among them.

### 6.2 Confirm the blockers are gone

- [ ] libtiff and abseil configure without a CMake 3.x tool dependency being pulled in.
- [ ] Boost configures its toolset — confirm `b2 5.3.3+` was selected and `using msvc : 14.5` was accepted.

### 6.3 Run the gates on 2026

- [ ] Full suite on Levi's machine at the same test count as the 2022 machine.
- [ ] Any delta in test count is a broken endpoint anchor, not a flake — chase it before proceeding.

**Gate:** both machines build all three repos and produce identical test counts.

# Phase 7 — Full migration to Visual Studio 2026

The end state, once both developers are on 2026 and interim support stops earning its keep.

### 7.1 Move both machines to 195

- [ ] Mason installs VS2026; both developers now detect `compiler.version=195`.
- [ ] Rebuild both apps from a cold cache once, to prove nothing depended on a 194 artifact.

### 7.2 Remove the interim scaffolding

- [ ] Drop the `195 → 194` compatibility entry from 2.4 — with both machines at 195 it only hides drift.
- [ ] Consider pinning `compiler.version=195` explicitly in the committed profile, since auto-detection is no longer buying anything.

### 7.3 Re-pin honuware in both apps

- [ ] Mason pushes `server_components`.
- [ ] Re-pin the `FetchContent` `GIT_TAG` in `communityfinder/server/communityfinder_server/CMakeLists.txt` (currently `cad2942…`) and the equivalent in `knottyyoga`.
- [ ] Verify the pinned build from a fresh clone with no `HONUWARE_SRC_DIR` override.

# Verification gates

Every phase uses the same two gates, in this order:

1. **Linux docker** — `server/docker/build_and_test.sh`, per change. The test-count floor is the real signal: a dependency swap is exactly the kind of change that can make routes and tests vanish while the exit code stays 0.
2. **Windows spot-check** — Mason on VS2022 + CMake 4.4 for Phases 1–5; Levi on VS2026 from Phase 6.

Cross-repo order within every phase: `server_components` → `knottyyoga` → `communityfinder`. Because both apps consume honuware via FetchContent at a pinned SHA, any phase that changes honuware is a bump point: Mason pushes honuware, the pin moves, then the apps build.

# Open Questions

1. **Do you want Levi blocked or unblocked during Phases 1–5?** This ordering means he cannot build until Phase 6. If he needs to keep working, the two mandatory bumps (libtiff in 3.2, abseil in 4.3) could be pulled forward into a single early commit that unblocks him without waiting for the rest. That costs the clean one-variable-at-a-time property you just asked for, so it is your call rather than mine.

2. **Does Boost 1.86 build under 19.51 once b2 is floored to 5.3.3+?** Only Levi's machine can answer this, in Phase 6. If yes, Phase 5.1 becomes optional and the riskiest change in the plan disappears. If no, ConanCenter also has an open report of Boost 1.89/1.90 failing under MSVC 19.50, so budget for iteration there.

3. **Should `abseil` be dropped from `server_components/conanfile.py`?** Nothing in honuware links `${ABSL_LIB}` — it is used only by the two app CLI mains. Removing it would shrink honuware's standalone graph and delete one of the two VS2026 blockers outright, but it inverts the "app conanfile is a superset of honuware's" invariant. My inclination is to remove it from honuware and keep it in both apps; confirm before I do.

4. **Is `ftxui` worth holding at 5.0.0?** 6.x and 7.x exist and are breaking. Holding is safe for this migration, but if the test-helper REPL is due for work anyway it may be cheaper to move it once, here, than separately later.

5. **Timing against the CommunityFinder bucket work.** This migration touches all three repos and wants a quiet window. Should it land before EV1 Slice 1 (which creates the shared vocab tables and unblocks Stream B), or after?

6. **Should CI's CMake be raised past 3.30 so it covers what the dev boxes now do?** Surfaced by 1.3. Both the CI workflow and `docker/Dockerfile` take bookworm's CMake 3.25, so `if(POLICY CMP0167)` is false there and Linux stays on module-mode FindBoost permanently. The dev boxes are now on config mode. That means the Boost lookup path we just adopted is the one path CI will never test — and the same blind spot applies to any future policy in the 3.25–4.4 window.

   Arguments for raising it: CI stops diverging from how anyone actually builds, and it would exercise CMake 4 against the Linux dependency sources — which is the same risk Phase 1.1 just cleared on Windows, so the evidence says it is low. Arguments against: the workflow comment is explicit that CI deliberately mirrors the *consuming app's release build* (`package/Dockerfile` + `build_linux_release.sh`), so raising CI's CMake without raising the deploy image's would trade one divergence for a worse one. My read is that this is a real gap but not urgent, and that it should be decided as part of Phase 7 rather than bolted onto Phase 1 — but it is your call, and if the deploy image is due a refresh anyway the two should move together.

7. **Should the real `HttpClient` get a live smoke test?** Surfaced by 3.4. The HTTP layer is tested entirely through doubles: `TestHttpClient` is a fake, `http_client_test_util_test.cpp` tests the fake, and the Square client tests inject it. So `MakeHttpClient()` — the actual libcurl-backed implementation — is never executed by the suite. That was tolerable while libcurl sat still at 7.86.0 for years; it is less comfortable now that it has moved to 8.21.0 and will keep moving.

   A single loopback test — stand up a Crow server on 127.0.0.1, issue one real GET through `MakeHttpClient()`, assert the round trip — would close it, and Crow is already linked into the tests target. The cost is introducing a live-server pattern the codebase has deliberately avoided, plus port-binding flakiness in docker and CI. I did not add it as part of a version bump: that is an architectural call, not something a dependency change should smuggle in. Worth doing as its own small piece of work if you agree.

9. **Is `IMAGE_TYPE_TIFF` reachable from a real upload — and do we fix it or delete it?** Found in 3.2: `ResizeImage` throws for every TIFF because the output sink cannot seek. It has never worked.

   The decision depends on something only you know. If users can upload TIFFs — `image_helper_test.cpp` lists `"tiff"` among accepted types, which suggests they can — this is a live crash path and the fix is small and worth doing soon: write the TIFF branch through a seekable sink (a `std::ostringstream`, then copy into the result vector) instead of `back_insert_device`. If TIFF support is vestigial and nothing real ever sends one, the more honest fix is deleting `IMAGE_TYPE_TIFF` and the libtiff dependency along with it — which would also drop a package from all three graphs.

   Either way it wants its own commit. The characterization test currently pins the broken behaviour and will fail once it is fixed, deliberately.

10. **What would it take to get onto Boost 1.91 / mailio 0.26?** The blocker is the SIGSEGV in mailio 0.26's `smtps` path (5.1). Everything else is already done — the Asio API fix and the `thread_pool.h` PIMPL are in the tree and compile on 1.86, so the pin can move as a one-line change the moment the crash is understood. Worth doing at some point: 1.86 is from 2024 and the gap will only widen. Someone needs to get a backtrace out of that crash (gdb in the Linux container against `--gtest_filter=MailHelperTest.SendMessage`) and decide whether it is a mailio regression worth reporting upstream or a change in how `smtps` must be used. Not urgent, and not something to chase inside the migration.

11. **A live Gmail app password is committed to the repo.** `components/services/util/secrets/secret_values.cpp` (lines 18–22) holds a real app password alongside `smtp.gmail.com:465`. Found while diagnosing the 5.1 segfault; entirely pre-existing and unrelated to the migration. Two consequences worth separating:

    - **The credential itself.** It is in version control, in a repo with a public GitHub Actions workflow. It should probably be rotated and moved to the same `config_secrets` / environment path everything else uses. Not touched here — rotating a credential is your call, not a side effect of a dependency migration.
    - **`MailHelperTest.SendMessage` is not a mock.** It authenticates to Gmail and sends a real email to a real hotmail address on **every full suite run** — the 1681 ms that test takes is a live SMTP round trip. Every gate run in this migration sent one. Worth deciding whether the unit suite should do that at all; the natural fix is to drive it through the existing `TestMailHelper` double and leave real sending to the `--send_real_email` path the test helper already has.

8. **The knottyyoga test-count floor is slack.** Carried over from the Phase 1 Linux run: `MIN_EXPECTED_TESTS=3500` against an actual 5163. A third of the suite could vanish before it trips, which is the opposite of what the floor is for. Raise it toward ~4800? Unrelated to the migration, so not changed here.
