# PROJECT KNOWLEDGE BASE

**Generated:** 2026-08-18
**Commit:** 32a138c92c
**Branch:** main

## OVERVIEW

`aspnetcore-ohos`: fork of upstream `dotnet/aspnetcore` (main, .NET 11 preview era) intended as the base for an OpenHarmony (OHOS) port. **The tree currently contains ZERO OHOS-specific code** — it is a pristine 1:1 mirror of upstream at commit `32a138c92c` (Aug 2026). No OHOS TFM/RID, no `#if OHOS`, no harmony NuGet feed, no hvigor/ohpm/DevEco build files exist yet. Verify with: `grep -ri "openharmony\|ohos" --include="*.cs" --include="*.props" --include="*.targets" .`

Stack: C# / .NET 11 (`net11.0`), Arcade build, `AspNetCore.slnx` (XML solution), CI in Azure DevOps.

## STRUCTURE

```
./
├── src/                  # ALL product code; one dir per area (42 dirs)
│   ├── <Area>/           # each: src/, test/, build.sh|cmd; some: samples/, perf/, testassets/
│   ├── Shared/           # shared internal source compiled into MANY assemblies (see src/Shared/AGENTS.md)
│   ├── Framework/        # Shared Framework definition: App.Ref, App.Runtime, AspNetCoreAnalyzers
│   └── submodules/       # git submodules (googletest, MessagePack-CSharp) — UNINITIALIZED
├── eng/                  # build infra: Arcade (eng/common, read-only), targets/, tools/, helix/, scripts/, testing/
├── docs/                 # process docs; docs/README.md is index
├── .azure/pipelines/     # REAL CI (Azure DevOps): ci.yml, ci-public.yml, ci-unofficial.yml
├── .github/workflows/    # triage/automation ONLY (labeler, locker, backport) — not build CI
├── AspNetCore.slnx       # solution (slnx format; no root .sln)
├── Directory.Build.props # project classification (IsShipping*), RID list (line ~177)
├── global.json           # SDK 11.0.100-preview.6.26359.118, NO rollForward
└── restore.sh|cmd, activate.sh|ps1, clean.sh|cmd   # dev entry points (no root build.sh)
```

## WHERE TO LOOK

| Task | Location |
|------|----------|
| HTTP abstractions / routing | `src/Http/` |
| Kestrel / IIS / HttpSys servers | `src/Servers/` (Kestrel transports: `src/Servers/Kestrel/Transport.*`) |
| MVC / Razor Pages / TagHelpers | `src/Mvc/`, `src/Razor/` |
| Blazor / Components | `src/Components/` (has its own upstream AGENTS.md) |
| SignalR / gRPC | `src/SignalR/`, `src/Grpc/` |
| Middleware (Diagnostics, CORS, StaticFiles, WebSockets…) | `src/Middleware/` |
| Security / Identity / DataProtection | `src/Security/`, `src/Identity/`, `src/DataProtection/` |
| dotnet CLI tools (dev-certs, user-secrets…) | `src/Tools/` |
| dotnet new templates | `src/ProjectTemplates/` |
| Tests for area X | `src/<Area>/test/` (never top-level `test/`) |
| Version pinning (darc-managed) | `eng/Version.Details.xml`, `eng/Versions.props`, `eng/Dependencies.props` |
| Build workarounds (must be issue-tracked) | `eng/Workarounds.props`, `eng/Workarounds.targets` |
| Test retry / quarantine config | `eng/test-configuration.json`, `eng/QuarantinedTests.*.props` |

## CONVENTIONS

- **Tests**: `src/<Area>/test/*.Tests.csproj`; framework is xunit v3 + Moq injected centrally via `eng/targets/CSharp.Common.props` — NEVER add xunit/MSTest packages to a unit-test csproj. MSTest only in Components.Testing E2E infra.
- **Per-area build**: `src/<Area>/build.sh` (add `-test`). Repo-level: `./eng/build.sh -all -pack -arch x64`.
- **Bootstrap**: `source activate.sh` before ANY `dotnet` command; `./restore.sh` installs local SDK to `.dotnet/`.
- **TFM**: `net11.0` (`$(DefaultNetCoreTargetFramework)`); `SupportedRuntimeIdentifiers` at `Directory.Build.props:~177` (no ohos RID — port must add one).
- **Style**: file-scoped namespaces, `var` everywhere, braces always, throw expressions DISALLOWED, `_camelCase` private fields, XML docs required for shipping code (`.editorconfig`, `.globalconfig`).
- **NuGet**: dnceng Azure DevOps feeds only — `nuget.org` is NOT a package source.
- **Versions**: darc dependency flow — never hand-edit `eng/Version.Details.xml`/`Versions.props`.
- **Submodules**: `git submodule update --init` required (currently uninitialized; `src/submodules/Directory.Build.props` is an empty `<Project/>` to break MSBuild inheritance).

## ANTI-PATTERNS (THIS PROJECT)

- **`eng/common/**` is Arcade-synced — do NOT edit locally; durable changes go upstream to dotnet/arcade** (per `eng/common/AGENTS.md`).
- Do NOT modify "BACKCOMPAT OVERLOAD -- DO NOT TOUCH" methods (e.g. `src/Mvc/Mvc.Core/src/ControllerBase.cs:1837,1952`, HealthChecks DI extensions) — binary compat.
- Do NOT remove overrides marked "DO NOT remove" (`TransportConnection.cs:64` Abort override = stack-overflow guard).
- Serialized formats are append-only: `OutputCacheEntryFormatter.cs:395` — never remove/reorder fields.
- Never change `global.json`, `package.json`, `package-lock.json`, `NuGet.config` unless explicitly asked; no new `InternalsVisibleTo`/`[UnsafeAccessor]` across assembly boundaries.
- Build workarounds must cite the tracking issue (see `eng/Workarounds.*` header) so they can be removed later.
- There is NO `TestTargetFramework` MSBuild property (only a C# helper in `src/Tools/NativeAot/test/`).
- Grep traps: `ohos` substrings in `NetworkToHostOrder`/`*Host*` and the `@rollup/rollup-openharmony-arm64` npm package are FALSE POSITIVES, not port markers.

## UNIQUE STYLES

- **OHOS port status**: not started in this tree. When porting, expected insertion points: add RID to `Directory.Build.props` `SupportedRuntimeIdentifiers`; `TargetOsName`/`TargetRuntimeIdentifier` branch in `eng/Common.props`; new Kestrel transport under `src/Servers/Kestrel/Transport.*`; OHOS native module under `eng/`; `#if OHOS` symbols; harmony NuGet feed in `NuGet.config`.
- **src/Shared/** = shared-source pattern: internal code compiled into multiple assemblies + `runtime/` files copied from dotnet/runtime (see `src/Shared/AGENTS.md`).

## COMMANDS

```bash
source activate.sh            # REQUIRED before dotnet commands (puts .dotnet/ on PATH)
./restore.sh                  # installs pinned SDK + runtimes (wraps eng/build.sh --all --restore --no-build)
cd src/Http && ./build.sh     # per-area build (add -test for tests)
./eng/build.sh -all -pack -arch x64   # full-tree package build
./eng/build.cmd -test -projects <abs path>.csproj   # targeted tests
npm run build                 # JS assets (SignalR TS clients, Blazor JS)
./clean.sh                    # git clean -dix, preserves .dotnet/ and .tools/
git submodule update --init   # populate src/submodules before building native code
```

## NOTES

- No root `build.sh` exists — `eng/build.sh` is the canonical entry (Windows: `eng/build.cmd`/`eng/build.ps1`).
- CI is Azure DevOps (`.azure/pipelines/ci.yml`, `ci-public.yml` for forks) — GitHub Actions workflows are triage only.
- Local dev downloads preview .NET runtime from internal feed (CI uses `dotnetbuilds-internal` token); public mirror = `ci-public.yml`.
- Existing upstream AGENTS.md files (do not overwrite): `eng/common/AGENTS.md`, `src/Components/AGENTS.md`, `src/Components/Testing/AGENTS.md`.
- Native C++ tests (googletest submodule) exist only for Windows IIS ANCM — the pattern to imitate for future OHOS native tests.
