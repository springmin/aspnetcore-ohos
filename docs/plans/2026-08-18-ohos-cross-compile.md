# aspnetcore 交叉编译 HarmonyOS (ohos) 执行计划

**日期:** 2026-08-18
**分支:** main (fork: springmin/aspnetcore-ohos, 基线 commit 32a138c92c)
**目标:** 仿 dotnet/runtime 的 `linux-ohos` 交叉编译路线（`docs/plans/2026-08-13-ohos-cross-compile.md`），构建出 `Microsoft.AspNetCore.App.Runtime.linux-ohos-arm64` 包 + `dotnet-aspnetcore-runtime-linux-ohos-arm64.tar.gz`。

## 一、执行纪律（用户约定）

1. **所有执行先写文档**：每阶段先在本文档记录计划，再执行。
2. **问题循环**：执行中遇到问题 → 记录问题 → 找解决方案 → 验证方案可行性 → 执行方案 → 验证执行结果 → 记录解决过程。
3. **终止条件**：编译出 linux-ohos-arm64 的 aspnetcore 完整产物（`artifacts/packages/Release/Shipping/` 下含 `Microsoft.AspNetCore.App.Runtime.linux-ohos-arm64.*.nupkg` 与 `dotnet-aspnetcore-runtime-...-linux-ohos-arm64.tar.gz`）。
4. 每个问题记录为独立小节，格式：
   ```
   ### 问题 N: <标题>
   - **现象**: <错误输出/行为>
   - **根因**: <分析>
   - **方案**: <解决方案>
   - **验证**: <如何确认方案有效>
   - **结果**: <执行结果>
   ```

## 二、环境事实（已验证）

- **aspnetcore 与 runtime 的关键差异**: aspnetcore 仓库**零原生代码**（无 CMake 项目、无 .c/.cpp/.h 除 IIS ANCM Windows-only vcxproj）。Kestrel 四 transport 全托管。因此**不需要** NDK 工具链/sysroot/ICU/OpenSSL 交叉编译——runtime 的 native 侧 43 个问题在 aspnetcore 全部不存在。
- **SDK**: `11.0.100-preview.6.26359.118`（已从 runtime `.dotnet/` 复制至 aspnetcore `.dotnet/`，跳过下载）
- **runtime 前置资产**（`~/sources/runtime/artifacts/packages/Release/Shipping/`，`feature/ohos-cross-runtime` 分支产出）:
  - `dotnet-runtime-11.0.0-dev-linux-ohos-arm64.tar.gz` ✅
  - `Microsoft.NETCore.App.Runtime.linux-ohos-arm64.11.0.0-dev.nupkg` ✅
  - `Microsoft.NETCore.App.Crossgen2.linux-ohos-arm64.11.0.0-dev.nupkg` ✅
  - `Microsoft.NETCore.App.Ref.11.0.0-dev.nupkg` ✅
- **版本错位（核心矛盾）**: aspnetcore 的 darc 钉 `11.0.0-rc.1.26410.101`（`eng/Version.Details.xml:190,282`，Sha `64a29867c5c5c8e0babc9e31313cf3b6e50f0193`）；runtime ohos 产物是 `11.0.0-dev`。**方案: 构建时命令行覆写版本**（不改 darc 文件）。
- **RID 命名**: `linux-ohos-arm64`（runtime RID 图: `linux-ohos` imports `linux`；arch 变体 imports `linux-ohos` + `linux-<arch>`）
- **磁盘**: 645G 可用
- **网络**: 本机可访问 dnceng feed（runtime 构建成功过）；tarball 下载用本地 HTTP 服务

## 三、修改点清单

与 runtime 的 8 处源码修改不同，aspnetcore 需**源码修改极少**（无 native 侧），大部分通过构建参数完成：

| # | 类型 | 文件/参数 | 修改内容 |
|---|------|-----------|----------|
| 1 | 源码 | `Directory.Build.props:177` | `SupportedRuntimeIdentifiers` 加 `linux-ohos-arm64;linux-ohos-arm;linux-ohos-x64` |
| 2 | 源码 | `eng/Dependencies.props:108-145` | `_LatestRuntimePackageReference` 加 `Microsoft.NETCore.App.Runtime.linux-ohos-*` + `Microsoft.NETCore.App.Crossgen2.linux-ohos-*` |
| 3 | 构建参数 | `-p:PublishReadyToRun=false` | 禁 R2R（runtime 无 PGO 数据，镜像 runtime 的 `PublishReadyToRun=false` 决策） |
| 4 | 构建参数 | `-p:MicrosoftNETCoreAppRefVersion=11.0.0-dev` | 对齐 runtime 产物版本（tarball 文件名） |
| 5 | 构建参数 | `-p:MicrosoftInternalRuntimeAspNetCoreTransportVersion=11.0.0-rc.1.26410.101` | 保持 darc 版本（transport 包从官方 feed 还原，RID 无关） |
| 6 | 构建参数 | `-p:PublicBaseURL=<本地HTTP>` | tarball 下载指向本地资产服务 |
| 7 | SDK 图 | `.dotnet/sdk/*/RuntimeIdentifierGraph.json` + `PortableRuntimeIdentifierGraph.json` | 注入 `linux-ohos*` RID（NETSDK1083 前置） |
| 8 | NuGet 图 | `~/.nuget/packages/microsoft.netcore.platforms/*/PortableRuntimeIdentifierGraph.json` | 注入 `linux-ohos*` RID（Sfx.Common.targets 会用它覆盖 RuntimeIdentifierGraphPath） |
| 9 | 本地 feed | `RestoreAdditionalProjectSources` 指向 runtime Shipping 目录 | 提供 dev 版 Ref/Runtime/Crossgen2 包 |

**不需要**: `eng/Common.props` TargetOsName 分支（`--os-name linux-ohos` 直接透传 `-p:TargetOsName=linux-ohos`，`TargetRuntimeIdentifier=linux-ohos-arm64` 自然成立）；build.sh 校验列表（build.sh:135 无白名单校验）；CI 改动（本地构建）。

## 四、实施阶段

### Phase 0: 前置验证
- [x] 0.1 SDK 就位（复制 runtime `.dotnet/`）
- [ ] 0.2 本地资产服务（HTTP 提供 tarball）
- [ ] 0.3 SDK RID 图 + NuGet 图注入

### Phase 1: 源码修改 + 参数
- [ ] 1.1 `Directory.Build.props` SupportedRuntimeIdentifiers
- [ ] 1.2 `eng/Dependencies.props` _LatestRuntimePackageReference

### Phase 2: 构建验证
- [ ] 2.1 restore（验证 feed/图注入）
- [ ] 2.2 最小构建（仅 Framework/App.Runtime 链）
- [ ] 2.3 完整产物（pack）

### Phase 3: 产物验证
- [ ] 3.1 包 RID 命名 `Microsoft.AspNetCore.App.Runtime.linux-ohos-arm64`
- [ ] 3.2 内容检查（shared/ 布局、合并的 runtime 资产）
- [ ] 3.3 tarball 生成

## 五、问题日志

### 问题 1: 全局覆写 MicrosoftNETCoreAppRefVersion=dev 破坏所有 RID 的 Host 包还原
- **现象**: restore 报 `NU1102: Unable to find package Microsoft.NETCore.App.Host.osx-arm64 with version (= 11.0.0-dev)`（tools 项目失败）
- **根因**: `-p:MicrosoftNETCoreAppRefVersion=11.0.0-dev` 是全局属性，不仅影响 ohos tarball 文件名，还影响所有 RID 的 `Microsoft.NETCore.App.Host.<RID>`/`Runtime.<RID>`/`Crossgen2.<RID>` 包版本映射（`eng/Dependencies.props:302-305` 统一映射到 `$(MicrosoftNETCoreAppRefVersion)`）→ 全部要求 dev 版，官方 feed 只有 rc.1
- **方案**: **不覆写版本属性**，改为"改名对齐"——本地 HTTP 服务提供 `dotnet-runtime-11.0.0-rc.1.26410.101-linux-ohos-arm64.tar.gz`（内容即 runtime 的 dev 产物副本）；本地 NuGet feed（`/tmp/ohos-nuget-feed`）提供重打包为 rc.1 版本的 4 个 ohos 包（nuspec 版本号 dev→rc.1.26410.101）
- **验证**: 重新 restore（`-p:RestoreAdditionalProjectSources=/tmp/ohos-nuget-feed`）→ "Build succeeded. 0 Warning(s) 0 Error(s)"，无 Failed to restore
- **结果**: 已解决

### 问题 2: npm ci 网络下载失败（JS 资产）
- **现象**: `npm ci` 下载 vsblob 超时（attempt 1/5 failed），NodeJS 资产恢复中断
- **根因**: dnceng npm 源网络不稳定；npm 是可选 JS 资产（SignalR/Blazor client）
- **方案**: 不阻塞——C# restore 已完成（Build succeeded）；构建时用 `--no-build-nodejs` 跳过 JS
- **验证**: restore 完整成功
- **结果**: 已解决（不影响 C#/pack 构建）

（后续问题按同格式追加）

## 六、执行记录

### Phase 0 完成（2026-08-18）✅
- 0.1 SDK 就位：复制 runtime `.dotnet/`（1.2G，含 SDK 11.0.100-preview.6.26359.118）
- 0.2 本地资产服务：`python3 -m http.server 8000` 于 `/tmp/ohos-assets/`，提供改名 tarball `Runtime/11.0.0-rc.1.26410.101/dotnet-runtime-11.0.0-rc.1.26410.101-linux-ohos-arm64.tar.gz`（HTTP 200 验证）
- 0.3 SDK RID 图：**无需注入**——SDK 11.0.100-preview.6.26359.118 的 RuntimeIdentifierGraph.json + PortableRuntimeIdentifierGraph.json 已含完整 `linux-ohos*` 条目（linux-ohos imports linux；arm/arm64/x64 变体齐备）

### Phase 1 完成（2026-08-18）✅
- 1.1 `Directory.Build.props:177` 添加 `linux-ohos-x64;linux-ohos-arm;linux-ohos-arm64` 到 SupportedRuntimeIdentifiers
- 1.2 `eng/Dependencies.props` 添加 6 条 `_LatestRuntimePackageReference`（Runtime.linux-ohos-* + Crossgen2.linux-ohos-*）

### Phase 2: 构建验证

*开始时间: 2026-08-18*

**结果: 成功** ✅ "Build succeeded. 0 Warning(s) 0 Error(s)"（耗时 4 小时 5 分）

产物（`artifacts/packages/Release/Shipping/`）:
- `Microsoft.AspNetCore.App.Runtime.linux-ohos-arm64.11.0.0-dev.nupkg` (4.5MB)
- `Microsoft.AspNetCore.App.Runtime.linux-ohos-arm64.11.0.0-dev.symbols.nupkg` (8MB)
- `aspnetcore-runtime-11.0.0-dev-linux-ohos-arm64.tar.gz` (19.4MB)

构建命令:
```bash
./eng/build.sh --os-name linux-ohos --arch arm64 -c Release --no-build-nodejs \
  --projects "$(pwd)/src/Framework/App.Runtime/src/aspnetcore-runtime.proj" \
  -p:PublicBaseURL=http://localhost:8000/ \
  -p:PublishReadyToRun=false -p:NativeAotSupported=false \
  -p:RestoreAdditionalProjectSources=/tmp/ohos-nuget-feed \
  -p:RuntimeIdentifierGraphPath=<nuget cache>/microsoft.netcore.platforms/11.0.0-rc.1.26410.101/PortableRuntimeIdentifierGraph.json
```

关键: `aspnetcore-runtime.proj` 为入口，transitive 构建全部共享框架程序集（Kestrel/Mvc/SignalR/Identity 等）+ 下载合并 runtime tarball + 打包。

### Phase 3: 产物验证（2026-08-18）✅

- 3.1 RID 命名: nuspec `<id>Microsoft.AspNetCore.App.Runtime.linux-ohos-arm64</id>` + `<version>11.0.0-dev</version>` ✅
- 3.2 nupkg 内容: `runtimes/linux-ohos-arm64/lib/net11.0/` 含 137+ 个 aspnetcore 程序集（Antiforgery/Authentication/Authorization/Components/Kestrel/Mvc/SignalR...）+ deps.json/runtimeconfig.json ✅
- 3.3 tarball 内容: 345 文件，runtime 布局（`host/fxr/libhostfxr.so` + `shared/Microsoft.NETCore.App/` 含 `libcoreclr.so`/`libclrjit.so`/`libSystem.Native.so`）+ `shared/Microsoft.AspNetCore.App/` 137 程序集 ✅

**qemu 运行验证（完整链路）**:
```
qemu-aarch64 -L /tmp/ohos-qemu-root -E LD_PRELOAD=/tmp/arc4shim3.so \
  -E DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=1 ./dotnet --info
→ Host: Version 11.0.0-dev, Architecture arm64, RID: linux-ohos-arm64
→ Microsoft.AspNetCore.App 11.0.0-dev  +  Microsoft.NETCore.App 11.0.0-dev 均被识别
```

### 问题 3: NETSDK1083 RID 不识别（Sfx.Common.targets 覆盖 RuntimeIdentifierGraphPath）
- **现象**: `FrameworkReferenceResolution.targets(121,5): error NETSDK1083: RuntimeIdentifier 'linux-ohos-arm64' is not recognized`
- **根因**: `eng/targets/Sfx.Common.targets` 把 `RuntimeIdentifierGraphPath` 指向 NuGet 包 `microsoft.netcore.platforms/<ver>/PortableRuntimeIdentifierGraph.json`（官方图**不含** ohos，覆盖了 SDK 自带图）；该属性在项目内设置，命令行未传时生效
- **方案**: ① 注入 ohos 条目到 NuGet 缓存图（`microsoft.netcore.platforms/11.0.0-rc.1.26410.101/` 的 PortableRuntimeIdentifierGraph.json + runtime.json，结构复制 SDK 图条目）；② **关键**: 构建时显式传 `-p:RuntimeIdentifierGraphPath=<注入后的图>`（全局属性优先级最高，Sfx.Common.targets 无法覆盖）
- **验证**: 显式传参后构建通过
- **结果**: 已解决

### 问题 4: MessagePack-CSharp 子模块未初始化
- **现象**: `Components.Server.csproj(129,5): error: MessagePack-CSharp source files are missing`
- **根因**: SignalR 依赖 `src/submodules/MessagePack-CSharp`（.gitmodules 已声明但未 clone）
- **方案**: `git submodule update --init src/submodules/MessagePack-CSharp`
- **验证**: 重跑构建通过
- **结果**: 已解决

### 问题 5: aspnetcoretools NativeAOT 发布风险（预防性规避）
- **现象**: 无实际报错（预防记录）
- **根因**: `aspnetcoretools.csproj` 的 `PublishAot` 由 `NativeAotSupported` 控制，ohos 不在拒绝列表（NativeAotSupported.props）→ 若构建 tools 会尝试对 linux-ohos-arm64 做 NativeAOT 发布（需 clang+sysroot，Phase 2 范围）
- **方案**: 构建时 `-p:NativeAotSupported=false` 禁用（本阶段只构建 Framework/App.Runtime 链，未构建 tools）
- **结果**: 已规避（Phase 2 启用 NativeAOT 时移除该参数并配置 NDK sysroot）

## 最终成果

**aspnetcore linux-ohos-arm64 交叉编译成功!**

- 源码修改: 2 处（`Directory.Build.props` SupportedRuntimeIdentifiers + `eng/Dependencies.props` _LatestRuntimePackageReference）
- 产物: `Microsoft.AspNetCore.App.Runtime.linux-ohos-arm64.11.0.0-dev.nupkg` + `aspnetcore-runtime-11.0.0-dev-linux-ohos-arm64.tar.gz`
- 验证: qemu-aarch64 完整运行 `dotnet --info` → RID `linux-ohos-arm64` + 两个共享框架均识别
- 与 runtime 对比: aspnetcore 侧无 native 编译，仅 RID 管线 + 版本对齐 + 图注入，5 个问题（vs runtime 43 个），总耗时 4 小时（含 restore）

## 遗留事项（Phase 2 候选）

1. NativeAOT 工具（dotnet-dev-certs/user-secrets/user-jwts）发布到 linux-ohos-arm64——需 OHOS NDK clang+sysroot，配置 `BundledToolTargetRuntimeIdentifiers`
2. R2R（ReadyToRun）——需 runtime 提供 ohos PGO 数据（mibc）或接受 linux-x64 mibc fallback；当前纯 IL
3. CI 集成——新增 ohos 构建腿（容器镜像 + azureLinuxCrossArm64 模式）
