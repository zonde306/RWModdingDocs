# 基准、证据入口与维护记录

## 当前安装与局部复核（2026-10-17）

当前版本为 `1.7.4981 rev136`，runtimeconfig 为 `net10.0 / 10.0.13-waylithclr.dev`，主程序集 MVID 为 `f1e6382600a84191bbc6d8a184fd6b57`。局部复核与实际游戏测试见本页末尾的 Lost Forest 合并迁移记录。

## 前次安装与增量复核（2026-10-16）

本轮读取 Version.txt 和主程序集确认安装已更新。下表是 2026-10-16 的历史安装；旧专题及其历史验证记录仍保留各自版本，不视为全部复核。

| 项目 | 当前值 |
|---|---|
| Version.txt | `1.7.4980 rev60` |
| Assembly-CSharp.dll 大小 | `15632384` bytes |
| SHA-256 | `EFF7D97949F919FB2B151EBFB906E9BE749B63410B7DBDA78EEEBB7972B245C3` |
| MVID | `8e962ecc7ac742d28f3b1b16cd24d2cd` |
| runtimeconfig | `net10.0` / `10.0.10-waylithclr.dev` |

1.7.4980 本轮增量复核：Pawn.DeSpawn / SpawnSetup、Pawn_JobTracker.CleanupCurrentJob / StartJob、WealthWatcher.WealthTotal / ForceRecount、TerrainGrid.SetTerrain 方法体；RoofGrid.SetRoof 与 Pawn_JobTracker.EndCurrentJob 符号/签名。其余旧 API、代码块构建和游戏行为没有全面回归。

本地参考样本 [巴尔「楼上楼下」](multilayer-reference-baal.md) 的 About 标注适配 1.7.4977；其 DLL 与当前游戏版本分别记录，未运行验证。以下早先记录中“本页基准”指旧版表，不是当前安装表。

## 历史文档基准（1.7.4979）

核对日期：**2026-10-15**；范围：用户指定的本地 **RimWorld 1.7**。其他版本不在本知识库承诺范围。

| 项目 | 值 |
|---|---|
| 游戏根目录 | `G:/SteamLibrary/steamapps/common/RimWorld` |
| Version.txt | `1.7.4979 rev109` |
| Assembly-CSharp.dll 大小 | `15622656` bytes |
| Assembly-CSharp.dll SHA-256 | `A97A01C4319414414C0CA39F3565215E83300DC608DC94B69A27720095B4E751` |
| Assembly-CSharp.dll MVID | `425a593954d7432a8f44e3076b25fce9` |
| runtimeconfig TFM | `net10.0` |
| included framework | `Microsoft.NETCore.App` / `10.0.10-waylithclr.dev` |
| Harmony 程序集 | 工坊 `2009463077/1.7/Assemblies/0Harmony.dll` |
| Harmony 程序集版本 / MVID | `3.0.0.0` / `f89a6d4239504658a80568a32790beea` |
| Harmony mod About 版本 | `3.0.1.0`，packageId `brrainz.harmony` |

## 哪项知识依据什么

| 结论 / 专题 | 首选证据入口 |
|---|---|
| 引擎兼容、.NET、引用路径 | 根目录 runtimeconfig、deps、UnityStubs/Waylith 文件；不是旧 Managed 路径 |
| 内容目录选择与文件优先级 | `Verse.ModContentPack.InitLoadFolders`、`GetAllFilesForMod`、`GetAllFilesForModPreserveOrder` |
| 条件目录属性 | `Verse.ModLoadFolders.LoadDataFromXmlCustom`、`Verse.LoadFolder.ShouldLoad` |
| 模块加载与 Mod 构造时机 | `Verse.LoadedModManager.LoadAllActiveMods`、`CreateModClasses`、`Verse.ModAssemblyHandler.ReloadAll` |
| XML Patch → 继承 → Def 创建 | `LoadedModManager.ParseAndProcessXML` 及前置调用 |
| 1.7 新 XML 解析 | `Verse.DirectXmlToObjectNew.DefFromNodeNew`、`ValidateMayRequires`、`XmlToObjectUtils` |
| 继承与列表合并 | `Verse.XmlInheritance.RecursiveNodeCopyOverwriteElements`、`IsListElement` |
| Patch 操作语义 | 各 `Verse.PatchOperation*` 的字段与 `ApplyWorker` |
| DefOf / references / startup | `Verse.PlayDataLoader.DoPlayLoad`、`StaticConstructorOnStartupUtility.CallAll` |
| Tick、interval 与 Rare | `Verse.ThingWithComps.Tick` / `TickInterval` / `TickRare`、`Verse.GenTicks` |
| 组件生命周期与构造差异 | MapComponent / WorldComponent / GameComponent 类型；`Verse.Game.FillComponents` |
| Scribe / API 签名 | 当前程序集相应类型的元数据与所需方法体 |
| JobMaker / RecipeWorker 命名空间 | `Verse.JobMaker`、`Verse.RecipeWorker` 的类型解析 |
| `-mod` 的分隔符与自动 DLC | `Verse.ModsConfig.TryGetCommandLineModList`、`ApplyCommandLineModList` |
| savedatafolder | `Verse.GenFilePaths.SaveDataFolderPath`、本地数据目录存在性 |
| 基础 Def 与格式 | `Data/Core/Defs/ThingDefs_Items/Items_Resource_Base.xml`、`Items_Resource_Stuff.xml`、`ThingDefs_Buildings/Buildings_Base.xml`、`Buildings_Furniture.xml`、`Buildings_Production.xml` |
| Recipe / Job / 声音 | `Data/Core/Defs/RecipeDefs/Recipes_Production.xml`、`JobDefs/Jobs_Misc.xml`、`SoundDefs/World_Sustainers_Weapons.xml` |
| 纹理与语言目录 | Assets.zip 的相关索引条目、`Data/Core/Languages` |
| Harmony 1.7 | 工坊 About、LoadFolders、1.7 DLL 的类型与方法元数据 |

`ModUpdating.txt` 和 `Readme.txt` 作为辅助格式资料，**其旧版本示例、Unity 路径和日志路径不是 1.7 的最终依据**。工坊 mod 里大量旧版工程也只能作为特定项目的历史上下文。

## 官方外部参考

- 引擎发布说明：[Ludeon：Announcing the Waylith Engine and Update 1.7](https://steamcommunity.com/games/294100/announcements/detail/672882362547178634)。本地运行时与源码接口判断优先使用上表证据。
- [Harmony 3 API](https://harmony.pardeike.net/v3/api/HarmonyLib.Harmony.html)、[Annotations](https://harmony.pardeike.net/v3/articles/annotations.html)、[Injected values](https://harmony.pardeike.net/v3/articles/patching-injections.html)。
- [Harmony Prefix](https://harmony.pardeike.net/v3/articles/patching-prefix.html)、[Postfix](https://harmony.pardeike.net/v3/articles/patching-postfix.html)、[Transpiler](https://harmony.pardeike.net/v3/articles/patching-transpiler.html)。

网络文档用于框架语义，不从搜索结果拼出游戏 API。旧教程与当前程序集冲突时，记录差异并采用本地 1.7 的实现。

## 本次验证记录

以下代码块构建与初始数据检查属于历史 1.7.4979 基准，未在当前 1.7.4980 全量重跑。

已完成以下检查：

- **7 个完整 C# 代码块**：从 recipes、persistence、csharp-api、harmony 原文提取；使用 environment 的项目模板、实际游戏与 Harmony 引用，在临时目录以 .NET SDK `10.0.402` 离线 Release 构建。结果：**0 警告、0 错误**。编译输出只有自己的 DLL/PDB/deps，没有复制游戏或 Harmony DLL。
- **17 个 XML 代码块**：按完整文档或 XML 片段解析，均通过。单独片段不是可直接投放游戏的文件时，所属专题已说明应包在哪个根节点/文件中。
- **58 个本地 Markdown 链接**：目标均存在；所有文档代码围栏成对，未发现编码替换字符。
- Steel 的 `statBases/MarketValue` XPath 在当前原版 XML 中恰好命中 1 次；ResourceBase 的 stackLimit 和示例用到的基础父模板、建筑资源路径已核对。
- **230 个 API 签名**：来自当前程序集的成员元数据，已按明确的类型/成员列表筛选为 api-signatures.md；不是从旧教程补写。
- 已创建游戏根入口、14 篇 docs 专题/索引，以及工坊根目录的共享知识库入口。

没有启动游戏、安装示例 mod、修改玩家启用列表或存档；因此不把静态验证称作游戏内验证。对生成、tick、读档、资源呈现、补丁实际安装和第三方 mod 共存的结论，仍需要在具体开发任务中执行相应游戏内验证。

## 性能优化调研补充

`2026-10-15` 针对十二个用户指定的工坊本地参考完成定向调研，并重新核对根目录 Assembly-CSharp.dll 的 SHA-256，与本页基准一致。新增 [performance.md](performance.md)，完整参考分析与文件指纹见 [RimWorldPerformance17/Research](../Mods/RimWorldPerformance17/Research/README.md)。

新增事实的证据入口为 `Corpse.ShouldTickContents`、`IThingHolderTickable.ShouldTickContents` 的调用点 `Thing.DoTick`、`ThingWithComps.GetComp<T>` / `TickInterval`、`CompDeepDrill.CanDrillNow` / `ValuableResourcesPresent` / `TryProducePortion`、`DeepDrillUtility.GetBaseResource`、`StatPart_GearStatOffset.TransformValue`、`Ideo.IdeoTick`、`JobDriver_ConstructFinishFrame.MakeNewToils`、`ListerBuildings.AllBuildingsColonistOfDef` / `Track` / `StartTracker` / `Add` 与 `TrackingScope` 构造器。均来自当前主程序集的元数据和所需方法体，未导出整个程序集。

工坊 `3697920753` 的 About、LoadFolders 和 1.7 DLL 确认已有 1.7 菜单优化参考；`ArchitectMenuOptimizer.EngineFrame.Count` 使用 `Waylith.Time.FrameCount`，该 DLL MVID 为 `624f065ff32f42aabdb52b73bd645e73`。参考只读，未联网核查最新版本。

新建的 `Mods/RimWorldPerformance17` 无补丁骨架使用 SDK `10.0.402` Release 构建通过，0 警告、0 错误。XML、链接和输出检查结果记录在该项目 Research/validation.md；没有游戏内验证和性能收益数据。

补充核对 `2026-10-16`：主程序集 SHA-256 与本页相同。多线程事实来源为 `GenThreading.ParallelForEach` 及当前程序集调用者、`TickManager.DoSingleTick`、`TickList.Tick`、`PathFinder.PathFinderTick` / `ScheduleBatchedPathJobs` / `ForceCompleteScheduledJobs`。YaOpt 代表入口新增检查 `ParallelPawnJob.Execute`、`JobPredictor.ProcessPawn`、`ParallelMapTickManager.ParellellyPostTickMaps`；其中 Pawn Job 为预测计算，不能由名称推断为完整 Pawn Tick。结论见 [performance.md](performance.md) 和 [项目多线程调研](../Mods/RimWorldPerformance17/Research/multithreading.md)，未引入线程代码，未运行性能测试。

缓存补充核对 `2026-10-16`：同一主程序集方法证据为 `StatWorker.GetValue` 的三个重载、`GetValueAbstract` / `GetValueUnfinalized` / `FinalizeValue` / `GetBaseValueFor` / `SetCacheability` / `ClearCacheForThing` / `StatOffsetFromGear`，`StatExtension.GetStatValue` / `GetStatValueAbstract`，`Frame.WorkToBuild`，`StatWorker_MarketValue.CalculableRecipe`，Map/Game/World 的 GetComponent，`ThingDef.GetCompProperties`、`Def.GetModExtension`、`HediffWithComps.GetComp`、`HediffDef.CompProps` / `CompPropsFor`，`Room.GetStat`、`Reachability.GetCachedResult` / `CanUseCache`、`ReachabilityCache.CachedResultFor`、`GenConstruct.CanGetResources` / `CanGetResources_NewTemp` / `CanConstruct`、`ForbidUtility.IsForbidden(Thing,Pawn)`、`ThoughtHandler.TotalMoodOffset`、`DeepResourceGrid.SetAt`、`Text.CalcSize`、`GenTemperature.TryGetTemperatureForCell` / `TryGetDirectAirTemperatureForCell`。

数据证据为 `Data/Core/Defs/Stats/Stats_Building_Special.xml` 的 WorkToBuild，及 `Stats_Basics_General.xml` 的 MarketValue 与父模板 MarketValueBase。结论及适用条件见 [缓存专项核对](../Mods/RimWorldPerformance17/Research/cache-audit.md)。只有研究文档与路线变化，未实现缓存代码，未进行游戏内或性能验证。

## 多层地图调研补充

2026-10-16：重新核对主程序集 SHA-256，与本页基准一致。新增 [多层地图架构调研](multilayer.md)。新增事实依据为 LocalTargetInfo 的字段、Reachability.CanReachNonLocal、Pawn_PathFollower.StartPath、JobGiver_Work.TryIssueJobPackage、ReservationManager.CanReserve、Pawn.DeSpawn / ClearAllReservations、Pawn_JobTracker.CleanupCurrentJob、JobDriver_EnterPortal / JobDriver_TakeAndEnterPortal.MakeNewToils、WorkGiver_DoBill.TryFindBestIngredientsHelper、StoreUtility 储存搜索签名、Verb.TryFindShootLineFromTo、Projectile.TickInterval、PocketMapUtility.GeneratePocketMap、MapParent.ShouldRemoveMapNow、Game.AddMap 和 Thing.mapIndexOrState / Game.currentMapIndex 元数据。原版 XML 入口为 Jobs_Misc.xml 的 EnterPortal 及 Jobs_Work.xml 的 TakeAndEnterPortal Driver。只读取目标成员及相关数据，没有导出全程序集。

关键边界：DeSpawn 会停止工作并清理所有地图的多类预订；原版入口流程不保连续跨层 Job 语义；射线/弹丸仍属单图；AddMap 最多接受 128 张 Map。跨层路由、任务意图、资源声明和战斗方案均为待实现建议。外部参考与实际阅读范围见专题末尾；1.6 或更旧 mod 不作为本地 1.7 API 依据。此次只有文档改动，无构建/XML 实现、游戏内验证或性能测试。

跨层目标选择补充（2026-10-16）：定向核对 Targeter.BeginTargeting 重载与 CurrentTargetUnderMouse、GenUI.TargetsAtMouse / ThingsUnderMouse、AttackTargetFinder.BestShootTargetFromCurrentPosition / CanSee / GetShootingTargetScore。鼠标候选取自 Find.CurrentMap，视线和掩体评分依赖射手地图。分离 casterMap、viewMap、targetMap，以及切图点选、复合渲染和跨层候选评分，均为设计建议；Waylith 多层绘制接口未核对，未进行游戏内验证。详见 [多层地图调研](multilayer.md) 的目标选择补充。

跨层建造补充（2026-10-16）：用户确定高层只能建在紧邻下层屋顶上。新增 RoofGrid.SetRoof / RemoveRoofUnsafe、MapEvents.RoofChanged 事件签名、RoofDef 字段、GenConstruct.CanBuildOnTerrain / TerrainCanSupport 方法体及 CanPlaceBlueprintAt / CanPlaceBlueprintAt_NewTemp / CanConstruct 签名、RoofCollapseCellsFinder.CheckAndRemoveCollpsingRoofs / RemoveBulkCollapsingRoofs 方法体证据。原版数据入口为 Data/Core/Defs/Misc/RoofDefs/Roofs.xml、TerrainDefs/Terrain_Floors.xml 的 FloorBase、ThingDefs_Buildings/Buildings_Base.xml 的 BuildingBase。SetRoof 有变化通知，RemoveRoofUnsafe 无通知；RoofDef 没有现成楼层承载字段。结构面映射、空洞、施工依赖和失效恢复为设计建议，未完成其余调用链审计、实现或游戏内验证。详见 [多层地图调研](multilayer.md) 的跨层建造章节。

连续跨层寻路与需求调研补充（2026-10-16）：主程序集 SHA-256 再核对与本页一致。新增 [连续跨层寻路与任务协调](multilayer-routing.md)，记录一层经二层到三层、多次取料往返、需求选物/床/娱乐家具/就餐位置/工作台，以及最终目标保持、中断仲裁、资源声明、取消代次和读档匹配方案。来源成员为 Pawn_JobTracker.JobTrackerTickInterval / DetermineNextJob / TryFindAndStartJob / EndCurrentJob / StopAll / StartJob / SuspendCurrentJob / CheckForJobOverride / ShouldStartJobFromThinkTree / TryTakeOrderedJob / ExposeData，JobDriver.IsContinuation / Notify_PatherArrived / Notify_PatherFailed / ExposeData，JobGiver_GetFood.GetPriority / TryGiveJob，FoodUtility.TryFindBestFoodSourceFor / BestFoodSourceOnMap，JobGiver_GetRest.GetPriority / TryGiveJob，RestUtility.FindBedFor / IsValidBedFor，JoyGiver.GetSearchSet、JobGiver_GetJoy.TryGiveJob、JoyGiver_InteractBuilding.FindBestGame / CanInteractWith，Toils_Ingest.TryFindChairOrSpot、WorkGiver_DoBill.JobOnThing。大型方法只输出本次有关片段，未保存全量反编译缓存。原版 XML 入口为 ThinkTreeDefs/Humanlike.xml 的 SatisfyingNeeds 与 Joy/JoyGivers.xml。正文区分已核实事实与待实现协议；仅更新研究、路由及 API 提示，未改游戏行为、构建 mod 或进行游戏内/性能验证。

多层框架实现前置补充（2026-10-16）：主程序集指纹与本页一致。新增 [携物转移、地图生命周期与结构楼面](multilayer-foundations.md)。新证据为 Pawn.SpawnSetup，PawnComponentsUtility.RemoveComponentsOnDespawned / AddComponentsForSpawn / AddAndRemoveDynamicComponents，Pawn_CarryTracker.ExposeData / TryStartCarry，Pawn_DraftController.Drafted / FireAtWill，Thing.DeSpawnOrDeselect，Pawn_PathFollower.StopDead / DisposeAndClearCurPathRequest，GenSpawn.Spawn / CanSpawnAt，TickManager.DoSingleTick，Map.MapPreTick / MapPostTick，PathFinder.PathFinderTick / ForceCompleteScheduledJobs，MapGenerator.GenerateMap，PocketMapParent.ExposeData / MapGeneratorDef，MapInfo.ExposeData / Tile，Map.IsPlayerHome / Tile / TileInfo，WorldObject.Tile，World.ExposeComponents，Game.DeinitAndRemoveMap，MapDeiniter.Deinit / PassPawnsToWorld，PocketMapUtility.GeneratePocketMap / DestroyPocketMap，WealthWatcher.ForceRecount，TerrainGrid.SetTerrain / DoTerrainChangedEffects，GenGrid.Walkable、Verse.AI.PathGrid.CalculatedCostAt / Walkable。对 RoofGrid.RemoveRoofUnsafe 的主程序集直接调用者查询确认 GravshipUtility.GenerateGravship，并读取其对应片段；不宣称覆盖第三方或反射调用。数据证据为 Core WorldObjects.xml 的 PocketMap，MapGeneration/CommonMapGenerator.xml 和 BasePlayerMapGenerator.xml 的生成链，TerrainDefs/Terrain_Water.xml 的 WaterDeepBase 父链。候选实现和 P0 验证步骤均未编译/运行；本轮只更新文档。

本地多层参考补充（2026-10-16）：用户指定工坊 3820071953，包 ID Baal.SecondFloor、名称“巴尔：楼上楼下”，About 声明适配 1.7.4977。样本 DLL 大小 69120 bytes、MVID fa38fb6d4e2149d5b216e112b6970b4b、SHA-256 EB8FEF852716E42E20ECA86D48985296110ED5DF8E36D3A11B2D8FC5A08AE65F。证据为 About/About.xml、Defs/SecondFloor.xml，以及 TransitLeg/FloorRoute/TripPlan/Trips、JobDriver_StairTravel、FloorOrderContinuation、TripSerialization、FloorQuery/QueryMap/QueryPosition、FloorWork/CrossFloorNeeds、Supplies、FloorMedical/HostileStairUse、UpperFloorParent/FloorMap、UpperFloorGenerator、FloorPlacement、StairLandingProtection、FloorWealth 和索引失效相关成员。只定向读取本次所需方法及少量小型类型，不复制资产或缓存完整反编译源码。详见 [本地参考专题](multilayer-reference-baal.md)。发现当前游戏已更新至 1.7.4980，指纹与有限复核范围单列于本页顶部；财富重复累计等结论标记为静态推导，尚未启动游戏复现。工坊和玩家数据均未修改。

## IsoRim 单 Map 多楼层源码调查（2026-10-16）

用户指定 Mods/IsoRim-ProjectZ（Peren.IsoRimProjectZ，Git 66db5b6）与 Mods/IsoRim-1.7（Peren.IsoRim17，Git 1f8f5c2）；读取目标 About、LoadFolders、csproj、LICENSE、相关源码与 Def。两个工作树检查时均无本地改动。随附 DLL SHA-256 分别为 A399156B7451945289735FBDBA4744F170D09918D2CA3EE5FA45FF48C7F789E5（484352 bytes）和 3740005D6C22A60B02D25A6E169754913887F5B89006620ACB91F2EEFB848050（561664 bytes）；未由构建或运行证明源码与 DLL 一致。

证据入口：ProjectZ.cs 的 AnnexMapComponent / ProjectZ 坐标转换；MapGenPatches / GridFixer；PathPatches 的 StairRouter、GenerateNewPathRequest、FindPathNow、跨带移动和读档补丁；ReachPatches 的查询期楼梯/跳落连接、District 抑制与有向缓存；IsoRimBridge、IsoRim Floors.cs 的 IFloorLayout / 绘制转换 / PointerFloorAt / ScreenToMap、Interaction.cs 拾取；Stairs.cs 的 PlaceWorker_UpperFloor；Slabs / FloorRoof / FloorHeat、PZ_Terrain.xml；Combat/Heights、CombatPatches 与 VirtualPatches；TestM1b / TestDeadEnd / TestSave 等仅作为测试场景源码证据，未执行。

主程序集 SHA-256 再核对为本页当前 1.7.4980 值；定向读取 Pawn_PathFollower.TryEnterNextPathCell（MVID 8e962ecc7ac742d28f3b1b16cd24d2cd、metadata 060044A9）确认该方法在阻挡/碰撞/门处理后直接写入 nextCell，没有先拒绝非相邻格。只复核此成员，不宣称完整寻路或渲染运行验证。

新增 [IsoRim 专题](multilayer-reference-isorim.md)，更新总体选型及路由/前置专题的适用范围。单 Map 可保留普通过层 Job，但有限选料距离、剪枝漏路、楼板规则差异和弹丸高度未存档须单独处理；这些风险为源码事实或标注的静态推导。参考目录、DLL、玩家存档和启用列表未改；本轮仅研究文档，无 mod 构建或游戏内/性能测试。

## 单 Map 工作查询与动态路径增量审计（2026-10-16）

主程序集指纹再次确认与当前 1.7.4980 表一致。新增 [multilayer-queries.md](multilayer-queries.md)，定向读取 JobGiver_Work.TryIssueJobPackage、WorkGiver_Scanner.Prioritized、WorkGiver_BuildRoof 声明/元数据；GenClosest.ClosestThingReachable / RegionwiseBFSWorker / ClosestThing_Global_Reachable、RegionProcessorClosestThingReachable.RegionEntryPredicate / ProcessThing；WorkGiver_DoBill.TryFindBestBillIngredients / TryFindBestIngredientsHelper / TryStartNewDoBillJob / TryFindBestIngredientsInSet_NoMixHelper / TryFindBestBillIngredientsInSet_AllowMix；Verse.AI.JobDriver_DoBill.TryMakePreToilReservations / MakeNewToils / CollectIngredientsToils / JumpToCollectNextIntoHandsForBill；Toils_JobTransforms.IngredientPlaceCellsInOrder / SetTargetToIngredientPlaceCell、Toils_Haul.PlaceHauledThingInCell、Toils_Recipe.FinishRecipeAndStartStoringProduct；FoodUtility.BestFoodSourceOnMap / SpawnedFoodSearchInnerScan / GetMaxRegionsToScan、RestUtility.FindBedFor / IsValidBedFor、JoyGiver_InteractBuilding.FindBestGame / CanInteractWith；StoreUtility 的普通仓储与非 SlotGroup 容器查找 / IsGoodStoreCell，GenConstruct.CanGetResources_NewTemp；Pawn_PathFollower.PatherTick / NeedNewPath / TrySetNewPathRequest / SetupMoveIntoNextCell / ExposeData / TryResumePathingAfterLoading。Bill.MaxIngredientSearchRadius 字段为 const 999。

证据还包括原版 WorkGivers.xml 的 BuildRoofs / RemoveRoofs，ProjectZ StairGrid.Changed / Register / Deregister / Building_Stairs.TakeLanding 和 PathPatches 相关入口。确认同 Map 下仍需业务空间适配；普通格子扫描与多处半径的数值反例经只读 Python 复算，不能称为游戏测试。当前路径绕过已删除连接器的结论是源码条件推导，需实际场景复现。没有导出整程序集、编译/运行 mod 或修改参考目录；只保存研究文档与索引。

## 跨层战斗当前原版增量审计（2026-10-16）

新增 [multilayer-combat.md](multilayer-combat.md)。当前 1.7.4980 定向证据：CastPositionFinder.TryFindCastPosition / EvaluateCell / CastPositionPreference；JobGiver_AIFightEnemy.FindAttackTarget / ShouldLoseTarget、其 TryFindShootingPosition 的覆盖列表及 JobGiver_AIFightEnemies 具体实现；Verb.CanHitTargetFrom / TryFindShootLineFromTo；Projectile.ExposeData / TickInterval / CheckForFreeInterceptBetween；Bullet.Impact；Projectile_Explosive.Impact / Explode / ExposeData；CompShield.PostPreApplyDamage、CompProjectileInterceptor.CheckIntercept；Verb_MeleeAttack.TryCastShot；GenExplosion.DoExplosion；Explosion.StartExplosion / AffectCell / GetCellAffectTick / GetDamageAmountAt / ExposeData；DamageWorker.ExplosionCellsToHit / ExplosionAffectCell；GenSight.LineOfSight 与 GenGrid.CanBeSeenOverFast。参考入口另包括 ProjectZ Combat 及 VirtualPatches、IsoRim Projectiles.cs 绘制高度缓存。

主要确认：原版射位搜索范围仍是存储坐标；范围盾与普通格子拦截为不同入口；近战 TryCastShot 的范围失败只警告，提交点需维护触达；爆炸 overrideCells 不会替换所有距离和副作用；在途/引爆/传播状态需要与原版保存协同。样本缺口和坐标带泄漏为源码条件推导，未游戏内复现；仅写研究文档，未改参考项目、DLL 或玩家数据。

## 严格屋顶施工与事件入口增量审计（2026-10-16）

新增 [multilayer-structure.md](multilayer-structure.md)。当前 1.7.4980 定向证据：Frame.CompleteConstruction；Toils_Goto.GotoBuild（06004296，先寻找合法相邻工作格，不要求结构楼板目标预先可站）；Blueprint.TryReplaceWithSolidThing、Blueprint_Build.MakeSolidThing；JobDriver_ConstructFinishFrame.MakeNewToils；TerrainGrid.DoTerrainChangedEffects；RoofGrid.SetRoof / RemoveRoofUnsafe；MapEvents / RoofDef 相关元数据；RoofCollapseCellsFinder.ProcessRoofHolderDespawned / RemoveBulkCollapsingRoofs、RoofCollapseBufferResolver.CollapseRoofsMarkedToCollapse；RemoveRoofUnsafe 的程序集直接调用者查询及 GravshipUtility.GenerateGravship 批量移除相关片段；DropCellFinder.CanPhysicallyDropInto、Skyfaller.HitRoof / SpawnThings / Impact、ExitMapGrid.Rebuild。参考源为 ProjectZ Slabs / FloorRoof / Stairs / HeightRule / GroundOnly / CellFinderPatches / ExclusionPatches，以及 PZ_GroundOnly.xml / PZ_Roofs.xml。

确认最终支撑检查须在 Frame 完工副作用前，SetTerrain 拦截太晚；屋顶直接移除和延迟坍塌需同时覆盖；原版天空/离图入口仍是二维存储矩形。结构事务、逐格严格屋顶规则、整列天空检测均为未实现建议。仅更新文档，未运行构建/游戏；公共系统的电力、设施与环境传播尚需继续定向调查。

## 多层公共系统与调度接口收口（2026-10-16）

主程序集再次核对为当前表中的 1.7.4980 指纹。新增 [公共系统](multilayer-services.md)、[仲裁与声明](multilayer-coordination.md)、[原始需求/实施对照](multilayer-readiness.md)，更新旧专题的后续指引，保留历史与运行未验证边界。

公共系统定向方法/元数据：PowerNetMaker.ContiguousPowerBuildings / NewPowerNetStartingFrom，PowerConnectionMaker.BestTransmitterForConnector / PotentialConnectorsForTransmitter，CompPower.ConnectToTransmitter / PostExposeData，PowerNet 构造 / Map / PowerNetTick，PowerNetManager.UpdatePowerNetsAndConnections_First / TryCreateNetAt / TryDestroyNetAt / Notfiy_TransmitterTransmitsPowerNowChanged；CompAffectedByFacilities.CanPotentiallyLinkTo_Static / CanPotentiallyLinkTo / CanLinkTo / IsValidFacilityForMe / IsPotentiallyValidFacilityForMe_Static、CompFacility.CanLink、CompProperties_Facility 元数据；MapTemperature.MapTemperatureTick、RoomTempTracker.EqualizeTemperature / ThinRoofEqualizationTempChangePerInterval；GasGrid.Tick / GasCanMoveTo / TryDiffuseGases / TryDissipateGases / EqualizeGasThroughBuilding / SetDirect / ExposeData；Fire.TrySpread / VulnerableToRain；GenClamor.DoClamor；WeatherManager.WeatherManagerTick；SteadyEnvironmentEffects.SteadyEnvironmentEffectsTick / DoCellSteadyEffects / AddFallenSnowAt、SnowGrid.CanHaveSnow；GlowGrid.GroundGlowAt / GlowGridUpdate_First / LightBlockerAdded / Removed / RegisterCustomGlower、Verse.Glow.ComputeGlowGridsJob.Execute / Flood。另查 SnowGrid.AddDepth 的程序集直接调用者，以定位原版天气采样入口。

调度/声明定向证据：ReservationManager.CanReserve / CanReserveStack / Reserve / RespectsReservationsOf(Pawn,Pawn) 与 ReservationsReadOnly；Pawn_JobTracker.DetermineNextJob / TryFindAndStartJob / EndCurrentJob / CheckForJobOverride / ShouldStartJobFromThinkTree；ThinkNode_Priority / PrioritySorter / QueuedJob.TryIssueJobPackage；JobGiver_Work.GetPriority；JobDriver.SetFinalizerJob / GetFinalizerJob / IsContinuation；Pawn_RopeTracker.RopingTick / IsStillDoingRopingJob、Pawn_PathFollower.RopeeWithStretchedRopeAtNextPathCell；MechanitorUtility.InMechanitorCommandRange、Pawn_MechanitorTracker.CanCommandTo；Lord.Map / CanAddPawn / AddPawnInternal / Notify_PawnLost。

原版 XML 入口为 ThinkTreeDefs/Humanlike.xml、SubTrees_Misc.xml 的 MainColonistBehaviorCore，Buildings_Misc.xml 的 ToolCabinet、Buildings_Furniture.xml 的 Dresser；样本参考 ProjectZ FloorHeat.cs、IsoRim Glow.cs。已核实事实包括电网四邻接与 6 格用电连接、气体不检查地形行走性、原版并行光照输入、普通续行不能无条件放在 PreMain，以及强制预订分支不走普通 ReservationAdded 通知。新算法、租约转换和跨层公共连接均为未实现建议，未构建、未运行游戏或测试。

收口只针对用户的实施前调研与文档要求：每项原始需求已映射到本地证据、方案/接入点、保存/失效边界和验收。运行测试、原型性能、具体第三方组合以及所选替代分支的完整实现不冒充为本轮已完成成果。没有修改参考项目、原版数据、DLL 或玩家文件。

## RimLayers M1 实现与真实进程验收（2026-10-16）

目标目录为 Mods/RimLayers，第一阶段实现/边界见 [M1 记录](../Mods/RimLayers/Docs/M1-VALIDATION.md)。本节与前述仅研究阶段分开：本轮确实新增代码、构建并运行了独立测试进程。游戏仍为 1.7.4980 rev60，主程序集 SHA-256 EFF7D97949F919FB2B151EBFB906E9BE749B63410B7DBDA78EEEBB7972B245C3、MVID 8e962ecc7ac742d28f3b1b16cd24d2cd。

新增定向证据为 MapGenerator.GenerateMap（06001642）、Map.ConstructComponents（06001456）/ FinalizeInit（0600145B）/ ExposeData（06001458）/ FinalizeLoading（0600145A）；CameraDriver.Update / ApplyPositionToGameObject（060008CF）/ CurrentViewRect / SetRootPosAndSize / JumpToCurrentMapLoc；Section.DrawSection、MapDrawer.DrawMapMesh、DynamicDrawManager.DrawDynamicThings / ComputeCulledThings、GenUI.ThingsUnderMouse、DesignatorManager.ProcessInputEvents / Deselect、CameraJumper / Selector 元数据；ExitMapGrid.Rebuild / IsExitCell / Grid / MapUsesExitGrid；GlowGrid.GlowGridUpdate_First / LightBlockerAdded / Removed。确认视口按 frame 缓存、原版光照任务在 update 内 Schedule.Complete、家园公共出口网格 getter 可返回 null。

存档对照补充：MapFileCompressor.BuildCompressedString（0600112C）/ ThingsToSpawnAfterLoad（0600112E）、CompressibilityDeciderUtility.IsSaveCompressible（06001129）、GenSpawn.Spawn（060038B0）/ SpawnBuildingAsPossible（060038B1）/ WipeAndRefundExistingThings（060038B4）/ SpawningWipes（060038B8）、Building.SpawnSetup（0600280D）、Filth.ExposeData / SpawnSetup、Settlement.ShouldRemoveMapNow（0600E3CC）。实际读档污渍销毁调用栈确认原版阻挡物品/建筑重生时的清理；相关规则补到 [persistence](persistence.md)。

代码参考限用户指定的 MIT ProjectZ 生成/网格修复实现，许可与归属保留在 RimLayers/ThirdPartyNotices.md；没有复制其 DLL 或资产，也没有建立 IsoRim/ProjectZ/Baal 运行依赖。

验证：Release 构建 0 警告/错误；布局 862,803 项通过；独立游戏生成 38 项、新进程重启读档 40 项均 0 失败。最终证据目录 Mods/RimLayers/Tests/Artifacts/m1-20261016-211733，含机器可读结果、保存基线、多图清单和四张已检查截图。组合为 Harmony + Core + 全部本机官方 DLC + RimLayers；没有证明无 DLC、长期性能或任意第三方兼容。所有玩家存档/设置/启用列表、参考项目、原版 Data 和游戏 DLL 未修改。

## rimworld-animations 1.7 移植核实（2026-10-17）

目标 mod：`Mods/rimworld-animations`（c0ffee.rimworld.animations 2.2.1，1.6 目录保留）。新增 `1.7/` 目录：复制 Defs/Patches/Sounds/Textures/Source，新 csproj（net10.0，Krafs.Rimworld.Ref 1.7.*-*，游戏根 Assembly-CSharp/UnityStubs/Waylith.UnityCompat + 工坊 2009463077/1.7 Harmony 3 + `Mods/rjw/1.7/Assemblies/RJW.dll`，ManagePackageVersionsCentrally=false 以脱离根 Directory.Packages.props 的 1.6 版本钉）。Release 构建 0 错误 0 警告。

源码改动仅两处：`GroupAnimationDef.cs` 删未使用的 `using System.Runtime.Remoting.Contexts`（同 RJW 移植）；csproj 排除列表只排 `Assemblies/**`，不排 `Source/Defs`（DefOf 类所在，初版误排导致 CS0117）。

1.7 API 核实（Assembly-CSharp MVID 8e962ecc）：`PawnRenderTree.TryGetMatrix`（Public）、`AdjustParms`/`TrySetupGraphIfNeeded`/`nodeAncestors`/`nodesByTag`（Private，但补丁用 `___field` 注入 + AccessTools 反射，无需 publicize）；`PawnRenderNode.AppendRequests`/`TexPathFor`/`AnimationWorker`/`requestRecache`、`PawnRenderNodeWorker.CanDrawNow`、`PawnRenderer.BodyAngle(PawnRenderFlags)`/`GetBodyPos`、`Pawn_DrawTracker.DrawPos`、`Dialog_DebugRenderTree.RightRect` 均存在且签名不变。动画栈：`AnimationDef`（keyframeParts/durationTicks/loopMode）、`AnimationWorker_Keyframes` 全部 worker 虚方法带 `(tick, def, node, part, parms)`、`Keyframe{tick,offset,angle,scale,graphicState}` 未变；`ExtendedKeyframe : Verse.Keyframe` 直接兼容。

渲染挂点核实：`PawnRenderNodeProperties.subworkerClasses`（List<Type>）与 `PawnRenderSubWorker.TransformOffset` 仍存在，`AnimationPatch_PawnRenderTree_OffsetSubWorker.xml` 的 XML 注入路径有效；原版 `PawnRenderTreeDefs.xml` root 无 `subworkerClasses`，`not(subworkerClasses)` 条件命中。`ThingComp.CompTick` 仍由 `ThingWithComps.Tick()` 调用（1.7 同 tick 保留 Tick 与 TickInterval 双路径），CompExtendedAnimator 的 override 无需改名。

RJW 补丁目标核实（RJW.dll 1.7，MVID b97010b9）：`Bed_Utility.in_same_bed`、`JobDriver_JoinInBed.MakeNewToils`、`JobDriver_Sex.setup_ticks/SexTick/Animate/PlaySexSound`、`JobDriver_SexBaseInitiator.Start/End` 均存在；`WorkGiver_Sex` 已不存在但对应补丁整段被注释，无需处理。Patch_HumanoidAlienRaces（1.4 预编译 DLL）1.6 LoadFolders 未引用，不移植；Milkable-Colonists-Animations 为遗留 PatchAll 骨架，不移植。

LoadFolders.xml 加 `v1.7` 节点（含 NoRimNudeWorld 条目），About.xml supportedVersions += 1.7。未在游戏内验证（需重启游戏加载程序集与补丁）。

## 后续更新流程

```powershell
Get-Content -LiteralPath Version.txt
Get-FileHash -LiteralPath Assembly-CSharp.dll -Algorithm SHA256
```

1. 指纹不变时，直接复用这些已核实事实，只探索本次需求新增的成员。
2. build / 指纹变化时，先更新环境基准；对这次用到的 API、补丁目标和加载行为做定向复查。
3. 保留有价值的完整示例，重新编译；涉及 XML 的变更检查字段、继承、Patch 命中和资源逻辑路径。
4. 更新受影响的专题和 API 快照，记录证据符号及检查结果；不要重新反编译全部游戏。
5. 新知识按“事实 / 建议 / 待验证骨架”区分。只有对复杂底层接口确有需求时再扩展专篇。

## 明确边界

本知识库覆盖日常 Def/Patch、常见 C# 扩展、读写档、资源与翻译、构建和排错。它不是全部游戏类型、Waylith 渲染引擎、网络同步、复杂 QuestGen/Lord/世界生成的完整规范，也不承诺任意第三方框架 mod 的 API。遇到边界时按 [environment.md](environment.md) 定向查证，避免把未知内容写成保证。

## Lost Forest 与 ResourcePackCore 合并迁移（2026-10-17）

基准：本机 `Version.txt=1.7.4981 rev136`；根目录 `Assembly-CSharp.dll` 的 MVID 为 `f1e6382600a84191bbc6d8a184fd6b57`，Waylith 的 MVID 为 `fbe66ae50ee044898f92424459f70a7e`。这是本次定向复核，不代表其他旧专题已整体重验。

目标：`Mods/1543566982 [1.0]Lost Forest (WIP)` 内的 `1.7/`、`Source/LostForest/`；旧 ResourcePackCore 仅作输入，合并取其完整 1.0 内容，保留 291 个显式 Def 标识。八种 PawnKind、资源、配方和建筑的实际加载/运行状态记录在目标目录的 `README_1.7.md` 与 `Source/Validation/`。

定向反编译：Verse.Thing / ThingWithComps 的绘制和 tick 入口；PlaceWorker 的签名；RimWorld.BuildingProperties.ResolveReferencesSpecial；JobDriver.DriverTick / DriverTickInterval；JobDriver_VisitJoyThing 的 delta 入口；PawnRenderer.GetBodyPos / BodyAngle、PawnRenderNodeWorker.CanDrawNow；IncidentWorker_SelfTame.Candidates、IncidentWorker_FarmAnimalsWanderIn.TryFindRandomPawnKind / SelectionChance；StatPart_WorkTableOutdoors / Temperature.Applies；CompPowerBattery 的 tick 与储能接口；Building_Bed.PlayerCanSeeOwners；ThingDefGenerator_Corpses.GenerateCorpseDef、ThingDefGenerator_Meat.ImpliedMeatDefs；Waylith.Logging.FileLog.StartLocked / ResolveFallbackDirectory。旧模组 DLL 按迁移所需恢复其自身源码，没有导出整份游戏程序集。

原版数据入口：Core 的 `ThingDefs_Races/Races_Animal_CatGroup.xml`、`Races_Animal_Base.xml`，`DamageDefs/Damages_MeleeWeapon.xml`，`TerrainDefs/Terrain_Water.xml`，`ThingDefs_Items/Items_Resource_Stuff.xml`，`ThingDefs_Buildings/Buildings_Furniture.xml`、`Buildings_Security_Turrets.xml`，`ThingDefs_Misc/Weapons/RangedMechanoid.xml`，`Stats/Stats_Building_Special.xml`，`WorkGiverDefs/WorkGivers.xml`。

新核实的可复用结论已补入 xml-defs.md、csharp-api.md、debugging.md。实际启动确认 `-savedatafolder` 下保存配置、缓存与测试存档，但引擎日志仍写常规用户目录的 `Logs/Player.log`；应根据日志头的 PID 和 Command 识别测试实例。

## Cherrypie / rjw-debug-patch net10.0 升级（2026-10-17）

目标：`Mods/Cherrypie/Soruce`（Cherrypie.csproj）与 `Mods/rjw-debug-patch/Source/RW_RJW_Patch`（RW_RJW_Patch.csproj）。两者原为旧式 `TargetFrameworkVersion v4.8` 工程，引用 RJW **1.6** DLL 与 `..\..\..\Assembly-CSharp.dll` 相对路径。

改动：改写为 SDK 风格 `net10.0` 工程，引用游戏根 `Assembly-CSharp.dll` / `UnityStubs.dll` / `Waylith.UnityCompat.dll`、工坊 `2009463077/1.7/Assemblies/0Harmony.dll`（3.0.0.0）、`Mods/rjw/1.7/Assemblies/RJW.dll`（MVID `b97010b9303d4e49b6fb117ad83d3d62`），debug-patch 另加 `Mods/rjw_menstruation/1.7/Assemblies/RJW_Menstruation.dll`（MVID `3fa4cba636c64fc8a9c4609bf6e1e7c4`）。全部 `Private=false`；`GenerateDependencyFile=false` 避免 `.deps.json` 进入 `Assemblies/`；两份 `About/About.xml` 的 `supportedVersions` += 1.7，保留 1.6。

1.7 API 差异导致的**运行期**（非编译期）修复，由真实 Harmony 3 `PatchAll` 探针发现：

- `rjw.RMB.RMB_Sex.DoChecks` 在 1.7 是 `private static AcceptanceReport DoChecks(Pawn pawn, Pawn target)`。原 `Patch_DoChecks` 复用 `Patch_can_rape.Prefix(Pawn, ref bool)`，Harmony 抛 `ArgumentException: The existing result contract requires an assignable type`。改为自有 `Prefix(Pawn pawn, Pawn target, ref AcceptanceReport __result)`。
- `rjw.PregnancyHelper.CanImpregnate` 1.7 签名为 `(Pawn potentialFather, Pawn potentialMother, rjwSextype sexType)`，Harmony 按参数名绑定，原 `(Pawn fucker, Pawn fucked, ...)` 抛 `Parameter "fucker" not found`。已按新参数名改名。

仍存在但**与 1.7 无关**的既有问题（未改动，超出本次升级范围）：`Cherrypie.SurgeryRacePatch` 的 `StaticConstructorOnStartup` 在 Def 数据库之外读取 `DefineOf.Surgery_RestoreHymen.recipeUsers`，纯反射加载时会 NRE（游戏内 DefOf 已完成注入，`Main`/`RJWPatch_HarmonyPatcher` 同处 `StaticConstructorOnStartup` 且已在 1.6 工作）；`RJWPatch_GenderChange.cs:90` 使用已过时 API `Genital_Helper.generic_vagina`（编译警告，运行仍可用，官方提示改用 `average_vagina`）。

验证手段（脚本备份在 `C:/temp/rjw-backup-1.6` 之外，探针源码 `Program9/13/16.cs`）：在独立 `AssemblyLoadContext` 中加载 1.7 运行时集合并加载两个新 DLL，解析全部 TypeRef/MemberRef 得 `Cherrypie 106 members 0 failures`、`RW_RJW_Patch 97 members 0 failures`；真实 Harmony 3 `PatchAll` 得 16 个目标全部绑定、`GetAllPatchedMethods` 共 30 个方法；`Patch_DoChecks`/`Patch_CanImpregnate` 修复后所有 `StaticConstructorOnStartup` 通过。Cherrypie 的 6 个 Def XML 按 XML 元素名对 1.7 Def 类型字段做了存在性核对，全部命中。

**未在游戏内验证**：需要重启游戏确认 Def 加载、补丁实际生效与存档行为。

## kemomimihouse 1.7 移植（2026-10-17）

基准：同日 `1.7.4981 rev136`（Assembly-CSharp MVID `f1e6382600a84191bbc6d8a184fd6b57`）。目标 `Mods/kemomimihouse`（psyche.kemomimihouse），原作无源码，v1.6 仅含预编译 `KemomimiCore.dll`（net472，ILSpy 11.1 完整反编译为 12 个类、381 行）。

结构事实：v1.5 与 v1.6 的 XML/纹理仅差 statBases 大小写（`Wildness`）与 zstd 纹理；v1.1–v1.4 的 `AnimalMultiPostures`/`InfoCardPortrait`/`Kemomimi_SpawnLimitation` 三 DLL 及 `AnimalMultiPostures.ThingDef_AnimalMultiPostures`、`Kemomimi.Data.ICPDataXmlLoader`、`Kemomimi.CompProperties_SpawnController` 在 v1.5+ XML 中已无引用，1.7 不需要移植。1.7 版即复制 `v1.6` → `v1.7`（含 zstd 纹理），重建 `KemomimiCore.dll` 为 net10.0（源码在 `Source/KemomimiCore/`），引用游戏根 `Assembly-CSharp.dll`/`UnityStubs.dll`/`Waylith.UnityCompat.dll` + 工坊 `2009463077/1.7` Harmony 3，构建 0 err 0 warn。

1.7 定向核对结论（csdecomp，rimworld17 上下文）：KemomimiCore 引用的全部成员在 1.7 中签名不变——`Verse.Dialog_InfoCard.FillCard(Rect)`（Protected）+ 私有字段 `thing`/`tab`，嵌套枚举 `InfoCardTab : byte` 成员顺序 Stats/Character/Health/Records/Permits 一致；`PawnRenderer.renderTree`、`PawnRenderTree.rootNode/pawn`、`PawnRenderNode.children`（`PawnRenderNode[]` public 字段）/`requestRecache` 均 public；`PawnRenderNode_AnimalPart`/`PawnRenderNode_AnimalPack` 的 ctor `(Pawn, PawnRenderNodeProperties, PawnRenderTree)` 与虚 `GraphicFor(Pawn)` 不变；`PawnRenderNodeProperties_Parent`/`_Carried` 存在；`RestUtility.Awake(Pawn)`、`PawnsFinder.All_AliveOrDead`、`PawnKindLifeStage.bodyGraphicData`、`Widgets.CheckboxLabeled(Rect,string,ref bool,...)` 与 `Widgets.HorizontalSlider(Rect,ref float,FloatRange,string,float)` 签名一致。这些签名检查不覆盖恢复源码的虚分派语义；运行期回归见下。

移植产物：`v1.7/Assemblies/KemomimiCore.dll`（10KB，net10.0，仅引用 Assembly-CSharp+0Harmony，无复制游戏 DLL）、`LoadFolders.xml` += v1.7、`about/About.xml` supportedVersions += 1.7。初次移植仅做编译和签名检查；下述调查补充了生命周期回归，尚未覆盖睡眠/驮包姿态切换及信息卡 UI 交互。


### 栈溢出修复与游戏内回归（同日）

玩家崩溃现场 PID 10552、主线程 `0x2048`。CDB 非侵入式读取配合游戏自带 DAC、WinDbg SOS `!dumpstack -EE`，在主线程扫描到 19,285 条 `KMSL_Comp_KemomimiSpawnLimiter.PostSpawnSetup` 匹配记录（原始栈扫描计数，不等同于精确递归深度）；外层调用为 `WildAnimalSpawnerTick → SpawnRandomWildAnimalAt → GenSpawn.Spawn → Pawn.SpawnSetup → ThingWithComps.SpawnSetup`。证据：`Diagnostics/Crash-20261017-10552/raw-stack.txt`、`FINDINGS.txt`。

根因为反编译源码把父实现调用恢复为 `((BaseType)this).VirtualMethod(...)`，导致当前 override 再次调用自己。已修复 8 处：生成限制组件的 PostSpawnSetup/CompTick/PostExposeData、睡眠组件 CompTick、肖像组件 CompTick、ModSettings.ExposeData，以及两个 PawnRenderNode 的 GraphicFor 回退。均改为对应 `base` 调用；未更改 Def、数量限制规则、packageId 或序列化 label。最终 DLL 的生成回调和两处绘制回退 IL 均已核对：父实现为非虚 `call`。

验证：Release 构建 0 错误、0 警告。独立目录 `DevTestData/Kemomimi17-RecursionFix`，实际加载 Core、五个 DLC、Harmony 3 与本地 kemomimihouse；新建 100×100 地图，11 个 PawnKind/变体全部生成，直接执行三类组件 tick 并加载肖像，运行 600 tick，重复生成限额仍有效，保存/读取后继续 60 tick，所有测试角色与合法准入标记均保留，结果为 `COMPLETE`。游戏画面已检查。探针源码和最终结果：`Mods/kemomimihouse/Source/Validation/`；临时验证 DLL/PDB 已从正式输出移除。原始 DLL、PDB 和 6 个被改源码的备份：`Diagnostics/Crash-20261017-10552/BeforeFix/`。

测试日志仍有 `Names/Animal_konf` 缺失及对应命名语法报错，另有安装目录重复包 ID 和失效 Vulkan 层路径提示；这些未纳入本次递归修复。未以玩家整套模组和正式存档进行长期运行测试。

## animal-hardworking 1.7 迁移（2026-10-17）

基准：本机 `1.7.4981 rev136`，根目录 Assembly-CSharp，MVID `f1e6382600a84191bbc6d8a184fd6b57`；工坊 2009463077/1.7 Harmony 3。输入为目标 mod 的 `1.6/Assemblies/HardworkingExtension 1.6.dll`（256 个含编译生成类型，程序集版本 3.0.3.0），随包 1.5 源码功能较少。仅恢复目标 mod 的完整源码至 `Mods/animal-hardworking/Source/HardworkingExtension/`，新建 net10.0 工程及 1.7 内容，不导出游戏程序集。

定向证据：`Verse.GenRecipe.PostProcessProduct`、`Verse.Pawn.GetDisabledWorkTypes`、`RimWorld.Frame.CompleteConstruction`、`RimWorld.JobDriver_BottleFeedBaby.FeedBabyFoodFromInventory` 与其结束回调、`RimWorld.QualityUtility.GenerateQualityCreatedByPawn` 重载；运行时探针实际检查了默认启用的全部 Harmony 补丁。品质转译器的静态调用 opcode 与喂婴回调实例注入修复见 harmony.md。核对本地 kemomimihouse/v1.7 九个角色的显式 comps 路径。

交付：`1.7/Assemblies/HardworkingExtension 1.7.dll`；About 按工作区规则仅声明 1.7，增加 Harmony 加载顺序；LoadFolders 显式保留旧内容版本映射并选择 1.7 和根纹理，旧文件保留用于回退。原 packageId、Kz 类型名、Scribe 字段及 39 个 Def 标识保持。Release 0 错误/0 警告；默认 43 个补丁类安装成功，12 个按原条件跳过；8 XML 语法通过，260 处类型/425 字段通过。完整输出与复现入口：`Mods/animal-hardworking/Source/Validation/`、`README_1.7.md`。未中断正在运行的游戏；游戏内工作、面板、取食、喂婴及存读档待重启验证，可选兼容组合未验证。


## Mimikko 1.7 迁移（2026-10-17）

基准：`1.7.4981 rev136`，根目录 Assembly-CSharp MVID `f1e6382600a84191bbc6d8a184fd6b57`，Harmony DLL `3.0.0.0`。目标 `Mods/mimikko`（`SM.Mimikko.Unofficial`）。基于现有 1.6 DLL 恢复本模组源码，并新增独立 1.7 目录；未导出整个游戏程序集。

定向证据：`RimWorld.JobDriver_Nuzzle.MakeNewToils`（06004B39）、`Pawn_InteractionsTracker.TryInteractWith`（06008A2A；实例字段 pawn 04005518）；`JoyUtility.JoyTickCheckEnd`（0600558D）；列出 PawnRenderer 与 Building_WorkTableAutonomous 的当前成员核对渲染及 tick 签名。原版对照 `Data/Core/Defs/PawnRenderTreeDefs/PawnRenderTreeDefs.xml` 的 Animal 树。新结论追加到 harmony.md。

Release 构建 0 警告/错误；独立 `Mods/mimikko/1.7/Tests/CompatibilityCheck` 真正安装 13 个核心 Harmony 补丁，核对 163 个 Def/模板节点顶层字段及 98 个 Class 类型，0 失败。独立游戏启动实际加载新 DLL，并于 19:01:02 写出补丁初始化完成日志，无 Mimikko 相关加载错误。证据：`DevTestData/Mimikko17/compatibility-check.log`、`startup-final.log`；使用独立测试配置，组合为 Core + 全部已安装官方 DLC + Harmony + Mimikko。截图接口超时；地图内生成、渲染、培养器、睡眠、蹭蹭、羁绊和重启存读档，以及无 DLC/第三方兼容尚未验证。迁移说明和后续验收清单位于 `Mods/mimikko/README_1.7.md`。


## RimLayers 当前原型回验（2026-10-18）

基准 1.7.4981 rev136，根 Assembly-CSharp MVID f1e6382600a84191bbc6d8a184fd6b57；本地 Harmony 3 / net10.0。定向核实 PathRequest.ValidateInt（06001AD8）、PathFinder.FindPathNow（06001A95）、Pawn_PathFollower.ResetToCurrentPosition、GenClosest 的实际候选扫描、WorkGiver_DoBill.TryFindBestIngredientsHelper（060057C7）、JobGiver_Work.TryIssueJobPackage（06005636）、Frame.CompleteConstruction（0600A94F）、施工 Driver.MakeNewToils（06004C19）、RoofGrid.RemoveRoofUnsafe（0600180E）、GenLeaving 的 Frame 容器转移和退款规则（0600CB2C / 0600CB31），以及 PathGridJob 自定义成本（06001ABE / 06001AC9）。新结论已补充 multilayer-queries / multilayer-structure。

项目实施和实际独立进程证据统一见 Mods/RimLayers/Docs/WORK-LOG.md：连续三层导航/拆梯重规划/在途读档、DoBill 多次取料与产品入库、床/食物查询、旋转占地与最后 tick 施工保护、结构事故与待处理队列重启。构建、纯算法检查与游戏验证分别记录；第一版完整战斗及垂直公共系统尚未完成。


## RimLayers 战斗与爆炸增量（2026-10-18）

基准仍为 1.7.4981 rev136 / Assembly-CSharp MVID `f1e6382600a84191bbc6d8a184fd6b57`。定向反编译本次实际使用的 Projectile_Explosive.Impact / TickInterval / Explode（06002A37 / 06002A36 / 06002A38）、Projectile.ArcHeightFactor / DrawAt（17000878 / 06002A2D）、CompProjectileInterceptor.CheckIntercept（0600A215）、CompShield.ShieldState（17001C50），以及 Explosion 的 StartExplosion / Tick / ExposeData / AffectCell / AddCellsNeighbors（0600285D / 0600285E / 06002863 / 06002865 / 06002868）。

DamageWorker 的 ExplosionCellsToHit / ExplosionAffectCell / ExplosionDamageThing / ExplosionStart（06000A02 / 060009FE / 060009FF / 060009FC）及当前派生 Flame/Vaporize 的覆写已核对；Vaporize 的距离截断为 2.9，基类与派生均需适配。新发现的原版爆炸气体参数保存缺口、弧线绘制和原版追击策略已归入 [战斗专题](multilayer-combat.md)。未导出整份程序集或修改原版文件。

原型输出与独立运行证据在 [RimLayers WORK-LOG](../Mods/RimLayers/Docs/WORK-LOG.md)，包括实际射击、敌人射位、个人/范围盾与 EMP、动态楼板、多格目标、飞行/延迟引爆/传播中重启。构建与游戏验证分别记录，第一版 M2–M5 完整范围仍未验收完毕。


## RimLayers 电网与空气增量（2026-10-18）

基准仍为 1.7.4981 rev136 / 主程序集 MVID `f1e6382600a84191bbc6d8a184fd6b57`。定向重核 PowerNetMaker.ContiguousPowerBuildings / NewPowerNetStartingFrom（06007B4D / 06007B4E）、PowerNetManager 的延迟动作与通知、PowerConnectionMaker 的 6 格查询、CompPower 的连接/注销/保存、CompPowerTrader.ResetPowerVars（06007B05）、Map.FinalizeInit（0600145B）；设施预览/实际连接/数量与增益入口；GasGrid.SetDirect、TryDiffuseGases、GasCanMoveTo、Tick 与耗散。

可复用结论已写入 [公共系统专题](multilayer-services.md)。原型只通过真实输电端子扩展原版网络，未复制能源；浓度使用原版网格，活跃列派生重建。生成期临时尺寸、原版先建电网后调用地图组件、支撑事务不改变玩家发电机开关均有定向回归。

测试组合复核：原版 ModsConfig.ApplyCommandLineModList（06001A0D）无条件加入所有兼容官方 DLC；这条规则已在 debugging.md 记录。RimLayers 旧 V1 的 `-mod` 记录实际是全 DLC，新脚本不再传该参数，在自己独立 profile 配置精确 activeMods / knownExpansions，运行 JSON 记录实际 activeMods 和 MVID。Core 与全 DLC 的空气生成/重启分别通过；其余严格 Core 基线按 WORK-LOG 新记录归档。

## RimLayers 热交换与天空增量（2026-10-18）

基准仍为 1.7.4981 rev136 / 主程序集 MVID `f1e6382600a84191bbc6d8a184fd6b57`。定向核实 RegionTypeUtility.GetExpectedRegionType（06001788）、Room 的温度/露天/边界与变更通知、RoomTempTracker 的屋顶/墙/深层交换、MapTemperatureTick（06001855），以及当前天气逐格入口/降雪积沙、GasGrid.TryDissipateGases、Fire.VulnerableToRain、GlowGrid.GroundGlowAt 与 SectionLayer_LightingOverlay.GenerateLightingOverlay（06001550）。

可复用事实写入 [公共系统专题](multilayer-services.md)：不可走隔离区仍可能有空气 Region；PushHeat 的单位为温差 × 格数；低层天花板不能同时向室外和实际高层重复换热；游戏光照与照明网格分属入口。原型采用独立房间、成对快照热交换和不写假屋顶的逻辑天空；纯算法 100000 随机图、独立游戏热/天空与读档证据见 [WORK-LOG](../Mods/RimLayers/Docs/WORK-LOG.md)。第一版完整范围仍在推进。

同一增量补充：定向核实 TemperatureVacuumSaveLoad 的编码/解码/逐格保存及 Region 应用（06001870 / 06001871 / 0600186E / 0600186F）：原版按 1/16°C 向零截断，不保持任意温度浮点。另核实 DamageWorker.ExplosionStart 的一次中心注热和唯一派生 Vaporize 的基类调用、GenTemperature.PushHeat（06001845）的房间/室外规则。非整数热量保存与三层爆炸总热预算的真实运行记录归入同一 WORK-LOG。

## RimLayers 灾害、警戒与命中侧增量（2026-10-18）

基准仍为 1.7.4981 rev136 / 主程序集 MVID `f1e6382600a84191bbc6d8a184fd6b57`。定向核实 Fire 的 TickInterval / SpreadInterval / TrySpread（0600B160 / 17001FF5 / 0600B166）、FireUtility 的可燃资格与点火（0600B16A / 0600B16B）、Spark.Impact（0600B34D）、GenClamor 的委托入口（060037C9）、GenExplosion.NotifyNearbyPawnsOfDangerousExplosive（06003819）、Pawn_MindState 的通知/天气记忆与 JobGiver_FleePotentialExplosion、Pawn_JobTracker.CheckForJobOverride / TryTakeOrderedJob、原版强制任务判断和直接逃离目标。

CompExplosive 的 StartWick / PostExposeData / ExplosiveRadius（06009CCB / 06009CC1 / 06009CCD）确认保存与平方根堆叠半径；已落地延迟弹丸使用原有 landed / ticksToDetonation。原版数据对照 Core ThingDefs_Items/Items_Resource_Manufactured.xml 的 Chemfuel、ThingDefs_Buildings/Buildings_Production.xml 的 FueledStove；未修改这些原版文件。

原型将空气、热、惊扰与火焰的物理开口资格集中，仍按不同权限传播；真实受管地图通知可复现强制 Wait 被整树重选为 Wait_Wander，现保留强制任务并记录危险。真实板下手雷又暴露障碍所属层与撞击空气侧混用，已按来向接触侧修正；点火副作用保留实际被击占地边缘。新增 19 项纯几何，Core 与全 DLC 各 32 / 10 项灾害生成/新进程读档，以及受影响弹道/护盾/爆炸/空气/热/天空回归均通过，详见 [WORK-LOG](../Mods/RimLayers/Docs/WORK-LOG.md)。当前不是第一版完整验收终点。


### 2026-10-18：危险警告的原生倒计时补证

RimLayers `hazards-timed` 使用同一 1.7.4981 rev136 程序集，不修改 StartWick 后的原版引信。独立 Core `v1-hazards-timed-20261018-134536` 生成 34 / 读档 11 通过，128 tick 原始预算与实际期限相等；全 DLC `134632` 同样 34 / 11，100 tick 期限。真实冲刺、移动、实际爆炸后无伤及读档不重启计时均通过。此前 hazards 的 600→1 可控试验仅作功能/恢复证据，不能替代原生期限证据。具体输入及目录见 [实施记录](../Mods/RimLayers/Docs/WORK-LOG.md)。


### 2026-10-18：M2 非单调/容器/仲裁运行补证

同一 1.7.4981 rev136 / MVID f1e6382600a84191bbc6d8a184fd6b57，定向读取 WorkGiver_DoBill.TryFindBestIngredientsHelper/FixedIngredients、JobDriver_DoBill.CollectIngredientsToils/预订、HaulAIUtility.HaulToCellStorageJob、JobDriver_HaulToCell.MakeNewToils、Pawn_JobTracker.StartJob/CheckForJobOverride、IHaulSource 实现者、Building_OutfitStand 开关及 AddApparel、PathRequest.IPathGridCustomizer 与 PathGridJob.CostForCell。具体成员与原生容器事实见 [queries](multilayer-queries.md)。独立 Core/最小 Odyssey/全 DLC 的 maze、work-save、work-container、work-haul、work-hunger、navigation-rules 结果与输入见 [实施记录](../Mods/RimLayers/Docs/WORK-LOG.md) M2 末节，均终态零失败；不是纯算法替代游戏行为，也没有另存原版材料/任务/预订。


### 2026-10-18：M3 天空落物当前调用方补核

同一当前 MVID，定向读取 Skyfaller SpawnSetup/Tick/HitRoof/SpawnThings/Impact/ExposeData/Destroy、全部已列出的派生类相关成员，DropCellFinder.CanPhysicallyDropInto、GenSpawn.Spawn 的 Thing 完整重载，以及 ActiveTransporter.PodOpen/Tick。确认“放置失败后清空容器”的原生调用方风险；[structure](multilayer-structure.md) 记录重定向时序和内容物恢复约束。仅接口与失败路径调查，尚无天空落物实现或运行验收，不并入既有结构通过项。


### 2026-10-18：M3 天空落物实施与真实回验

基准仍为 1.7.4981 rev136 / MVID f1e6382600a84191bbc6d8a184fd6b57。在前述接入调查基础上，补核 GenPlace.TryPlaceThing/Direct/PlaceSpotQualityAt/TryFindPlaceSpotNear、GenSpawn.CanSpawnAt/SpawningWipes/CheckMoveItemsAside、DropCellFinder.RandomDropSpot、SkyfallerMaker.SpawnSkyfaller、DropPodUtility.MakeDropPodAt、ActiveTransporter 的 Def、GenConstruct.PlaceBlueprintForBuild 与 TerrainGrid.Notify_TerrainDestroyed。精确路径/成员及实现约束见 [structure](multilayer-structure.md)。

生产实现为 RimLayers Structure/SkyfallService.cs、SkyfallPatches.cs；真实 Core/全部官方 DLC 的 skyfall 生成/读档、原版 5×3 skyfall-shuttle，以及结构/工作/爆炸回归见 [实施记录](../Mods/RimLayers/Docs/WORK-LOG.md) M3 末节。没有修改游戏 DLL、参考 mod 或玩家配置；保存仍使用原版 holder 与阶段标志，不另外复制 Pawn/物品。


### 2026-10-18：M4 护盾保存边界与原生手雷期限

基准保持 1.7.4981 rev136 / 根 Assembly-CSharp MVID f1e6382600a84191bbc6d8a184fd6b57。重读 CompProjectileInterceptor.CheckIntercept（0600A215）、Projectile.TickInterval/CanHit/DrawAt（06002A29 / 06002A2F / 06002A2D）、DynamicDrawManager.ComputeCulledThings/DrawDynamicThings（0600119D / 0600119B）与 GenAdj.OccupiedDrawRect（0600048A）。原生拦截 false 不扣盾，原生大 delta 时钟先推进再处理空间，非 Pawn 原生裁剪直接读取 Position；接入与保存约束归入 [combat](multilayer-combat.md)。

Core 手雷数据对照 RangedIndustrialGrenades.xml 的 Proj_GrenadeFrag/BaseFragProjectile/BaseGrenadeProjectile，半径 1.9、延迟 100。结合已核实 Projectile_Explosive.Impact/TickInterval（06002A37 / 06002A36），实际落地后的剩余 99 tick 由原生流程产生，测试未延长倒计时。Core 与全部五个 DLC 的真实撤离/实际爆炸/新进程读档、空气上方屋顶挡弹、最小 Royalty 的护盾成功/放行边界与受影响回归结果见 [WORK-LOG](../Mods/RimLayers/Docs/WORK-LOG.md) M4 末节。


### 2026-10-18：M4 友伤/体型/炮塔与 M5 天气/机体

同一 1.7.4981 rev136 / MVID f1e6382600a84191bbc6d8a184fd6b57，重读 Projectile.CanHit/CheckForFreeIntercept/ImpactSomething（06002A2F / 06002A2B / 06002A30）、PawnUtility.GetPosture（06008322）、Pawn_HealthTracker.MakeDowned（06002500），以及 Building_TurretGun.TryFindNewTarget/Tick/TryStartShootSomething/BeginBurst/IsValidTarget（0600AA44 / 0600AA41 / 0600AA43 / 0600AA47 / 0600AA46）。Core 对照 Races_Animal_SquirrelGroup.xml、Races_Animal_CowGroup.xml、Races_Mechanoid.xml 与 Buildings_Security_Turrets.xml。真实友伤/特殊体型/倒地目标与电池供电的自动炮塔在 Core 和全 DLC 中通过，证据见 [WORK-LOG](../Mods/RimLayers/Docs/WORK-LOG.md)。

天气记忆重核 MindStateTickInterval/CanGainGainThoughtNow（06004522 / 06004523）、CurWeatherLerped（17001550）；原版 Rain / SoakingWet 配置与实测结论见 [services](multilayer-services.md)。机体重核 CanCommandTo/DrawCommandRadius/AssignPawnControlGroup（060085EF / 060085EE / 060085DE）、全局 MechanitorUtility.InMechanitorCommandRange（060000C2）、Verse.Hediff_Mechlink.PostAdd（0600218D）及 PawnRelationWorker_Overseer.OnRelationCreated（060084D9），XML 对照 Biotech Hediffs_Mechanitor.xml。最小 Biotech/全 DLC 的真实控制组、范围、三层路径与二层途中重启证据见 [coordination](multilayer-coordination.md) 和同一 WORK-LOG。当前完整第一版尚未验收完毕。


机体显示增量：重核 Pawn.DrawAt（06002300）、GenDraw.DrawRadiusRing/DrawFieldEdges（060037FF / 06003802）、MapComponentDraw 及调用方（06001483 / 0600148B / 0600C546）。定位 SelectedPawns 重入枚举问题并修复为调用前快照。最小 Biotech / 全 DLC 的 17 / 15 运行与提交检查通过；165050 / 165140 的同镜头画面确认远层圈可见。此前近景缺边判断撤回：圈心在屏外时原生同层对照也不显示边界，保留原生裁剪。具体镜头与证据见 WORK-LOG 显示专项。


### 2026-10-18：牵绳、畜栏与 Lord 当前调用方及回验

同一 1.7.4981 rev136 / 根 MVID f1e6382600a84191bbc6d8a184fd6b57。定向重读 TryEnterNextPathCell（060044A9）、RopeeWithStretchedRopeAtNextPathCell（060044A2）、RopingTick（060088CB）、原生 RopeToDestination / RopeToPen / FollowRoper 和跟随 JobGiver；核对 Pawn.FenceBlocked（170006DF）、TraverseParms.WithFenceblockedOf（0600089F）、PenMarkerState.HasOutsideAccess 及唯一调用方（1700032C / 060099C2）、AnimalPenUtility 的实际筛选/排序/圈内目标、WorkGiver_TakeToPen.JobOnThing / MakeJob（06005689 / 0600568B）。Core 数据对照 Jobs_Work.xml、WorkGivers.xml 的 TakeToPen 与 Buildings_Misc.xml 的 PenMarker。

Lord 重核 MakeNewLord / SetJob / GotoToil / Notify_PawnLost（060048C0 / 06004845 / 06004851 / 06004858）、原生 Verse.AI.Group.LordJob_Travel / LordJob_DefendPoint 的图、保存与职责、TransitionAction_SetDefendLocalGroup.DoAction（06004950；直接采用真实成员位置），以及 Duties_Misc.xml 的 TravelOrLeave / Defend。实际接入结论归入 [coordination](multilayer-coordination.md)，避免把原生落脚等待误判为必须新建绳路协议，也避免把临时 Lord 防御当成永久改派。

独立 Core / 全 DLC：roping 各生成 30 / 读档 24，lord 各 13 / 13，终态零失败，实际双动物及三名敌方队伍均经过二层。畜栏围护、路线排序、征召/取消/拆梯收尾、同 Lord 保存与真实战损已核对。目录、失败探针的前提修正和复用有效存档的加载恢复详见 [WORK-LOG](../Mods/RimLayers/Docs/WORK-LOG.md) M5 末节。第一版的规模与完整矩阵审计仍待完成。


### 2026-10-18：第一版规模、落脚门与世界出口收口

基准同为 1.7.4981 rev136 / f1e6382600a84191bbc6d8a184fd6b57。补核 Region.Allows（06001707）的落脚门方向语义，原始 TraverseParms 须贯穿规划与提交；实际闭门/禁止/敌对改门与备用楼梯证据见 WORK-LOG。运行环境补核 PrefsData.Apply（060039B8）的后台更新策略；独立规模测试记录 120/250 地面、2/3 层、20/60 名真实 NPC，12000 tick 与原始路径/方法计时/进程内存，不用 CPU 提交数据推导 FPS。

出口定向成员：RCellFinder.IsGoodDestination/For/Best/Random/Near（0600C91B–0600C920），CellFinder.RandomEdgeCell 两重载（060037A0/A1）、TryFindRandomEdgeCellWith 两重载（060037AF/B0）、NearWith/InsideWith（060037B1/B3），Reachability.CanReachMapEdge（060016E9）、Region.Neighbors、GetValidRegionAt（06001743），Goto.MakeNewToils/TryExitMap（0600413D/3E）及实际委托（0601087A），Flee.MakeNewToils/ExitMap（06004EE8/E9）、TakeAndExitMap.MakeNewToils/离图委托（060050EE/F0/F2）、Pawn.ExitMap（0600231E）及直接调用方。原生商队补核 CanExitMapAndJoinOrCreateCaravanNow（0600E211）、StartFormingCaravan/FormAndCreateCaravan（0600E21F/1E）、LordJob_FormAndSendCaravan.CreateGraph/SendCaravan（06005A37/3D）、ExitMapAndCreateCaravan（0600E20F）。原生 Core JobDefs 的 Flee / Steal 与实际 Driver 对照后再测试；具体契约归入 [coordination](multilayer-coordination.md)，实际结果及早期失败保存在 RimLayers WORK-LOG。


### 2026-10-18：RimLayers 0.1.0 第一版验收完成

最终步行出口直接替换 Goto / Flee / TakeAndExitMap 原生执行委托中的边界几何，避免 Toil 重建早于布局恢复的漏装问题。Core 181553 / 全 DLC 181730 各生成 37 / 新进程读档 25，真实 Goto、Flee、Steal、原版家园商队均下到逻辑地面后一次离图；现有飞行/传送/运输政策未被全局拦截。当前中文 M1 181433 为 41 / 43，已核对真实地面/空域画面。最终出口补丁复用既有全 DLC 三层 250×250 / 60 人存档，完成 12000 tick、四趟及 12 项零失败规模加载回归；生成输入未替换。

完整 D0/M1–M5 对照、实际测试组合、原始目录、计时与内存及未验证范围见 [第一版验收](../Mods/RimLayers/Docs/V1-VALIDATION.md)。第一版固定三层、建议地面不超过 250，不将方法计时推广为 FPS，不将有限运行推广为多年或具体第三方兼容。天然岩顶、地下、旧图迁移、三维显示、CE 与未选定框架仍是后续适配范围。


### 2026-10-18：RimLayers 0.1.1 六项可用性与旧档加载

基准为 1.7.4981 rev136 / Assembly-CSharp MVID f1e6382600a84191bbc6d8a184fd6b57。定向核实 Map.ExposeData / ConstructComponents / ExposeComponents（06001458 / 06001456 / 0600145C）、MapSerializeUtility.LoadUshort（0600181C）及 DataSerializeUtility.LoadUshort（06003552）、TerrainGrid.ExposeData/ExposeTerrainGrid（06001386 / 0600138A）、FogGrid.ExposeData（0600121D）、GasGrid.ExposeData（060012DC）、SnowGrid.ExposeData（06001358）及 DataExposeUtility 的 byte/bit 解码。MapFileCompressor.ExposeData/ThingsToSpawnAfterLoad（0600112B / 0600112E）在新 NumCells 下还需完整压缩物体前缀。WaterInfo.ExposeData（060014BA）结合 Scribe_Values.Look<T>（06002743）核实：实际赋值只在 LoadingVars，后续阶段不会覆盖已完成的 x-major 重排；外层分支本身不能证明会重复读入。PlanetTile 的 TryParse 与 WorldObject.ExposeData（0600E519）用于明确表面/太空及地图父对象资格。

显示定向核实 MapDrawer.DrawMapMesh/SectionAt（060014D4 / 060014D6）、Section.DrawSection/RegenerateDirtyLayers（060014F3 / 060014F7）、Section.layers、SectionLayer.subMeshes、Pawn.DynamicDrawPhaseAt（060022FF）、Thing.DrawNowAt（06002AA1）、GenUI.ThingsUnderMouse（06003273）、Targeter.CurrentTargetUnderMouse（0600C452）、GenDraw.DrawTargetHighlight（060037F6）及 GenMapUI.LabelDrawPosFor（06001B1D）。键绑定对照 Core KeyBindings.xml 的 GameMapKeyBinding 与 KeyBindingDef.KeyDownEvent/MainKeyLabel。落点使用当前 GenSpawn 完整 overload 与 Pawn.Flying；只防非法新生落点，普通跨层移动不 DeSpawn/Spawn。

用户指定工坊3822131067为 shimakaze.nanami.startrail；只读其 About、Bootstrap/Definitions 和种族定义，未修改。实测出生/保存空域恢复与原装备身份通过；年龄/关系错误在不启用 RimLayers 的独立配置也复现，未标记整体兼容。完整源目录、真实界面、迁移全格与实体审计、有效结果及失败记录见 [0.1.1 更新验收](../Mods/RimLayers/Docs/USABILITY-VALIDATION.md)。


旧河流增量：定向读取 WaterInfo.SetTextures（060014B8）、GenStep_Terrain.Generate（0600776C）、TileMutatorWorker_River.GeneratePostTerrain / GeneratePostFog / GenerateRiverLookupTexture（06007A12 / 06007A13 / 06007A21），对照 Core CommonMapGenerator.xml、EncounterMapGenerator.xml 与 TileMutators.xml。194404 原生生成1350河水格，120×120旧图经加载扩为120×408，全部28800 flow float位模式在首次/转换后重启均保持，上层为零；生成5 / 加载11 / 再加载10与全格实体审计19项均零失败。JSON浮点快照的精度问题不归因为迁移数据损坏；原始失败保留，同一有效源存档复用，未靠新地图替换输入。


### 2026-10-18：RimLayers 0.1.2 楼面遮盖、相机与动态背景

主基准仍为 1.7.4981 rev136 / Assembly-CSharp MVID f1e6382600a84191bbc6d8a184fd6b57。定向读取 TerrainDef.PostLoad（06000C79）、TerrainGrid.GetMaterial（06001379）、SectionLayer_Terrain.Regenerate/GetMaterialFor/AllowRenderingFor（0600157A / 06001578 / 06001579）、Altitudes；动态绘制读取 DynamicDrawManager.DrawThings、Corpse.DynamicDrawPhaseAt（0600284C）、PawnRenderer 的预绘制/RenderPawnAt/GetBodyPos（06001BBF / 06001BBA / 06001BC1）、PawnRenderTree.Draw（06001BDC）、GenDraw.DrawMeshNowOrLater（06003808 / 06003809）。武器读取 PawnRenderUtility.DrawEquipmentAiming / DrawCarriedWeapon / DrawEquipmentAndApparelExtras（06001BF3 / 06001BF2 / 06001BF8），面向读取 Pawn_RotationTracker.Face / UpdateRotation（06002545 / 06002541），Stance_Warmup.StanceTick（0600256A）；装备射手来自公开 Pawn_EquipmentTracker.pawn。

实际延迟绘制实现位于根 UnityStubs.dll，MVID f1feb9ddfd5a43179c5626747b2137fb，非 Waylith.UnityCompat 的 Graphics。定向核实 Graphics.DrawMeshCore / Internal_DrawMesh（06000957 / 06000956）、MaterialPropertyBlock 包装路径、DrawPropertyBlock 独立提交重载（0600095A）及 DrawMeshNow。队列、最终提交与尸体投影结论写入 [资源与渲染专题](assets-localization.md)。

相机核实 CameraJumper.TryJump / 两个 TryJumpInternal / JumpLocalInternal（060008AD / 060008B0 / 060008B1 / 060008B4），CameraDriver.Update / CalculateCurInputDollyVect / JumpToCurrentMapLoc / PanToMapLoc（060008CE / 060008D0 / 060008D4 / 060008D5）、CameraPanner.Moving/Update，以及 ColonistBarColonistDrawer.HandleClicks（0600B7DA）的实际调用。跨层跳转前切层及旧 panner 取消有下一次真实相机更新证据；只改测试配置的 edgeScreenScroll。

只读 Nanami Startrail 的 Source/Rendering.cs，确认其 PostDraw 固定 70/290 度。实际该 worker、Core/全五 DLC 的原生绘制与相同存档重启证据集中在 [0.1.2 显示验收](../Mods/RimLayers/Docs/DISPLAY-VALIDATION.md)；该第三方组合的年龄/关系错误仍如实失败。雾为内置 image_gen 新生成透明 PNG，原始 Alpha、实际使用和最终提示词见 [素材记录](../Mods/RimLayers/Textures/Atmosphere/README.md)。未修改原版或参考 mod。


### 2026-10-18：RimLayers 0.1.3 食物可达性长卡顿修复

基准仍为1.7.4981 rev136 / Assembly-CSharp MVID f1e6382600a84191bbc6d8a184fd6b57。定向复核 Reachability.CanReach（060016E1）、FoodUtility.SpawnedFoodSearchInnerScan（06004A63）、PathFinder.FindPathNow（06001A95）、PathFinderJob.Execute（06001AAA）及 Region.Allows（06001707）。原生CanReach保留Immediate/同District、PathEndMode解析、Region缓存和特殊模式格级分支；FindPathNow才产生完整格级路径。

用户授权的两次WinDbgX -pv非侵入主线程栈记录在 [现场目录](../Mods/RimLayers/Tests/Artifacts/stall-20261018-live)，均为GetFood→FoodSearch→RimLayers完整跨层路径。修复 LayerReachPatch，新增只做连通性的LayerReachability；原生层内查询加合法有向楼梯端点搜索，不生成PawnPath、不改实际完整寻路或成本排序。无新存档状态。

Core/全DLC同进程130食物对照分别636.6508→0.2827ms、822.1789→0.2949ms，真正PathFinderJob执行均1675→0；功能/重启、非单调路线、门/网格、连续工作、牵绳和战斗AI结果及早期失败原文见 [可达性修复验收](../Mods/RimLayers/Docs/REACHABILITY-VALIDATION.md)。玩家165模组原档仍需在重启后回验；未修改玩家配置或存档，也未将外部异常吞掉。


## RimLayers 0.1.4 原生命令初始化（2026-10-18）

基准 1.7.4981 rev136，Assembly-CSharp MVID f1e6382600a84191bbc6d8a184fd6b57。定向反编译 DoPlayLoad 0600093B / 完成回调 0600EC9D、StaticConstructorOnStartupUtility.ReportProbablyMissingAttributes 0600074D、FloatMenuMakerMap.Init 0600C065 / GetProviderOptions 0600C067、Selector.HandleMapClicks 0600C417 / SelectorOnGUI 0600C415、Command_VerbTarget.ProcessInput 06003077、Targeter.OrderPawnForceTarget 0600C451、UIRoot_Play.UIRootOnGUI 0600C868。

Dashboard 的 About/About.xml 确认 rimiris.dashboard / 工坊3800182954，目标 RimirisDashboard.dll MVID ecac7f0afdff42bab210bdca707f7326。定向字段040000EC:F确认ConsoleTap.savedLevels为StackTraceLogType[]，Install/Uninstall也引用该缺失类型；该mod应自行更新此功能。玩家21:53故障日志、独立新图输入复现及修复候选检查记录在 [COMMAND-INPUT-VALIDATION](../Mods/RimLayers/Docs/COMMAND-INPUT-VALIDATION.md)。不将定向启动隔离扩大为第三方整体兼容结论。


同次定向核对 UnityStubs.Event（MVID f1feb9ddfd5a43179c5626747b2137fb，类型0200011D），确认构造、current、type/rawType和Use的实际字段语义，用于修正原生UI诊断事件，不改引擎程序集。最终完整UI事件数据见223722（Core）/223746（全DLC+RimIris Beta+Dashboard）原生测试标准输出；GizmoGridDrawer.DrawGizmoGrid 06003090 / GenUI.AbsorbClicksInRect 06003281 / Mouse.IsOver 060032A2解释固定屏幕落点被控件正常吸收的探针失败。

## 2026-10-19：Odyssey 多层飞船候选

基准 1.7.4982 rev1280，Assembly-CSharp MVID a46e6f45b6164261bb63a9471bf7402a。定向核实 GravshipUtility 生成/航行/到达、Gravship 构造/收集/旋转/保存、Building_GravEngine 合法性、原版放置顺序与 Controller 起降/确认释放。精确 token 和数据丢失陷阱见 [odyssey-gravship](odyssey-gravship.md)，候选源码与有效运行范围见 [项目实施记录](../Mods/RimLayers/Docs/GRAVSHIP-IMPLEMENTATION.md)。

UnityStubs MVID aae7bbfe87c54822ba4fd46cb1a3bd0f。原版读档/完整起降采用独立 Tests/Artifacts 配置和存档，未修改玩家数据。全 DLC 094429 通过 20/20/9 项，包含船边上层墙灯、飞行/落地重载、两次原版起降和已有多层地图返回；095051 用原版落点设计器提交回程，10 项通过。Core 094730 原版命令回归 18 项通过。各 JSON 与截图入口见项目记录，errors 均为空。已有未扩层地图的实时扩容最初被安全审查拒绝；用户后续明确授权“先隔离验证，通过后再接入”。现已完成隔离故障回滚、完整旧网格/实体/任务保留、扩容读档及已有太空目的地原版往返，证据见 [运行时扩容验收](../Mods/RimLayers/Docs/LIVE-EXPANSION-VALIDATION.md)。

### 2026-10-19：运行时扩容与现有飞船目的地

同一 1.7.4982 rev1280 / MVID a46e6f45b6164261bb63a9471bf7402a。定向核实 Map.ConstructComponents 06001456、RegionAndRoomUpdater.RebuildAllRegionsAndRooms 06001726、RegionDirtyer.SetAllDirty 0600173C、FloodFiller 构造/FloodFill/ClearVisited 0600378E/06003792/06003794、PathFinder 完成 jobs/Dispose 06001A8B/06001AA0、Pawn_PathFollower.StopDead/StartPath 06004495/06004494、MapDrawer.Dispose 060014DB、GlowGridUpdate_First 06001255、WaterInfo.GetWaterMovement/SetTextures 060014B9/060014B8，以及 HaulDestinationManager 构造/SlotGroupAt 0600AB14/0600AB1D。已核实的哨兵值、原生句柄所有权、三维仓储数组和河流步长写入 [Odyssey 专题](odyssey-gravship.md)。

核实 CheckConfirmSettle 0600DC2F 的确认委托执行顺序及 ArriveExistingMap / Notify_LandingAreaConfirmationStarted 0600CB6F / 060007E0 的未 Spawn marker 契约。Space 原版模板来自 Data/Odyssey/Defs/MapGeneration/SpaceMapGenerator.xml 与 WorldObjectDefs/WorldObjects.xml；太空防护夹具使用官方 Apparel_Vacsuit/Helmet，不屏蔽真空伤害。

河流扩容 140546 为 19/7 项，250×267 太空扩容 140758 为 17/7 项；均含实际故障注入和新进程读档。太空完整原版往返 141241 为 22/22/11 项，errors 为空，含密闭上层舱温度/真空提交数值检查。各原始结果、截图和失败记录见项目 [扩容验收](../Mods/RimLayers/Docs/LIVE-EXPANSION-VALIDATION.md)。未修改原版 DLL/数据或玩家存档。

确认出发追加验收：141547 已有地表目的地原版流程 26/22/11 项全部通过、errors 为空，包含真实 CheckConfirmSettle 委托，验证目的地 Report 与待确认弹窗不扩容，确认后的动作执行前地图已准备、燃料未改变；未用最终 InitiateTakeoff 拦截冒充普通驾驶台的扣油前保护。

### 2026-10-19：楼梯完工触发上层创建

同一 1.7.4982 rev1280 / MVID a46e6f45b6164261bb63a9471bf7402a。Frame.CompleteConstruction（0600A94F）先发送建成信号并清空销毁资源，再销毁 Frame、Spawn 成品；扩层准备必须前置，不能在 SpawnSetup 中才发现无法扩容而损失材料。MapPostTick（0600145E）在环境/原生管理器 tick 后调用 MapComponentUtility.MapComponentTick，可承接成品 SpawnSetup/读档提出的派生请求。Settlement.ShouldRemoveMapNow（0600E3CC）允许回收没有玩家驻留等阻止条件的 NPC 地图；独立夹具不能把原版回收误认为扩容未执行。

太空合法核心地图原版楼梯完工/直接成品触发 150332 为 18/5 项通过；Core 150815 为 19/6，包含在成品待扩层时保存并由读档重建请求。已有结构生成回归 151024 的 31 项通过；该保存的加载回归发现 SpawnSafety.Recover 抢先移走待坍塌 Pawn，保留坍塌队列的处理权后，同一保存的 151343 加载重放 15 项全通过。原失败与修复证据见 [楼梯按需扩层](../Mods/RimLayers/Docs/STAIR-BOOTSTRAP-VALIDATION.md)。


### 2026-10-19：任意层逆重核心、基础层、屋顶与电梯

同一 1.7.4982 rev1280 / Assembly-CSharp MVID a46e6f45b6164261bb63a9471bf7402a。定向核实 TerrainDef.IsSubstructure（17000250）、TerrainGrid.SetFoundation/RemoveFoundation/TerrainAt（0600137D/0600137E/0600136A）、CompAffectedByFacilities 的候选/几何/链接与活动入口（0600996A/0600995B/06009958/06009972）、CompGravshipFacility 及对应 Link 回调（17001D3C /0600A6F4/F5）、InsideFootprint（0600CB6A）、CompSubstructureFootprint.Valid（17001CB1）、推进器 CanLink/IsBlocked/IsOutdoors（0600A704/A707/A708）。Def 对照 Odyssey Terrain_Foundation.xml、Buildings_Gravship.xml 及 Core Autodoor/Autodoors/MicroelectronicsBasics。核实 CompFlickable 的开关/保存与 CompPowerTrader.ResetPowerVars（170014E3 /06007A84 /06007B05）。

增量与故障入口核实 AutoBuildRoofAreaSetter（02000308）的 queuedGenerateRooms 引用队列、FogGrid.Refog/Unfog（06001223/0600121E）、Map.Dispose（0600146E）及 GenAdj.OccupiedRect（0600048D）。可复用结论归入 [Odyssey 专题](odyssey-gravship.md)。实际候选、存档规则、成功/失败记录与画面证据集中在 [新验收](../Mods/RimLayers/Docs/DECKS-AND-ELEVATORS.md)。原版文件和参考 mod 未修改。

- 2026-10-19 单甲板高层逆重飞船：基准仍为1.7.4982 rev1280 / Assembly-CSharp MVID a46e6f45b6164261bb63a9471bf7402a；定向读取 GravshipPlacementUtility.PlaceGravshipInMap 0600CB4E、GravshipUtility.GenerateGravship 0600CB6D、Gravship 构造0600DCFB及DetermineLaunchDirection 0600DCFF。证据 `Mods/RimLayers/Tests/Artifacts/v1-ship-structure-20261019-171347`；使用契约见mod Docs/DECKS-AND-ELEVATORS.md。

- 2026-10-19 起降动画查看层修复：基准1.7.4982 rev1280，Assembly-CSharp MVID a46e6f45b6164261bb63a9471bf7402a；定向核实GravshipCapturer.BeginGravshipRender/BeginTerrainRender/Update (06008CDB/DC/DE)、GravshipRenderer.BeginCutscene/UpdateTakeoff/UpdateLanding/DrawGravship (06008CE5/E7/E8/EA)、控制器InitiateLanding/DrawGravship/DrawTerrainCurtain (060007D3/DD/DC)、MapDrawer.DrawMapMesh (060014D4)、Section.DrawSection (060014F3)、DynamicDrawManager.ComputeCulledThings (0600119D)、GenGrid.Fit (06003870)。另定向读取UnityStubs.Graphics.DrawMeshCore 06000957（MVID aae7bbfe87c54822ba4fd46cb1a3bd0f）。证据及夹具时序失败见 Mods/RimLayers/Docs/SHIP-CUTSCENE-VALIDATION.md。

- 2026-10-19 发射跨层乘员：基准1.7.4982 rev1280 / MVID a46e6f45b6164261bb63a9471bf7402a；读取StageEndTrigger_RoleArrivedAndSpectatorsOnSubstructure.ArrivedCheck 06009004、StageEndTrigger_RoleTagSet.ArrivedCheck 06008FFE、JobDriver_BoardOrLeaveGravship.GetGotoToil/CheckFailedToBoardOrLeave 06004C8E/90、GravshipUtility.TryFindSpotOn/Off 0600CB7B/7C、RitualBehaviorWorker_GravshipLaunch.CanReachGravship/PawnCanFillRole 060090B8/B9；父配置Data/Odyssey/Defs/Rituals/Ritual_Behaviors.xml。结果与复现入口 Mods/RimLayers/Docs/SHIP-BOARDING-VALIDATION.md。


### 2026-10-19：RimLayers 山体、矿物与竖向开采

基准 1.7.4982 rev1280，Assembly-CSharp MVID a46e6f45b6164261bb63a9471bf7402a。读取 Data/Core/Defs/Misc/RoofDefs/Roofs.xml、ThingDefs_Buildings/Buildings_Natural.xml、MapGeneration/BasePlayerMapGenerator.xml / CommonMapGenerator.xml，以及 WorkGiverDefs/WorkGivers.xml / JobDefs/Jobs_Work.xml 的原版采矿配置。

定向核实 GenStep_RocksFromGrid.Generate / GetResourceBlotchesPer10KCellsForMap / RockDefAt（060076F8 / 060076F9 / 060076F7），RockNoises.Init（06001658），GenStep_ScatterLumpsMineable.Generate / CalculateFinalCount / ChooseThingDef / CanScatterAt / ScatterAt（06007731–06007735），GenStep_Scatterer.GetPlacementFactor / CountFromPer10kCells（060015FF / 06001600），GridShapeMaker.IrregularLump（06007892）。

核实 JobDriver_Mine.MakeNewToils / DoDamage / ResetTicksToPickHit（06004FCD / 06004FCE / 06004FCF），Mineable.PreApplyDamage / DestroyMined / Notify_TookMiningDamage（0600A9A1 / 0600A9A3 / 0600A9A7），WorkGiver_Miner.PotentialWorkThingsGlobal / JobOnThing（0600589B / 0600589D），MineAIUtility.JobOnThing（06005E63）。核实 TerrainGrid.SetTerrain / RemoveTopLayer（0600137A / 0600137C）、FogGrid.Refog（06001223）、PathFinderMapData.GatherData（06001ABC）、PathRequest.ValidateInt（06001AD8）、Pawn_PathFollower.StartPath / PatherTick（06004494 / 06004496）。

实现位于 Mods/RimLayers/Source/RimLayers/Mountains，证据与原始反例见 Mods/RimLayers/Docs/MOUNTAINS.md 和 WORK-LOG.md；未改变原版程序集或玩家配置。


补充核实原版 JobDriver.DriverTickInterval（06004217）直接把 delta 传给当前 Toil 的 tickIntervalAction；不能把回调次数当 tick 数。World.NaturalRockTypesIn（0600DEEE）使用实际 tile 生物群系限制、非资源天然石种列表及隔离随机序列。山体验证中的耗工按实际 delta 与每次HP变化核对。


### 2026-10-19：RimLayers 飞船/普通层共存

基准1.7.4982 rev1280，Assembly-CSharp MVID a46e6f45b6164261bb63a9471bf7402a。定向读取Designator_Build.Visible/CanDesignateCell（1700218A/0600B999）、PlaceWorker_InSubstructureFootprint.AllowsPlacing（0600D15D）、GenConstruct.PlaceBlueprintForBuild（0600CB01）、TerrainGrid.SetFoundation/RemoveFoundation/RemoveGravshipTerrainUnsafe/Notify_FoundationDestroyed（0600137D/7E/7F/89）、GravshipUtility.GetConnectedSubstructure（0600CB69）、Building_GravEngine.UpdateSubstructureIfNeeded（0600AD6D）、CompPilotConsole.StartChoosingDestination_NewTemp（0600A711）、Designator_MoveGravship.ValidGravshipLocation/IsValidCell/DesignateSingleCell（0600B8E7/E8/EA）、GravshipLandingMarker.BeginLanding（0600B03C）、WorldComponent_GravshipController.TakeoffEnded（060007D7）、Gravship构造/CopyCellContents/ShouldBringOnGravship/AddThing/CheckAffectedGoodwill/Roofs（0600DCFB/0600DD03/04/05/07及1700269A）、AreaManager.AllAreas/Home（17000346/347）、Area_Home.Set（06007290）、Designation.ExposeData（0600115F）。原版Def为Data/Odyssey/Defs/TerrainDefs/Terrain_Foundation.xml。

设计先写入 Mods/RimLayers/Docs/SHIP-COLONY-COEXISTENCE.md；实现、真实成功/失败和后续回归见SHIP-COLONY-VALIDATION.md。普通原版Data和程序集未修改。


### 2026-10-20：RimLayers 三维战斗与有人炮台

基准 1.7.4982 rev1280，Assembly-CSharp MVID a46e6f45b6164261bb63a9471bf7402a。定向核实 Verb_LaunchProjectile.TryCastShot（06003AB6）、Verb.TryCastNextBurstShot（06003A6C）、CanHitTargetFrom（06003A79）、ShotReport 字段/掩体入口、Projectile.TickInterval/ImpactSomething（06002A29/06002A30）、ProjectileProperties.arcHeightFactor（04000637）；Building_TurretGun.OrderAttack/Tick/TryStartShootSomething/TryFindNewTarget/TargSearcher（0600AA40/41/43/44/45）及随机建筑闭包（0601440A）；JobDriver_ManTurret（02000A55）、CompMannable.ManForATick（06009FB7）、CompChangeableProjectile.LoadShell（06009BC3）及 Notify_ProjectileLaunched 方法；CastPositionFinder.CastPositionPreference（0600467C）、Targeter.CurrentTargetUnderMouse（0600C452）、GenDraw.DrawLineBetween/DrawTargetHighlight（060037EB/060037F6）。

原版数据入口为 Data/Core/Defs 的 ThingDefs_Buildings/Buildings_Security.xml 中 Sandbags/Barricade、Buildings_Security_Turrets.xml 中 Turret_Mortar 和 Artillery_Mortar、BaseGrenadeProjectile 模板、JobDefs/Jobs_Misc.xml 的 ManTurret/RearmTurret。实施与真实游戏生成/重载证据见 Mods/RimLayers/Docs/COMBAT-3D.md；保存失败及修复过程见 WORK-LOG.md。


同轮补证：CompChangeableProjectile.Projectile（17001A94）、Verb_LaunchProjectile.Projectile（17000B72）、VerbUtility.ProjectileFliesOverhead（06003A95）、VerbProperties.isMortar（040007AF）。RimWorld.StunHandler.Stunned（1700202D）用于实际 EMP 炮弹结果验证；PawnGenerationRequest 构造（0600244B）的固定年龄/基因基础概率仅用于控制隔离诊断中的体型对照。


### 2026-10-20：真实袭击、手榴弹落层与等待射击缺陷

基准 1.7.4982 rev1280，根 Assembly-CSharp，MVID a46e6f45b6164261bb63a9471bf7402a。定向读取 IncidentWorker_RaidEnemy.TryExecuteWorker（0600711C）、IncidentWorker_Raid.TryGenerateRaidInfo/TryExecuteWorker（0600714C/4E）、IncidentParms（0200115A）；PawnGroupMakerUtility.GeneratePawns/GeneratePawnKindsExample/TryGetRandomPawnGroupMaker（06008285/86/87）；JobGiver_AIFightEnemy.TryGiveJob（06005311）、JobDriver_Wait.CheckForAutoAttack（06004156）、AttackTargetFinder.BestShootTargetFromCurrentPosition（06004672）。核实 VerbProperties.forcedMissRadius 为私有字段（040007AC），公开属性为 ForcedMissRadius（170001C9）。

原版 Def：Data/Core/Defs/ThingDefs_Misc/Weapons/RangedIndustrialGrenades.xml，BaseGrenadeProjectile.arcHeightFactor=2，Weapon_GrenadeFrag.range=12.9、forcedMissRadius=1.9，BaseFragGrenadeProjectile.explosionDelay=100；Core/Biotech FactionDefs/Factions_Misc.xml 的海盗编组。真实事故存档复现、修复、二三层实弹和手榴弹重载证据见 Mods/RimLayers/Docs/RAID-COMBAT-VALIDATION.md，失败夹具记录见 WORK-LOG.md。

### 2026-10-20：RimLayers 地下生成与虫灾边界

基准：1.7.4982 rev1280，Assembly-CSharp MVID a46e6f45b6164261bb63a9471bf7402a。定向核实 GenStep_ElevationFertility.Generate（06007637，地下 elevation=1）、GenStep_RocksFromGrid.Generate/RockDefAt（060076F8/060076F7）、RockNoises.Init/Reset（06001658/06001659）、MapGenCavesUtility.Dig/SetCaveAround（0600789F/060078A1）、TileMutatorWorker_UndergroundCave.GeneratePostElevationFertility（06007A4D）、GenStep_ScatterLumpsMineable.CalculateFinalCount/ScatterAt（06007732/06007735）、GenStep_CaveHives.Generate/TrySpawnHive/TryFindHiveSpawnCell（0600762E/2F/30）。

虫灾证据：InfestationCellFinder.GetScoreAt、CalculateLocationCandidates、StraightLineDistToUnroofed、DistToBlocker、NoRoofAroundAndWalkable、GetMountainousnessScoreAt、CalculateTraversalDistancesToUnroofed、CalculateClosedAreaSizeGrid（0600C902/904/906/907/908/909/90A/90B）；CellFinderUtility.CalculateDistanceToColonyBuildingGrid（0600C8E0）计算平面邻接距离，单 Map 坐标带需把邻接限定为同层。保留原生厚岩顶、已揭雾、空间、温度及殖民建筑距离资格；生成 CaveHives 与后续 Infestation 是不同入口。

XML 入口为 Data/Core/Defs/MapGeneration/CommonMapGenerator.xml 的 Underground_RocksFromGrid、BasePlayerMapGenerator.xml 的 CaveHives、TileMutators.xml 的 UndergroundCave，以及 Biomes_Cold.xml 的 IceSheet/SeaIce。Odyssey InsectLairMapGenerator.xml 仅作地下步骤参考，不新增 DLC 依赖。真实失败重放核实 RoughHewn 石面不能只靠 natural/Diggable 分类；需识别原生岩体 BuildingProperties.naturalTerrain/leaveTerrain 及其 smoothedTerrain 关联。

当前布局版本 2 区分稳定存储带 ID 与物理高度；接口、旧档和游戏证据见 Mods/RimLayers/Docs/UNDERGROUND.md、UNDERGROUND-VALIDATION.md。

### 2026-10-20：飞船超高停靠与预览颜色

基准 1.7.4982 rev1280；Assembly-CSharp MVID a46e6f45b6164261bb63a9471bf7402a。定向核对 Designator_MoveGravship.SelectedUpdate / ValidGravshipLocation / DrawMouseAttachments（0600B8ED / 0600B8E7 / 0600B8EC）：占地颜色由 validCells/invalidCells 驱动，错误文字由 AcceptanceReport 驱动，必须分别更新。RimLayers 的实际双层载荷、三处高度拒绝、原生预览红白格与合法停靠回归见 Mods/RimLayers/Docs/SHIP-COLONY-VALIDATION.md；不以只改返回报告作为整船标红证据。


### 2026-10-20：RimLayers 设置与异常实体持久化恢复

基准 1.7.4982 rev1280，根目录 Assembly-CSharp MVID `a46e6f45b6164261bb63a9471bf7402a`。定向核实 Thing.SpawnSetup/DeSpawn（06002A88/06002A8A）、Thing.Position（1700088D）、GenSpawn.Spawn 完整重载（060038B0）、Pawn.DeSpawn/Notify_Teleported（0600231A/06002308）、Pawn_PathFollower.TryRecoverFromUnwalkablePosition（060044A6）、Game.DeinitAndRemoveMap（06000DDB）、Thing.BeingTransportedOnGravship（170008AD）。原版设置窗口 Dialog_ModSettings 构造/DoWindowContents（0600BCDA/0600BCDC）用于真实 UI 截图入口，Prefs.UIScale（17000B0A）用于独立测试缩放。

暂存、对象身份、存读档和地图移除证据：Mods/RimLayers/Tests/Artifacts/v1-settings-recovery-20261020-224405（generate 21 项、load 9 项，均通过）。扩展矩阵仍在进行；完整任务状态与失败证据在 Mods/RimLayers/Docs/SETTINGS-WORK.json，不把该局部通过表述为整项验收。

### 2026-10-20：批量地下生成、活跃任务载物与六层飞船

基准 1.7.4982 rev1280，Assembly-CSharp MVID `a46e6f45b6164261bb63a9471bf7402a`。定向读取 `RegionAndRoomUpdater`（020002F7，Enabled 与重建流程）、`Pawn.DeSpawn`（0600231A）、`Pawn_JobTracker.StopAll/CleanupCurrentJob`（060044FA/060044F4）；复核 `Designator_MoveGravship.ValidGravshipLocation/IsValidCell`（0600B8E7/0600B8E8）。原版引擎/延展器容量来自 `Data/Odyssey/Defs/ThingDefs_Buildings/Buildings_Gravship.xml`。

性能原始证据 `Mods/RimLayers/Tests/Artifacts/v1-settings-perf-20261020-232818`；设置真实进程重启证据 `v1-settings-behavior-20261020-233435`（6/36 项通过）。六层飞船 `v1-gravship-20261020-233107` 生成56、在途加载27项通过，返航11项中旧“强制地表视图”断言失败；随后在 `v1-gravship-20261020-233945` 记录原选层，确认原版到达镜头会先改写 ViewedLayer，导致错误高层预览。补丁现于 Prefix 捕获玩家原选层，在原版到达流程结束后恢复；不是放宽断言。最终复测状态见 `Mods/RimLayers/Docs/SETTINGS-WORK.json`。


### 2026-10-21：暂存存续保护、设置原生输入与预留故障边界

基准 1.7.4982 rev1280，根 Assembly-CSharp MVID `a46e6f45b6164261bb63a9471bf7402a`。定向核对 `WorldPawnGC.GetCriticalPawnReason`（0600E542）、`GameEnder.CheckOrUpdateGameOver`（0600681B）、`PawnUtility.GetPosture`（06008322）、`Pawn_FlightTracker.StartFlying`（0600D237）及 `Widgets.CheckboxLabeled/ToggleInvisibleDraggable/ButtonInvisibleDraggable`（0600332C/2D/4C）。存续正反例见 `v1-settings-recovery-20261021-001625`；床、原生飞行、载体/容器排除和逐实体故障隔离见 `v1-settings-recovery-20261021-002057`。

UnityStubs MVID `aae7bbfe87c54822ba4fd46cb1a3bd0f` 的 `Event.type`（170001E5）在每次读取时还查询 `GuiBackend.Current.MouseOutsideHitClip`。Waylith 诊断注入不能只修改 Event.mousePosition；需要一致的 Input 鼠标状态、滚动坐标和后端命中裁剪输入。RimLayers opt-in 测试适配这些输入后，仍执行原版控件判断及回调；`v1-settings-ui-20261021-002027` 的复选框、页签、滑块和确认恢复默认均通过。注入只属于诊断场景，不是生产 UI 补丁。

飞船 `v1-gravship-20261021-002230` 实际验证配额/太空拒绝发生在原生扣燃料和拆离之前、未起飞失败释放预留、原生起飞开始后异常仍保留在途承诺，以及后续存读档、接收和返回（58/28/11 项通过）。六甲板完整往返另见 `v1-gravship-20261020-234510`；缺失预留记录的旧档新地图接收见 `v1-gravship-20261021-000437`。

最终恢复扩展验收 `v1-settings-recovery-20261021-002902`（45/18）、中文缩放 UI `v1-settings-ui-20261021-002724`（10）和最终大图性能 `v1-settings-perf-20261021-002446`（13）均通过。完整结果、性能原始样本入口和保留的失败记录见 `Mods/RimLayers/Docs/SETTINGS-VALIDATION.md`。


### 2026-10-21：垂直电梯双向扩井的原版施工位置

基准 1.7.4982 rev1280，根 Assembly-CSharp MVID `a46e6f45b6164261bb63a9471bf7402a`。定向核对 `WorkGiver_ConstructDeliverResourcesToBlueprints.JobOnThing`（060056C8）、`GenConstruct.CanTouchTargetFromValidCell/CanConstruct(bool)`（0600CB11/0600CB07）、`RCellFinder.TryFindGoodAdjacentSpotToTouch`（0600C924）。供料与施工要求原生可达邻格，只有出口自身可走还不够；不改全局原版限制，以真实清理施工邻格解决。方法行为见 `docs/multilayer-structure.md`，实际失败及修复证据见 `Mods/RimLayers/Docs/SHAFT-VALIDATION.md`。

### 2026-10-21：VGE 两章飞船机制与 VEF 屋顶补丁

依据用户指定工坊目录 3609835606 / 3799737423 的 1.7 程序集及 Def/Patch XML；未修改工坊副本。Chapter 1 MVID 57be97aa5bc146fbafd0a60f764bf91e；Chapter 2 MVID 05f048ed829d4ceca8a79bd49c74b77c；VEF（2023507013）MVID 9f62edc95eae47eeb742ac995a89b5bd。游戏根程序集 MVID a46e6f45b6164261bb63a9471bf7402a。

定向证据：VGE GravshipHelper 静态构造、CompMultipleGravEnginesHandler、GravshipUtility_GetConnectedSubstructure_Patch、ExtraGravshipDataComp、GravshipUtility_PreLaunchConfirmation_Patch.ExecuteGravshipLaunch、CompResourceThruster / AstrofuelPipeNet / GravshipFuelProviderUtility；第二章 Building_GravEngine_CanLaunch_Patch 与 Designator_MoveGravship_IsValidCell_Patch；VEF.Maps.VanillaExpandedFramework_SectionLayer_LightingOverlay_GenerateLightingOverlay_Patch。结论归入 docs/odyssey-gravship.md；实际验收与已暴露失败记录见 Mods/RimLayers/Docs/VGE-COMPATIBILITY.md。

### 2026-10-21：完整地图 GPU 捕获与换层合成

本次重新加载后的根 Assembly-CSharp MVID 为 `484efdc5765b463ca9cc54a6131e38f0`（1.7.4982 rev1280）；此前同版本记录的 a46e… 不作为本次成员 token 的依据。UnityStubs MVID `aae7bbfe87c54822ba4fd46cb1a3bd0f`，Waylith `c3426a2f113341f2bbd61f8a0d3f8064`，RimWorld.dll 宿主 `266e3ba0aa6f42eb8fc961a4ae260dbb`。

定向成员：宿主 MapRenderFeature.CaptureFrame（060000A5）、CameraTextureBaker.Render/RenderItems（06000093/94）、CameraCaptureFeature.CaptureFrame（0600008A）；Waylith MeshDrawFrame（02000065）；UnityStubs Camera.CopyFrom/Render（06000721/06000762）、Graphics.DrawMeshNow(Matrix4x4)（06000983）、Graphics.TryDrawTextureGpu（06000998）。RimWorld.dll 的后端源确认完整 Draws + PersistentDraws、背景颜色与 SunDirection 路径；实机反例确认显式 Camera.Render 单独捕获漏光照，故地图捕获使用完整帧提交，双纹理合成才用显式 Camera.Render。

输入注入另核实当前 Verse.UI.MouseCell（0600064D）经 UIToMapPosition（0600064A）读取底部原点 MousePositionOnUI，MapToUIPosition（06000648）返回顶部原点；UnityStubs Event.alt（170001EF）是独立属性，并非 modifiers 的计算值。生产按键逻辑未改为测试专用路径。

结论见 assets-localization.md；实际运行目录、保留失败、曲线端点、输入回归及性能数据见 Mods/RimLayers/Docs/TRANSITION-VALIDATION.md。

### 2026-10-21：原版跳跃背包、预热与飞行持有链

基准 Version.txt：1.7.4984 rev1242；显式加载根 Assembly-CSharp.dll，MVID 484efdc5765b463ca9cc54a6131e38f0。未导出整个程序集。

定向入口：RimWorld.Verb_Jump 的 CanHitTargetFrom、ValidateTarget、OrderForceTarget、TryCastShot、DrawHighlight、OnGUI；JumpUtility.DoJump / OrderJump / CanHitTargetFrom / ValidJumpTarget（0600CF1F–22）；PawnFlyer.MakeFlyer（0600B2AA）、RecomputePosition（0600B29B）、SpawnSetup（0600B2A1）、RespawnPawn（0600B2A2）、TickInterval（0600B2A4）、CheckDestination（0600B2A5）、ExposeData（0600B2AB）。Verse.Verb.TryStartCastOn（06003A65）确认预热额外查射线；Toils_Combat.CastVerb、JobDriver_CastJump 确认原生作业入口。

落地队列证据：Pawn_JobTracker.StartJob（060044F0）、EndCurrentJob（060044F3）、CleanupCurrentJob（060044F4）、RestoreCapturedJobs（060044F6）、CheckForJobOverride（060044F9）、ShouldStartJobFromThinkTree（06004501）。拾取重载：Pawn_CarryTracker.TryStartCarry（060024AD / AE）。Def 依据 Data/Royalty/Defs/ThingDefs_Misc/Apparel_Packs.xml、Apparel_Various.xml，以及 Core/Ethereal_Various.xml 的 PawnFlyerBase 父链和 Jobs_Misc.xml 的 CastJump / Wait_MaintainPosture / Goto。

RimLayers 最终游戏入口 Tests/Artifacts/v1-jump-20261021-042941（58/11）、Core 射击回归 v1-combat-shot-20261021-043045（14/9），纯几何 JumpTests 58。新结论在 multilayer-combat.md；完整失败、修复、截图和范围在 Mods/RimLayers/Docs/JUMP-VALIDATION.md。

## 自动化测试流程整理（2026-10-21）

新增 [testing.md](testing.md)，并在 README 和 debugging 添加入口。本轮读取 `Version.txt` 为 `1.7.4984 rev1242`，runtimeconfig 为 `net10.0 / 10.0.13-waylithclr.dev`；未对旧专题做全量版本复核。

直接证据入口：`Mods/RimLayers/Tests/Run-V1.ps1` 的参数、隔离配置、构建、阶段选择、进程等待和结果读取；`Tests/JumpTests/JumpTests.csproj` 与 `Program.cs` 的生产源码链接及失败退出；`Source/RimLayers/Diagnostics/V1Integration.cs` 的 Configure / Update / Check / Finish（专用开关与目录限制、就绪判断、逐项断言、结果字段和退出）。这些是 RimLayers 的自有测试协议，不是游戏为所有 mod 提供的通用测试命令。

发现并在文档标明的现有 runner 边界：`-CheckPure` 的七项清单不含 JumpTests/SettingsTests；游戏非零退出码只打印；未强制报告非空、预期断言集合和实际 mod 列表匹配；未逐阶段自动归档全局 Player.log。本文提出的完整性门槛属于后续 runner 接入建议，本次不修改该 mod 的测试实现。

`-mod` 自动 DLC、savedatafolder 与日志隔离差异、后台更新和 DLL 锁定沿用 debugging 已注明的历史证据；本轮未重新验证这些游戏行为。本文的通用状态机、报告协议与 CI 分层是工程建议。完成文档 UTF-8、链接、代码围栏及 PowerShell 示例语法检查；没有启动游戏或执行 mod 构建/场景测试。
