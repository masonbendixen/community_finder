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
| CMake | 3.29.9 | 4.4.3 |

Two things to note. Your default Conan profile says `compiler.version=193`, but `conan_provider.cmake` derives the real value from `MSVC_VERSION` at configure time, so your builds have actually been running at **194** — which is the one configuration ConanCenter publishes binaries for. And the *Visual Studio 18 2026* generator exists only in CMake 4.2 and later, which is why Levi's 4.4.3 matters.

## 2. What actually blocks Visual Studio 2026

**2.1 — Two recipes pin CMake below 4, and CMake 3.x cannot target VS2026.** This is the blocker, and it is almost certainly the error Levi is seeing:

- `abseil/20220623.1` → `cmake/[>=3.16 <4]`
- `libtiff/4.6.0` → `cmake/[>=3.18 <4]`

Conan honours those caps, downloads a 3.x CMake as a tool dependency, and that CMake rejects the *Visual Studio 18 2026* generator. Levi's system CMake being 4.4.3 does not help — the recipe's own tool requirement wins. No other pinned recipe in any of the three repos caps CMake; everything else uses the system one, which is fine on both machines.

**2.2 — Boost's build engine does not know toolset 14.5.** The Boost recipe writes `using msvc : 14.5 : …` into `user-config.jam` (Conan resolves v145 correctly). But `b2/5.3.2` — the version in your cache — declares `.known-versions = 14.3 14.2 14.1 14.0 …` in `msvc.jam`. There is no 14.5, so the toolset never configures. **b2 5.3.3 and newer add 14.5.** Boost asks for `b2/[>=5.2 <6]`, so this is fixed with a version floor, not a Boost upgrade — but a cached 5.3.2 satisfies that range and keeps winning until pinned.

**2.3 — Every dependency compiles from source on VS2026, and no version bump changes that.** ConanCenter publishes msvc binaries at exactly one configuration: `compiler.version=194`, `compiler.cppstd=17`. Verified against zlib, openssl, boost, abseil, gtest and libpqxx at both their oldest and newest versions — there is no 195 anywhere in the index. Upgrading recipes does not buy prebuilt binaries; it buys sources and recipes that survive a source build under MSVC 19.5x and CMake 4.4. Expect a long first build on Levi's machine and do not read it as a fault.

**2.4 — mailio pins Boost exactly, so the two move as one.** `mailio/0.25.3` requires `boost/1.86.0` — an exact pin, not a range. That, rather than a preference, is why Boost sits at 1.86. `mailio/0.26.0` pins `boost/1.91.0`. A mismatched pair fails at graph resolution with a version conflict before anything compiles.

## 3. What is *not* the problem

- **Conan 2.31.2 is fine.** It lists 195 in `settings.yml` and emits toolset `v145` and the right generator.
- **`conan_provider.cmake` is fine.** `string(SUBSTRING ${MSVC_VERSION} 0 3 …)` turns 1951 into "195" with no change needed.
- **No recipe rejects msvc 195.** At `compiler.cppstd=20`, not one recipe in any of the three graphs fails `validate()`. Nothing needs forking or patching — only moving forward. (At `cppstd=14`, or with cppstd unset, crow / mailio / libpqxx all go *invalid*; the profile in Phase 2 sets it explicitly for that reason.)

## 4. Scope: three repositories, one dependency set

All three conanfiles carry **identical version pins**, so every bump below is the same edit applied three times. The apps' lists are supersets of honuware's.

| Repo | Conanfile | Notes |
|---|---|---|
| `server_components` (honuware) | `conanfile.py` | The framework. Lists `abseil` but **never links `${ABSL_LIB}`** — see 5.2. |
| `knottyyoga` | `server/knottyyoga_server/conanfile.py` | Superset; adds `ftxui/5.0.0` + `replxx/0.0.4`. Consumes honuware via FetchContent. |
| `communityfinder` | `server/communityfinder_server/conanfile.py` | Superset; same two additions. Pins honuware at `GIT_TAG cad2942…`. |

## 5. Library → layer map (drives the work order in every phase)

| Layer | Libraries consumed |
|---|---|
| foundation | boost, crow, date, libpng, libtiff, zlib, libcurl (`util/http`) |
| data | libpqxx, boost, libcurl, crow |
| services | libsodium (secrets), boost + openssl + mailio (mail) |
| square / scheduler | (via foundation), boost |
| platform | libsodium (auth), libzip (branding / theme bundles) |
| testing / tests | crow, gtest |
| **app only** | abseil (`database_helper`, `test_helper` mains), ftxui + replxx (test-helper REPL) |

**5.2 — abseil is an app-only dependency.** Nothing in `server_components` links `${ABSL_LIB}`; it is used solely for `absl/flags` in the two app CLI mains (`absl::ParseCommandLine`, `absl::GetFlag`). That API has been stable since 2020, which drops the abseil bump from "four years of risk" to "low". It also means honuware's own conanfile carries a dependency it never uses — see Open Question 4.

# Phase 1 — Confirm the diagnosis on Levi's machine

No repo changes. The whole plan rests on 2.1 being the real failure, and that inference was made on the 2022 machine. Owner: Levi, with Mason.

### 1.1 Capture the real failure

- [ ] Levi runs a clean configure of `communityfinder` and saves the full Conan/CMake output.
- [ ] Confirm the first hard error is a CMake that does not recognise the *Visual Studio 18 2026* generator, raised while building a dependency (not the app).

### 1.2 Confirm the two capped recipes in isolation

- [ ] `conan install --requires=libtiff/4.6.0 --build=missing` — expect failure at configure.
- [ ] `conan install --requires=abseil/20220623.1 --build=missing` — expect the same.

### 1.3 Confirm the Boost/b2 toolset failure

- [ ] `conan install --requires=boost/1.86.0 --build=missing` and check which b2 is selected. A fresh cache should resolve `b2/5.5.3` and get further than a cache holding `5.3.2`.
- [ ] Record whether Boost 1.86 then builds to completion under 19.51. This decides whether Phase 5.1 is needed at all.

**Gate:** the captured log agrees with Findings 2.1–2.2. If it does not, stop and revise this plan before touching a conanfile.

# Phase 2 — Shared Conan configuration

Pure configuration; no version churn, and it is the step that fixes Boost. Lower layer first means honuware before the two apps. There is nothing unit-testable here — the gate is that both machines still produce a working build.

### 2.1 Add a committed Windows profile to `server_components`

- [ ] Create `server_components/conan/profiles/windows`:

```ini
[settings]
os=Windows
arch=x86_64
compiler=msvc
compiler.cppstd=20
compiler.runtime=dynamic

[tool_requires]
b2/[>=5.3.3 <6]
```

- [ ] Leave `compiler.version` out so it is still auto-detected per machine — 194 for Mason, 195 for Levi. That is what makes this one file serve both.
- [ ] `compiler.cppstd=20` must stay explicit: at 14, or unset, crow / mailio / libpqxx all go invalid.

### 2.2 Mirror the profile into `knottyyoga`

- [ ] Add the same profile file, or reference honuware's, whichever matches how the repo is checked out.

### 2.3 Mirror the profile into `communityfinder`

- [ ] Same again.

### 2.4 Interim only — teach Conan that 195 can consume 194 binaries

Optional, and explicitly removed in Phase 7. Microsoft guarantees binary compatibility across v140–v145 and asks only that you link with the newest toolset present, so this is safe for the interim period.

- [ ] Add `"195": "194"` to the msvc fallback map in `extensions/plugins/compatibility/compatibility.py` (it ships with only `194 → 193`).
- [ ] Distribute it with `conan config install` rather than hand-editing on each machine, so the two cannot drift.
- [ ] Note the ceiling: this will **not** help gtest or abseil. Both set `compatibility_cppstd: False`, and ConanCenter builds only at C++17 while we build at 20 — those two compile from source on every machine, today included.

**Gate:** Mason builds and runs the suite on VS2022 unchanged; Levi gets past the Boost failure.

# Phase 3 — Mandatory recipe bumps

The two recipes that genuinely cannot support VS2026. Foundation before app, per the layer map.

### 3.1 foundation — libtiff 4.6.0 → 4.7.2

- [ ] `server_components/conanfile.py`
- [ ] `knottyyoga/server/knottyyoga_server/conanfile.py`
- [ ] `communityfinder/server/communityfinder_server/conanfile.py`
- [ ] Target stays `TIFF::TIFF`; no `ConanLibImports.cmake` change and no CMakeLists edit.
- [ ] **Tests:** `components/foundation/util/image_resize_test.cpp` and `components/platform/business_logic/images/image_helper_test.cpp` already characterise this path. Read them first; if neither decodes an actual TIFF, add a case that does — this is the one bump where a decoder change would be silent.

### 3.2 app — abseil 20220623.1 → 20250814.2

- [ ] `knottyyoga` and `communityfinder` conanfiles.
- [ ] `server_components/conanfile.py` — bump for consistency, or drop the entry entirely (Open Question 4).
- [ ] Target stays `abseil::abseil`; `${ABSL_LIB}` link lines in `src/database_helper/CMakeLists.txt` and `src/test_helper/CMakeLists.txt` are unaffected.
- [ ] **Tests:** the two consumers are `main()` functions with no test coverage and no seam to test against. Verify by hand instead: `--recreate_database`, `--migrate`, and the test-helper REPL flags each still parse. Do not refactor the mains to make them testable as part of this migration.

**Gate:** docker suite green with the same test count; Mason spot-checks VS2022; **Levi gets a first successful full-graph configure**.

# Phase 4 — Low-risk bumps, lowest layer first

Same-line or ABI-stable moves. None of these should touch a line of C++; if one does, split it out and treat it on its own.

### 4.1 foundation

- [ ] zlib 1.3.1 → 1.3.2
- [ ] libpng 1.6.40 → 1.6.58
- [ ] libjpeg 9e → 9f
- [ ] date 3.0.4 → 3.0.5 — the 3.0.5 recipe adds CMake-4 policy handling 3.0.4 lacks, which matters now that one machine runs 4.4
- [ ] crowcpp-crow 1.3.2 → 1.3.3 — header-only, so its package is compiler-independent and already resolves at 195
- [ ] libcurl 7.86.0 → 8.21.0 — major-looking, stable in practice; curl kept its API across the 8 boundary
- [ ] **Tests:** `image_resize_test.cpp` (png/jpeg), `date_time_util_test.cpp` (date), `util/http/` and `square/util/square/square_client_test.cpp` (curl). The curl path is the thin one — it is covered through doubles rather than a real request. Add a direct `HttpClient` test against loopback if one can be written without network flakiness; if not, record here that curl 8 was verified by the Square client tests only.

### 4.2 services

- [ ] openssl 3.5.2 → 3.5.8 — same LTS line, patch-only. Deliberately **not** 4.0.x; that is a separate decision with its own blast radius.
- [ ] **Tests:** `components/services/util/mail/mail_helper_test.cpp` and `secrets_at_rest_test.cpp` cover the consumers.

### 4.3 platform

- [ ] libzip 1.10.1 → 1.11.4 — the theme-bundle reader, and the one dependency that parses untrusted input, so currency matters more here than elsewhere.
- [ ] **Tests:** `theme_bundle_zip_test.cpp` and `theme_bundle_round_trip_test.cpp` already cover it. Confirm a malformed-archive case exists; add one if not.

### 4.4 app only

- [ ] ftxui 5.0.0 → hold at 5.0.0 for now (6.x and 7.x are breaking UI-API changes and the REPL is not on the critical path).
- [ ] replxx 0.0.4 — already the newest published; nothing to do.
- [ ] Both were checked: neither caps CMake, and both satisfy msvc 195. Neither is a migration blocker.

**Gate:** docker green; both machines build the test runner.

# Phase 5 — Code-touching bumps, one commit each

The only changes that can force a C++ edit. Foundation/services before testing, per the layer rule.

### 5.1 foundation + services — boost 1.86.0 → 1.91.0 with mailio 0.25.3 → 0.26.0

- [ ] Must be a **single commit across all three conanfiles** — the graph refuses a mismatched pair.
- [ ] Boost is consumed by foundation, data, services and scheduler, so verify upward in that order.
- [ ] Re-read the mail composition path against 0.26's message API: the `FormatString`/`{placeholder}` HTML builders, `NormalizeCrLf(...)` wrapping, and `::Mail::LoadSenderAddress(secrets, txn)`.
- [ ] **Tests:** `mail_helper_test.cpp` is the characterisation test — read it before the bump and extend it to assert CRLF line endings on generated HTML if it does not already, since that is exactly what a mailio change would break silently.
- [ ] Fallback if 1.91 will not build under 19.51: hold boost + mailio at 1.86/0.25.3. Every other step in this plan works without them.

### 5.2 testing + tests — gtest 1.12.1 → 1.17.0

- [ ] All three conanfiles. 1.17 requires C++17; we build at 20.
- [ ] The "no fixtures, self-contained tests" convention keeps the surface area small — exposure is matcher spellings and death-test details, not test structure.
- [ ] Check the custom matchers in `components/testing/test/src/util/` first (`table_matcher`, `json_value_matcher`, `json_test_util`) — they are the code most likely to need an edit.
- [ ] **Tests:** those matchers have their own tests (`table_matcher_test.cpp`, `json_value_matcher_test.cpp`, `test_helper_test.cpp`); they are the gate for this bump.

**Gate:** full docker suite **with the test-count floor intact** — a silently vanished route or test is exactly what a dependency swap can cause.

# Phase 6 — CMake 4.4 on Mason's machine

Deliberately after Phase 4, not before. CMake 4 removes compatibility with `cmake_minimum_required` below 3.5, which is what 2022-era dependency sources declare — bumping the recipes first is what makes this safe.

### 6.1 Raise CMake

- [ ] Mason installs CMake 4.4.x to match Levi.
- [ ] Re-run a clean configure of `server_components` standalone, then both apps.

### 6.2 foundation scaffolding — CMP0167 and FindBoost

- [ ] `server_components/CMakeLists.txt` sets `cmake_policy(SET CMP0167 OLD)` to keep module-mode Boost. Under 4.4 that path is deprecated and noisy.
- [ ] Since Conan already generates `BoostConfig.cmake`, the clean landing is to drop the OLD setting and let config mode win. **Verify, do not assume** — `find_package(Boost 1.83 REQUIRED COMPONENTS filesystem)` in the standalone branch is the call to watch.
- [ ] Give this its own commit; it is independent of every version bump.

### 6.3 App scaffolding

- [ ] Apply the same CMP0167 decision in `knottyyoga` and `communityfinder` top-level CMakeLists.
- [ ] Confirm the `honuware_layering.cmake` supersets still validate under 4.4 in both apps.

**Gate:** both machines on CMake 4.4.x, both building, docker unchanged (Linux uses its own CMake and is unaffected).

# Phase 7 — Full migration to Visual Studio 2026

The end state, once both developers are on 2026 and the interim support is no longer earning its keep.

### 7.1 Move both machines to 195

- [ ] Mason installs VS2026; both developers now detect `compiler.version=195`.
- [ ] Rebuild both apps from a cold Conan cache once, to prove nothing was depending on a stale 194 artifact.

### 7.2 Remove the interim scaffolding

- [ ] Drop the `195 → 194` compatibility entry added in 2.4 — with both machines at 195 it only hides drift.
- [ ] Consider pinning `compiler.version=195` explicitly in the committed profile at this point, since auto-detection is no longer buying anything.

### 7.3 Re-pin honuware in both apps

- [ ] Mason pushes `server_components`.
- [ ] Re-pin the `FetchContent` `GIT_TAG` in `communityfinder/server/communityfinder_server/CMakeLists.txt` (currently `cad2942…`) and the equivalent in `knottyyoga`.
- [ ] Verify the pinned build from a fresh clone with no `HONUWARE_SRC_DIR` override.

# Verification gates

Every phase uses the same two gates, in this order:

1. **Linux docker** — `server/docker/build_and_test.sh`, per-change. The test-count floor is the real signal: a dependency swap is exactly the kind of change that can make routes and tests vanish while the exit code stays 0.
2. **Windows spot-check** — Mason on VS2022 (through Phase 6), Levi on VS2026 from Phase 3 onward.

Cross-repo order within every phase: `server_components` → `knottyyoga` → `communityfinder`. Because both apps consume honuware via FetchContent at a pinned SHA, any phase that changes honuware is a bump point: Mason pushes honuware, the pin moves, then the apps build.

# Open Questions

1. **What is the actual first error on Levi's machine?** Findings 2.1–2.2 are inference from a synthetic 195 profile resolved on the 2022 box. The abseil/libtiff CMake cap fits the symptom exactly, but Phase 1 exists to replace inference with a log. If the real failure is something else, this plan changes.

2. **Does Boost 1.86 build under 19.51 once b2 is floored to 5.3.3+?** If yes, Phase 5.1 becomes optional and the riskiest change in the plan disappears. If no, ConanCenter also has an open report of Boost 1.89/1.90 failing under MSVC 19.50, so budget for iteration there.

3. **How far apart can the two machines drift during the interim?** The plan assumes both developers stay on the same pins at all times and the shared profile is committed. If Levi needs to move ahead independently, say so and I will add a per-machine override layer instead.

4. **Should `abseil` be dropped from `server_components/conanfile.py`?** Nothing in honuware links `${ABSL_LIB}` — it is used only by the two app CLI mains. Removing it would shrink honuware's standalone graph and delete one of the two VS2026 blockers outright, but it breaks the "app conanfile is a superset of honuware's" invariant in the other direction. My inclination is to remove it from honuware and keep it in both apps; confirm before I do.

5. **Is `ftxui` worth holding at 5.0.0?** 6.x and 7.x exist and are breaking. Holding is safe for this migration, but if the test-helper REPL is due for work anyway it may be cheaper to move it once, here, than separately later.

6. **Timing against the CommunityFinder bucket work.** This migration touches all three repos and wants a quiet window. Should it land before EV1 Slice 1 (which creates the shared vocab tables and unblocks Stream B), or after?