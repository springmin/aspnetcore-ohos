# src/Shared — Shared-Source Convention

## OVERVIEW

One directory, TWO distinct sharing patterns (both compile the same files into MANY consumer assemblies):

1. **Internal shared source** — ~106 items (`.cs` files + dirs like `Buffers/`, `Hpack/`, `JSInterop/`, `Razor/`, `Metrics/`). No own assembly; consumers pull files via `<Compile Include="$(SharedSourceRoot)...">` with `LinkBase="Shared"` (property defined in `Directory.Build.props:151`; pattern in `eng/targets/CSharp.Common.props:12`, `src/Http/Http/src/*.csproj`).
2. **runtime/ mirror** — protocol infrastructure (HTTP/2, HTTP/3, HPACK) synced bidirectionally with dotnet/runtime. See `runtime/ReadMe.SharedCode.md`.

`build.sh|cmd`, `Shared.slnf`, `test/` exist only for the runtime-mirror tests (`src/Shared/test/Shared.Tests`).

## WHERE TO LOOK

| Task | Location |
|------|----------|
| Buffers / pooled memory helpers | `src/Shared/Buffers*`, `PooledArrayBufferWriter.cs`, `ValueStringBuilder` |
| HTTP/2+3 / HPACK (runtime-mirrored) | `src/Shared/runtime/` (Http2/, Http3/, Hpack) |
| Property access / activation perf helpers | `PropertyHelper`, `PropertyActivator`, `ObjectMethodExecutor`, `ClosedGenericMatcher`, `ParameterDefaultValue` |
| Roslyn/codegen helpers | `Roslyn/`, `RoslynUtils/`, `CodeAnalysis/` |
| HTTP parsing primitives | `HttpRuleParser.cs`, `UrlDecoder`, `MediaType/`, `PathNormalizer`, `QueryStringEnumerable.cs` |
| Shared test infra | `test/` (Shared.Tests), `TestResources.cs`, `SyncPoint`, `EventSource.Testing` |
| Type/reflection attributes | `TrimmingAttributes.cs`, `PlatformAttributes.cs`, `OperatingSystem.cs`, `Obsoletions.cs`, `ReferenceAssemblyInfo.cs` |

## CONVENTIONS

- **Do not create a project file here** — the point is that these files are compiled into multiple assemblies; a project here would defeat the pattern.
- Consumers reference shared files through `<Compile Include="$(SharedSourceRoot)..."/>` (optionally `LinkBase="Shared"`) — mirror how an existing consumer (e.g. Kestrel, Mvc, Http) includes the file you touch.
- **runtime/ sync rules** (per `runtime/ReadMe.SharedCode.md`):
  - Changes to `src/Shared/runtime/` MUST be checked into BOTH repos.
  - Sync scripts: `runtime/CopyToAspNetCore.sh|cmd` (needs `ASPNETCORE_REPO`) and `runtime/CopyToRuntime.sh|cmd` (needs `RUNTIME_REPO`) — use `rsync --delete`, so ADD FILES ON BOTH SIDES.
  - The `runtime-sync.yml` GitHub Action auto-creates PRs pulling dotnet/runtime → aspnetcore; the reverse (aspnetcore → runtime) is a MANUAL runtime-side action.
  - Tests for mirrored code live in `test/Shared.Tests/runtime/`.
- Keep shared files OS/transport-agnostic unless guarded — this is the area where an OHOS port must NOT fork a file without upstream coordination.

## ANTI-PATTERNS

- **NEVER** edit `runtime/` files in only one repo — the next sync (`rsync --delete`) will silently clobber or lose the change. Commit to both sides.
- **NEVER** add `InternalsVisibleTo` from a shared file — consumers compile it into their own assembly.
- **NEVER** put OHOS-specific `#if` directly in a shared file shared with upstream (especially `runtime/`) without a plan to flow it to dotnet/runtime; prefer a wrapper file in the consuming area.
- Do not add new dependencies: shared code must stay dependency-free so any consumer assembly can host it.
