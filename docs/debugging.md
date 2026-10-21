# 验证、日志与排错

自动化执行、游戏内测试驱动、跨进程读档和结果判定见 [自动化测试流程](testing.md)；本页保留日志排错与按改动选择测试的矩阵。

## 最小反馈循环

1. 读目标 mod 的有效内容目录与项目引用。
2. 做局部修改；XML 解析、XPath 命中或 C# 构建先得到快速结果。
3. 从输出目录确认游戏将加载的是刚生成的自己的 DLL；本地/工坊同包副本尤其要确认。
4. 用独立测试数据和最小依赖启动；开开发者模式读取第一条相关错误。
5. 只复现本次改动的行为，再加上存读档、多图和可选依赖等必要场景。
6. 交付明确区分“静态检查通过”“编译通过”和“游戏内验证通过”。

## 静态 XML 检查

在游戏根目录执行，先把 `$targetMod` 改成真实目标。它只证明 XML 语法完整，不验证 C# 字段或交叉引用。

```powershell
$targetMod = 'Mods/目标Mod'
$xmlFailures = @()
Get-ChildItem -LiteralPath $targetMod -Filter '*.xml' -Recurse -File | ForEach-Object {
    try {
        $xmlDoc = New-Object System.Xml.XmlDocument
        $xmlDoc.Load($_.FullName)
    } catch {
        $xmlFailures += "$($_.FullName): $($_.Exception.Message)"
    }
}
$xmlFailures
if ($xmlFailures.Count -gt 0) { throw 'XML parse failed' }
```

检查原版某个明确 XPath 的示例：

```powershell
$defDoc = New-Object System.Xml.XmlDocument
$defDoc.Load((Join-Path (Get-Location) 'Data/Core/Defs/ThingDefs_Items/Items_Resource_Stuff.xml'))
$matchesFound = $defDoc.SelectNodes('Defs/ThingDef[defName="Steel"]/statBases/MarketValue')
if ($matchesFound.Count -ne 1) { throw "Expected 1 match, got $($matchesFound.Count)" }
```

跨 mod 的补丁要测试实际合并 DOM，不能用单文件命中代替全加载链验证。XML 继承前后目标不同，见 [xml-patches.md](xml-patches.md)。

## 独立测试启动

新启动参数应在准备好具体改动后使用；本次编写文档没有启动游戏或改动玩家配置。

```powershell
# 示例命令；替换包 ID 和测试目录。不要与正在运行的玩家会话共用存档目录。
& './RimWorld.exe' '-savedatafolder=G:/SteamLibrary/steamapps/common/RimWorld/DevTestData/Example17' '-mod=example.agentdemo'
```

已核实本地 `ModsConfig.TryGetCommandLineModList` / `ApplyCommandLineModList`：

- `-mod=a,b` 或 `-mod=a;b` 接受逗号/分号分隔；也支持 `-mod` 与后一个参数。
- 优先按 packageId 匹配，之后尝试 mod 文件夹名；找不到会写 `-mod: no installed mod matches ...`。
- 这条路径自动加入 Core 与已安装、版本兼容的官方 DLC，再加入指定 mod，并排序。
- **它不是“严格只加载给出的项目”，也不是无 DLC 测试命令。** 要测缺 DLC/不同依赖组合，在独立测试数据目录中明确配置并检查最终激活列表。
- 不要假设第三方依赖会自动补全；Harmony/框架依赖按场景显式列出，并检查启动日志。

`-savedatafolder` 重定向游戏读写数据；只改 `-mod` 不等于隔离玩家配置。用多个独立测试目录区分场景，而非覆盖玩家原有 ModsConfig。启动游戏可能改变窗口和运行状态，任务只要求文档时无需启动。

## 日志优先级

当前 Waylith 日志位于 `C:/Users/zonre/AppData/LocalLow/Ludeon Studios/RimWorld by Ludeon Studios/Logs/Player.log`，上次启动日志为同目录的 `Player-prev.log`。2026-10-17 在 `1.7.4981 rev136` 实测：`-savedatafolder` 隔离了配置、存档和缓存，但没有重定向引擎日志。根据日志开头的 `Process`、`Command` 和 `Log file` 确认对应测试进程；重启前将需要保留的该次日志复制到测试记录目录，避免被下一次启动轮转。

```powershell
$testLog = 'C:/Users/zonre/AppData/LocalLow/Ludeon Studios/RimWorld by Ludeon Studios/Logs/Player.log'
if (Test-Path -LiteralPath $testLog) {
    Get-Content -LiteralPath $testLog -Tail 160
}
```

先找启动/操作后出现的**第一条相关错误**和完整异常链。后续 NullReference 常是加载失败的连锁反应。`Log.ErrorOnce` 抑制重复日志，没再出现不等于修好了；重启或按复现条件确认。

| 症状 | 优先检查 |
|---|---|
| mod 未出现 / 未加载内容 | About.xml、有效根目录、支持版本、实际 foldersToLoad、同包副本 |
| `Could not load reference` / 找不到 Def | defName、类型、依赖是否激活、MayRequire、交叉引用时机 |
| XML field 不存在 / type 不可解析 | 字段大小写、Class 与 compClass 混淆、新 DLL 是否加载、实际数据类型 |
| Patch 失败 / XPath 零命中 | 继承前 DOM、补丁顺序、目标 mod/字段是否存在 |
| `ReflectionTypeLoadException` | 先看 LoaderExceptions；缺程序集、旧框架引用、可选依赖被提前引用 |
| `MissingMethodException` / `TypeLoadException` | 引用了旧 build DLL、类型转发、签名或访问级别变化、重复程序集 |
| Comp 不执行 | thingClass 是否支持 Comp、comps 格式、tickerType、delta 路径、实体是否 Spawned |
| 保存后状态丢失 | Scribe label/mode、错误 LookMode、初始化是否覆盖恢复值 |
| 读档时对象/引用为空 | cross-ref 阶段未完成、对象已销毁/缺定义、缓存重建时机 |
| 缺图/紫块 | 逻辑路径、Graphic 类型、文件/目录、大小写、打包资源和 shader |
| Harmony 安装失败 | 当前 1.7 Harmony、重载、目标声明类型、参数注入、IL 匹配 |
| 数值与 Def 不一致 | StatWorker/StatPart、stuff、品质、健康状态、其他 Patch、UI 缓存 |

## 依改动选择测试

| 改动 | 必测 |
|---|---|
| Def 数值 / XML Patch | 启动无相关红字；目标数值正确；不影响非目标 Def |
| 新物品 / 建筑 | 生成、建造、交互、销毁；需要时最小化/重装/合堆 |
| Comp / 全局组件 | 正常模拟、暂停/加速、离图/多图、卸载和缓存清理 |
| Job / WorkGiver | 可达/不可达、预订冲突、强制工作、目标消失、工作中断、保存恢复 |
| 新持久状态 | 新档、保存、重启读档；缺新增字段的旧档副本 |
| 可选 DLC / 框架兼容 | 依赖启用与未启用；实际加载目录和补丁行为正确 |
| Harmony | 安装、目标行为、非目标行为、相关 mod 共存、重启复现 |
| 翻译 / UI | 英文/简中、长文本、占位符、窗口尺寸 |

不要为了文档或低影响改动添加模拟游戏内部实现的冗余测试。代码逻辑可独立测试时做有意义的单元测试；依赖 Def 加载和游戏调度的结论靠集成验证。

## 栈溢出与反编译恢复的父类调用

“无法创建新的堆栈防护页面”对应 `STATUS_STACK_OVERFLOW`。空闲物理内存充足也可能因单线程无限递归而触发；日志末尾的普通消息不是崩溃方法的证明。

2026-10-17，`1.7.4981 rev136` 的 kemomimihouse 崩溃现场已验证：保留错误弹窗，用已安装的 CDB `-pv -p <PID>` 非侵入式读取进程；使用 WinDbg 自带 SOS，`!setclrpath <GameRoot>` 指向当前游戏自带 `mscordaccore.dll`。普通 `!clrstack` 在主线程进入系统错误弹窗后可能为空；`k 50` 与 `!dumpstack -EE` 仍可查出原生栈和栈中托管返回地址。先保存决定性调用栈；完整内存转储可能很大，仅在需要对象内容时再抓取。游戏使用 .NET 10，依赖 `mono-2.0-bdwgc.dll` 的旧 Mono 栈诊断不能代替它。

恢复反编译源码时，`((ThingComp)this).PostSpawnSetup(...)` 仍执行虚分派，会再次进入当前 override；调用父实现应写 `base.PostSpawnSetup(...)`。同样检查 tick、序列化、设置保存和渲染回退的 override。核对生成 DLL 的 IL：父实现使用非虚 `call`，而不是经由当前实例的虚调用。签名检查与编译成功不能发现这类语义错误，必须走对应游戏生命周期回归。

证据与本地修复见 [sources.md 的 kemomimihouse 条目](sources.md#kemomimihouse-17-移植2026-10-17)。

## 性能排查

先确定是加载时间、模拟 TPS、渲染帧率还是 GC 分配。记录地图规模、Pawn 数、速度和依赖组合，再比较相同条件下的改动。

高频路径避免 LINQ/临时 List、每 tick 全图扫描、重复反射、ContentFinder/Material 创建和无限日志。使用地图索引、事件注册、dirty 标记和正确的 tick 分摊；delta 路径注意时间累计。缓存必须有失效策略，多图、读档和对象转移都可能使其失效。

只在测到瓶颈后做针对性优化。不要通过修改所有 Pawn tick、禁用异常或强制吞掉加载错误来“提高性能”。


## 隐藏测试进程必须明确后台更新策略

2026-10-18 / 1.7.4981 rev136：PrefsData 的 runInBackground 初始为 False；PrefsData.Apply（060039B8）设置 Application.runInBackground。隐藏或失焦的真实渲染进程应在自己的独立 `Config/Prefs.xml` 明确写 True，否则没有后续测试进展不能直接判为模拟死锁。保留原始未完成日志，核实进程状态；能继续时复用有效检查点，不用重生成替换输入。不要因此修改玩家偏好。RimLayers 的实际恢复与计时口径见 [WORK-LOG](../Mods/RimLayers/Docs/WORK-LOG.md)。

Windows 下运行中的 .NET 游戏会锁定当前 mod DLL。等待自己启动的进程自然退出后再输出同一路径；MSBuild 复制重试不能刷新已加载程序集，也不要为解锁终止用户会话。


## 任意右键都无效：检查原版菜单是否初始化

1.7.4981 rev136 的 `FloatMenuMakerMap.GetProviderOptions` 在提供器外层 foreach 直接访问 `providers`。若这里空引用、没有具体 `FloatMenuWorker` 名称，先向前找启动时 `Could not execute post-long-event action`，不要只查导航/预订。

当前 `PlayDataLoader.DoPlayLoad` 的同一完成回调依次运行静态构造、开发模式 `ReportProbablyMissingAttributes`、延迟材质解析、`FloatMenuMakerMap.Init` 和图集烘焙。中途异常可让菜单 providers 留空。2026-10-18 实际玩家日志与独立原生输入复现：Dashboard 的 ConsoleTap.savedLevels 引用当前 UnityStubs 不存在的 StackTraceLogType[]，资产诊断反射时 TypeLoadException，继而所有右键失效。RimLayers 的定向字段兼容及实际输入证据见 [命令修复](../Mods/RimLayers/Docs/COMMAND-INPUT-VALIDATION.md)。这条因果不自动覆盖射击按钮，因为 Targeter 并不依赖菜单提供器。

命令回归应从真实 UIRoot / Selector / Command_VerbTarget / Targeter 到 Job 和 Tick，不能直接 TryTakeOrderedJob 后称右键输入通过。跨层输入还应核实真实 Thing 身份、查看层及楼板遮蔽。自由射击可能掩盖未提交的攻击命令；测试应关闭自由射击，并比较每次命令前后的发射时间戳。


在当前 UnityStubs 中，`new Event()` 的 type/rawType 都默认为 Repaint，设置 type 不会同步 rawType。诊断构造鼠标事件需显式设置两者，并同步 UI.MousePositionOnUI 和 UI.MouseMapPosition 的屏幕/地图输入。仅替换地图落点、却让原生窗口和控件读取实际桌面鼠标，不能视为一致的用户输入。相关探针失败与最终事件记录保存在 RimLayers 的命令修复验收中。
