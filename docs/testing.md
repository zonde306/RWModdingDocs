# 自动化测试流程

改完一个 mod 后，先自动检查 XML、构建和纯逻辑，再用独立游戏进程验证实际行为；涉及存档时，让第二个进程读取第一个进程生成的检查点。每轮保留命令、逐项断言和日志，失败后能定位到具体阶段，而不是只得到“游戏启动成功”。

本流程面向 RimWorld 1.7 / Waylith / .NET 10。通用流程是工程建议；可直接复用的本地实例是 RimLayers 的测试入口。2026-10-21 读取本机 `Version.txt` 为 `1.7.4984 rev1242`；本文核对了脚本和测试源码，未重新运行游戏。历史运行证据保留其原有版本，见 [sources.md](sources.md)。

## 1. 按改动选择最短测试链

| 本次改动 | 自动化入口 | 还要验证什么 |
|---|---|---|
| XML / Def / Patch | XML 解析、明确 XPath 命中数 | 真正加载后的目标值、交叉引用、非目标 Def |
| 纯算法、坐标、配额、数值转换 | 无游戏依赖的 .NET 测试 | 调用方是否传入正确的游戏状态 |
| Comp、Job、Harmony、地图状态 | 构建 → 游戏内场景断言 | 正常路径、相关失败路径、未受影响的原版行为 |
| Scribe、新字段、实体引用 | 新档场景 → 保存 → 退出 → 新进程读档 | 数值、实体身份、引用、缓存重建；需要时测旧档副本 |
| 输入 / UI / 渲染 | 实际控件输入链、游戏状态断言、截图 | 人工或图像工具检查实际画面、遮挡、文字和交互反馈 |
| 性能 | 固定场景的前后对照 | 同输入、同加载组合，分别记录加载耗时、tick 进度与渲染耗时 |

从 [debugging.md 的测试矩阵](debugging.md#依改动选择测试) 选与改动有关的场景；不默认重跑全部组合。编译通过、纯逻辑通过、游戏内通过分别记录。

## 2. 固定输入与输出位置

先读目标 mod 的规则、About、LoadFolders 和项目文件，确认真实 packageId、有效内容目录和构建输出。已有测试脚本优先沿用；没有时，可采用以下布局：

```text
<ModRoot>/
  Tests/
    <Feature>Tests/       纯逻辑测试，bin/obj 留在测试工程内
    Run-Integration.ps1  构建、准备隔离配置、启动、等待、判定
    Artifacts/<runId>/   每轮新目录：配置、存档、日志、结果、截图
  Source/.../Diagnostics/  显式开启的游戏内测试驱动
```

每轮先确定：

- **输入**：游戏版本、目标 mod 版本或源码修订、场景名、随机种子、地图大小、Pawn 数、DLC/mod 顺序；用到的检查点来源。
- **断言**：操作前状态、要触发的入口、预期终态、必须经过的中间事件，以及一个能区分错误实现的反例。
- **预算**：游戏 tick 上限与墙钟超时分别设置；暂停、失焦、加载不算作模拟进度。
- **产物**：独立 runId、各阶段结果路径、日志、必要截图和存档。失败轮次原样保留，新一轮使用新目录。

测试配置只能写入本轮目录。不要修改玩家的 ModsConfig、Prefs 或存档；不要把测试 DLL、测试框架、游戏 DLL、运行时 DLL、另一份 Harmony 输出到 mod 的有效 `Assemblies` 目录。

## 3. 先跑不启动游戏的检查

### XML 与构建

下列 PowerShell 片段在游戏根目录执行。先替换目标路径；纯 XML mod 省略 C# 构建段。XML 检查失败立即停止，不继续启动游戏。

```powershell
$ErrorActionPreference = 'Stop'
$targetMod = 'Mods/目标Mod'
$project = Join-Path $targetMod 'Source/目标项目/目标项目.csproj'
if (-not (Test-Path -LiteralPath $targetMod -PathType Container)) {
    throw "Mod directory not found: $targetMod"
}
$xmlFiles = @(Get-ChildItem -LiteralPath $targetMod -Filter '*.xml' -Recurse -File |
    Where-Object { $_.FullName -notmatch '[\\/](bin|obj|Tests)[\\/]' })
if ($xmlFiles.Count -eq 0) { throw 'No mod XML files found' }
foreach ($file in $xmlFiles) {
    $doc = New-Object System.Xml.XmlDocument
    $doc.Load($file.FullName)
}
& dotnet build $project -c Release
if ($LASTEXITCODE -ne 0) { throw "Build failed: $LASTEXITCODE" }
```

这里排除了测试产物和构建中间目录。多版本目录中的 XML 可以做语法检查，但 1.7 实际行为应按 LoadFolders 选中的内容验收。XML 解析不检查 C# 字段和 Def 引用；XPath 检查示例、继承前 DOM 与跨 mod 补丁限制见 [debugging.md](debugging.md#静态-xml-检查) 和 [xml-patches.md](xml-patches.md)。

构建后检查有效输出目录，只应包含应发布的自身程序集和资源。引用与 `Private=false` 设置见 [environment.md](environment.md#构建与输出检查)。PowerShell 的 `$ErrorActionPreference` 不能替代每条外部构建/测试命令后的退出码检查。

### 纯逻辑测试运行生产代码

适合独立测试的部分应接收普通数据，避免依赖 `Find.CurrentMap`、Def 数据库、资源加载或引擎静态初始化。通过项目引用或链接源码测试**实际生产实现**，不要在测试中复制一份算法。

本地 [JumpTests.csproj](../Mods/RimLayers/Tests/JumpTests/JumpTests.csproj) 是 `net10.0` 控制台工程，链接生产 `ShotGeometry.cs` 和 `JumpGeometry.cs`；[Program.cs](../Mods/RimLayers/Tests/JumpTests/Program.cs) 逐项断言，失败抛异常，全部通过输出断言数。它没有测试适配器，应使用 `dotnet run`：

```powershell
& dotnet run --project 'Mods/RimLayers/Tests/JumpTests/JumpTests.csproj' -c Release
if ($LASTEXITCODE -ne 0) { throw 'Jump geometry tests failed' }
```

已有 xUnit / NUnit / MSTest 工程继续用其既有 `dotnet test` 入口和报告配置，不为统一命令另换框架。确认确实执行了预期测试，零测试不能算通过。

按业务选边界，例如：空输入、上下界、正反方向、目标消失，以及相同时间跨度拆成不同 `delta` 后应保持的结果。不要为测试方便伪造整套 Map/Pawn，也不要用直接调用回调证明真实 tick 调度正确。

## 4. 在独立游戏进程中执行场景

推荐外层脚本负责进程和产物，游戏内驱动负责准备场景、触发行为与断言。隐藏窗口仍是真实引擎进程，不等于 headless 测试宿主；不要直接套用旧 Unity 的 `-batchmode/-nographics` 参数。

### 隔离配置并确认实际加载

1. 为本轮创建新的绝对路径，传入 `-savedatafolder=<runRoot>`。
2. 在该目录的 `Config/ModsConfig.xml` 指定场景所需的 Core、DLC、第三方依赖和目标 mod；以 About 中的 packageId 为准。
3. 隐藏/失焦运行时，在该目录的 `Config/Prefs.xml` 明确设置 `runInBackground=True`。启动使用 `Start-Process -WindowStyle Hidden -PassThru`，保存返回的进程对象和 PID。
4. 进入游戏后记录实际激活列表、目标 DLL 路径和版本，确认加载的不是另一份本地/工坊副本。

`-mod` 会自动加入已安装且兼容的官方 DLC；严格 DLC 矩阵应使用独立配置并核对实际列表，不再附加 `-mod`。不要假设它自动补全第三方依赖。参数语义和隔离限制见 [debugging.md](debugging.md#独立测试启动)。

本机 Waylith 的引擎日志曾实测**不随 savedatafolder 重定向**。游戏测试应串行安排，避免与玩家会话争用日志或锁定同一个待重建 DLL；已有玩家会话不由测试脚本关闭。标准输出和错误输出分别重定向到本轮目录；每个阶段结束、下个进程启动前，根据 `Player.log` 头部的 Process/Command 确认归属并归档。路径见 [日志优先级](debugging.md#日志优先级)。

### 游戏内驱动只在显式测试模式启用

新建驱动时，按以下顺序实现；具体挂接成员在当前 1.7 程序集中核实后再使用。

1. **启用检查**：要求专用命令行开关，并验证 savedatafolder 位于该 mod 的测试产物目录。普通游戏不安装测试补丁、不自动生成地图、不写结果、不退出游戏。
2. **等待就绪**：Def/资源完成加载，再等地图存在、长事件完成。Mod 构造器中不创建依赖 Def 的测试对象。
3. **布置场景**：固定种子和必要对象，记录对象 ID、初始值；准备工作不得直接执行待验证行为。
4. **走真实入口**：测试 AI 就让真实 AI 选任务，测试工作就让真实 Job/Toil 推进；测试右键/按钮则走控件和输入链。直接发 Job 只能证明 Job 行为，不能证明用户输入可用。
5. **推进并观察**：用明确阶段和条件推进，记录关键事件与 tick。达到条件才断言；超过 tick 预算则报告失败，不靠固定睡眠猜测完成。
6. **写结果并退出**：记录每个断言的名字、布尔结果、预期/实际值，附完整异常和最终阶段。结果完整写入后正常退出；未捕获崩溃由外层超时、退出码和缺失报告识别。

可参考 [V1Integration.Configure / Update / Check / Finish](../Mods/RimLayers/Source/RimLayers/Diagnostics/V1Integration.cs)。其中 `-rimlayersv1test`、`-rimlayerscase`、`-rimlayersmode` 均是 **RimLayers 自有协议**；仅给其他 mod 加这些参数不会执行测试。其 `-quicktest` 与初始化挂钩也须连同驱动理解，不能把快速开图等同于完成验收。

驱动若为固定场景接管 tick、冻结部分 Pawn 或注入输入，应在报告中写出适用范围；测试改变了待验证调度时，补一条未被接管的真实调度回归。UI/渲染另检查实际截图，断言“调用了 Draw”不足以证明画面正确。

### 存读档必须跨进程

- **generate**：布置场景 → 执行到检查点 → 记录预期状态 → 使用游戏保存路径存档 → 写结果 → 退出。
- **load**：启动新进程，使用同轮隔离目录 → 读取该检查点 → 等交叉引用恢复 → 断言数据与实体引用 → 继续若干实际 tick，检查行为可续行。
- 旧档兼容用有明确来源的测试存档副本，或在本轮生成的副本中构造缺字段条件。不得就地修改玩家存档；删除某字段的合成夹具仅证明这一兼容条件。
- 前一阶段失败就停止后续验收，保留已生成的存档与报告。不要改断言或重造输入来把失败轮次改成通过。

## 5. 本地可直接复用的 RimLayers 入口

以下命令会构建 RimLayers、写入其测试产物目录并启动游戏，应在具体 mod 测试任务中执行。只补文档时不运行。先确保没有会锁定其 DLL 的现有会话；本节不适用于其他 mod。

```powershell
# 导航场景：脚本自动依次执行 generate 和 load
& './Mods/RimLayers/Tests/Run-V1.ps1' -Scenario navigation -TimeoutSeconds 240

# 输入场景：走该项目已有命令输入验收
& './Mods/RimLayers/Tests/Run-V1.ps1' -Scenario command-input -TimeoutSeconds 240

# 跳跃场景：脚本选择 Royalty，并执行生成和重启读档阶段
& './Mods/RimLayers/Tests/Run-V1.ps1' -Scenario jump -TimeoutSeconds 240
```

[Run-V1.ps1](../Mods/RimLayers/Tests/Run-V1.ps1) 的实际行为：

- 生成 `Tests/Artifacts/v1-<scenario>-<时间>/`，写独立 Prefs 和 ModsConfig，Release 构建后启动隐藏进程。
- 按场景决定 `generate/load` 或额外读档阶段；不是每个场景都测读档。
- 每阶段写标准输出/错误日志，读取 `<mode>-<scenario>-results.json`；缺报告、断言失败或超时抛错。默认墙钟超时为**每进程 240 秒**。
- JSON 含 `mode`、`failures`、`errors`、`checks`、`activeMods`、`assemblyMvid`。进程正常退出本身不代表断言通过。
- 超时会终止本脚本刚启动的子进程；不按进程名批量结束游戏。
- `-CheckPure` 当前只跑脚本枚举的七组测试，**不含 JumpTests 和 SettingsTests**。用到这两部分时单独运行对应工程。
- `-PlayerMods` 会读取玩家的完整启用列表，不适合作为默认最小依赖场景；只有兼容性任务需要该组合时才用。

该脚本目前仅打印游戏退出码，未据此拒绝非零退出，也未强制检查报告非空、预期断言集合或实际 mod 列表是否匹配。因此验收还需执行下一节的完整性判定，不能只采信末尾 `acceptance passed`。脚本保留 stdout/stderr，但不会逐阶段自动归档全局 Player.log；启动下一阶段前需要保留该日志时，在 runner 对应阶段加入归档步骤。

## 6. 用报告判定完成，而不是搜索一个 PASS

通用 runner 应采用下列判定；上节旧入口未覆盖的项目，在接入自动验收时补到对应 runner。

| 结果 | 判定条件与处理 |
|---|---|
| 通过 | 所需阶段齐全，报告属于本轮；预期断言均执行且通过，零相关运行错误，配置匹配，进程正常退出 |
| 失败 | 断言失败、相关异常、非零退出；保留第一条相关错误及其前后上下文 |
| 未完成 | 超时、崩溃后缺报告、报告损坏或为空、少阶段/少关键断言；不得转为通过 |
| 未执行 / 阻塞 | 缺 SDK、依赖或权限，或不能干扰现有玩家会话；记录具体原因和恢复条件 |

通用报告应包含 `runId/scenario/stage/status`、游戏/mod 版本、实际加载列表、起止时间和 tick、预期用例 ID、逐项检查、异常、检查点路径；最终状态只在整轮完成后写出。RimLayers 已有加载列表、逐项检查和错误等字段；其余字段是接入通用 runner 时的建议，消费现有报告时不要假设它们存在。

关键检查：

- 不使用“最新目录”猜测本轮结果；消费脚本返回的确切 runRoot。结果数为零、仅 setup 通过、缺最后一阶段都不算完成。
- 将预期断言 ID 与实际报告对照。可变参数场景按输入生成预期集合，不能只把当前输出条数当作覆盖标准。
- 同时看结构化错误与从启动开始的完整日志。挂钩安装前的错误、引擎错误、退出阶段错误可能不在游戏内收集器中；有报告也要检查退出码。
- 截图对应同一候选和同轮阶段，检查控件状态、实际对象、标签与遮挡；失败保留现场，不用另一轮截图替换。
- 超时先区分加载未完、后台暂停、测试状态机等待和真实行为停滞。墙钟 watchdog 只负责结束本轮等待，不替你推断原因。

推荐 CI 顺序为“静态检查 → 构建 → 相关纯逻辑 → 有本地游戏安装的专用环境执行集成 → 汇总”。不具备游戏安装或图形环境的任务只报告它实际完成的层级；不要分发游戏 DLL 来搭建公共测试环境。

## 7. 交付一份能复现的结论

记录放在目标 mod 自己的验证文档中，链接到本轮产物：

```text
改动/候选：
游戏版本、目标 mod 版本与实际加载组合：
执行命令、runRoot：
静态/XML、构建、纯逻辑：通过 / 失败 / 未执行
游戏阶段：generate / load / 其他；关键断言结果
行为结论与范围：
日志、结果 JSON、检查点、截图：
未通过或未执行项：原因；谁执行什么后可继续验收
```

修复后只重跑受影响的阶段链和相关回归；候选 DLL 改动后必须重启，不能沿用旧进程结果。存读档关联阶段应使用匹配的候选与检查点。日志归属、隐藏窗口暂停、DLL 锁定及输入链排错见 [debugging.md](debugging.md)。
