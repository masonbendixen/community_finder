---
fileClass: Project
Category: Documentation
Status: Active
Authors: Mason Bendixen
Last Updated: 9/12/2026
Version: 1.0
tags:
---
# Setting up machine

**This document has been superseded.** The full machine setup now lives in the
repository, next to the thing it sets up:

> `server_components/README.md` → **Developer machine setup**

Written down there instead of here (migration plan Phase 14.1) because a setup
document that lives in a personal vault cannot be reviewed with the change that
invalidates it — which is exactly how this one drifted. It is also where a new
developer can find it.

That section covers, in order: prerequisites → clone → Visual Studio CMake-mode
setting and which folder to open → first configure and build → generating the
debug launch configuration → Docker, `knotty-net` and PostgreSQL → creating and
seeding the dev database → which database is which → running the suites on both
platforms. It ends with **Known setup failures**, the three machine-level problems
this migration actually hit.

# What was here, and what happened to it

The eleven lines below were the entire real content. All of it is now in the
README, corrected:

| Was | Now |
|---|---|
| Install Git — <https://git-scm.com/> | Kept, plus: **re-check `git --version` after a Visual Studio reinstall.** A reinstall can leave a `git` that works inside the IDE but not in a developer prompt, and CMake's FetchContent needs it on `PATH`. This cost real time (Phase 7.1). |
| Installing CMake 4.4.3 — <https://cmake.org/download/> | Kept, with the pin called out as deliberate. The link serves "whatever is current", so naming a version beside it drifts silently; the README says to install the exact version and to update the document if that changes. |
| Install Conan 2.31.2 — <https://conan.io/downloads> | Same treatment. 2.x is required — the 1.x index is missing recipes used here. |
| Install **Pytyhon** | Typo fixed, and — more importantly — **the reason is now written down.** Nothing in these repos is Python. It is required because `libpq` builds with Meson, which is a Python application, so a missing or broken Python breaks a C++ dependency with an error that never mentions Python (Phase 6.2b). Beside it: install machine-scoped via `winget` from an elevated prompt, **not** from the Microsoft Store, whose "app execution alias" stub exits silently and makes Meson fail pointing nowhere near the cause. |
| `winget install Python.Python.3.13 --scope machine` | Kept verbatim. |
| Install server components — `git clone …` | Kept, with the note that the app repos pull honuware in via FetchContent at a pinned SHA, so you do not need them to build or test the framework. |

Two things that were **missing** and caused real failures are now steps in their
own right:

- **Creating and seeding the dev database.** This document stopped at cloning, so
  the step was never done on a fresh machine — and nothing tells you it was
  missed, because the test suites create their own databases and pass perfectly
  while the dev database does not exist (Phase 16.1).
- **Generating the debug launch configuration.** A fresh clone has no `.vs`
  folder, and the Visual Studio 2026 command that would normally create one is
  broken, so `tools\sync_launch_targets.ps1` exists to do it (Phase 8.1).

The old instruction *"if there is a `CMakeUserPresets.json` in each project,
delete it before opening in Visual Studio 2026"* is gone, and deliberately so. It
was a workaround for Conan regenerating that file, fixed at the source in Phase
8.2; leaving the instruction behind would have preserved a superstition after its
cause was removed.

# Related Documents

- `server_components/README.md` — machine setup, build & test, environment
  variables, debug targets.
- `server_components/database_server/README.md` — the shared PostgreSQL container
  used by all three repos, and the database inventory.
- `knottyyoga/server/SERVER.md` — knottyyoga's server-specific notes.
- *Migrating to Visual Studio 2026* — the migration plan this work came from.
