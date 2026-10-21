# Odyssey 逆重飞船：1.7.4982 接入事实

核对：2026-10-19；游戏 `1.7.4982 rev1280`；根目录 Assembly-CSharp MVID `a46e6f45b6164261bb63a9471bf7402a`。以下 token 属于这一基准，不套用旧版编号。

## 先复用原版载荷与生命周期

| 成员 | Token | 已核实行为 |
|---|---|---|
| GravshipUtility.GenerateGravship | 0600CB6D | 构造原版 Gravship；原版 Things/Pawns 经 PreSwapMap 后 DeSpawn；清理原版基底 |
| Gravship(Building_GravEngine) | 0600DCFB | TransferZones、CopyCellContents、CopyAreas 后计算 Bounds；收集作用域与边界计算须分开 |
| Gravship.CopyCellContents | 0600DD03 | 地形取 TerrainAt，可能是临时覆盖；原版记录屋顶、区域、气体、房间温度和真实实体 |
| Gravship.ExposeData | 0600DCFC | things/pawns 字典深保存；不要另外深保存同一 Pawn、Thing 或容器 |
| PrefabUtility.GetAdjustedLocalPosition | 0600734D | 旋转 x/z，保持 IntVec3.y；可在载荷内部用 y 编码相对层号，真实 Map 格仍用 y=0 |
| Gravship.GetPlacementValues | 0600DD23 | 处理非正方形/不可旋转对象的原版锚点修正，并按旋转缓存 |
| Building_GravEngine.CanLaunch | 0600AD66 | 检查基底、驾驶台、燃料、推进范围与冷却；多层功能不应直接把拒绝改成允许 |
| Building_GravEngine.OnValidSubstructure / ValidSubstructureAt | 0600AD69 / 0600AD68 | 附着物按支撑墙，其他对象按全占地；登船警告也会直接检查 cell |
| GravshipUtility.ArriveNewMap / ArriveExistingMap | 0600CB70 / 0600CB6F | 新目的地才生成 Map；已有地图进入落点确认，不应在上层实体生成后才发现缺少存储楼层 |
| GravshipPlacementUtility.PlaceGravshipInMap | 0600CB4E | 基础/地形/气体/区域 → 非 Pawn → Pawn → 屋顶/允许区/电力；自定义楼面与支撑需早于 Pawn/楼梯 SpawnSetup |
| GravshipPlacementUtility.SpawnTerrain(map,root,data,clear) | 0600CB57 | 若当前格不支持地形承载需求，先用 Biome.TerrainForAffordance 回退；临时覆盖会影响承载判断 |
| WorldComponent_GravshipController.WorldComponentOnGUI | 060007E3 | 确认按钮调用 landingMarker.BeginLanding 后立即清空 landingMarker；只直接调用 BeginLanding 的测试必须补齐这一所有权释放 |
| WorldComponent_GravshipController.LandingEnded | 060007E1 | 清除 controller.gravship 与 Current.Game.Gravship，交回原版场景流程 |

`CompPilotConsole.StartChoosingDestination_NewTemp`（0600A711）向 TilePicker 提供原版目的地 validator；实际确认回调会先 ConsumeFuel 再 InitiateTakeoff。新增目的地限制应接在 validator 内、原版接受后执行；validator 保持只读。需要实际准备地图时，应包装 `SettlementProximityGoodwillUtility.CheckConfirmSettle`（0600DC2F）的 settleAction：原版确认完成后、动作中的毁树/ConsumeFuel/InitiateTakeoff 之前准备。不能只在动画入口拒绝，否则可能先扣油。

## 三个容易丢失数据的细节

1. `TransferZones`（0600DD00）只复制候选区域内的格，却删除整个原区域。扩大上层候选时，跨船边的区域应先拒绝或明确拆分。
2. `TerrainGrid.SetUnderTerrain`（0600137B）会直接访问 newTerr.glowRadius，不能传 null。`SetTerrain`（0600137A）设置非可叠地形时本身会清空 underGrid。
3. 上层结构事实应先于原版贴地形建立；完整 top/under/temp 栈应在原版贴地形后、实体生成前恢复。已观察到临时积水使原版承载回退将木地板改成 Soil，不能仅保存 TerrainAt。

## RimLayers 中的实现与证据

实现与验收状态：[GRAVSHIP-IMPLEMENTATION](../Mods/RimLayers/Docs/GRAVSHIP-IMPLEMENTATION.md)。上层元数据只补充布局/结构/表层事实，实体继续由原版唯一保存。已有地图运行时扩容已获用户后续授权，并先通过隔离验证再接入；完整事务与原始证据见 [扩容验收](../Mods/RimLayers/Docs/LIVE-EXPANSION-VALIDATION.md)。不能只修改 Map.Size，须同步持久网格与原生缓冲。

## 现有 Map 扩容的已核实契约

基准同为 1.7.4982 rev1280。`Map.ConstructComponents`（06001456）会重置管理器、预订、区域等，不适用于存活地图转换。保持宽度才能保留一维 z-major 格索引；`HaulDestinationManager.groupGrid` 则为 `[x,y,z]`，不能直接复制一维前缀。

`FloodFiller` 构造（0600378E）将 traversalDistance 清为 -1；FloodFill（06003792）只进入值为 -1 的邻格，ClearVisited（06003794）仅清上次访问的索引。扩展 scratch IntGrid 时使用零初值会让新行被当作已访问。

`Pawn_PathFollower.StopDead`（06004495）释放当前路径并停止移动，但不结束 Job；转换后用原目标和 PathEndMode 重新 StartPath。替换 NativeArray 前须先完成 PathFinder 的 ForceCompleteScheduledJobs（06001A8B）；旧 PathFinder.Dispose（06001AA0）在提交后释放其自有缓冲。

回滚必须保留完整旧区域图及 RegionLinkDatabase，不是浅拷贝 Region[]。MapDrawer.Dispose（060014DB）按当前 Map.Size 遍历 section；旧 drawer 的清理需使用旧尺寸或其真实 section 数组。GlowGridUpdate_First（06001255）同步完成光照 jobs，并会通知 MapDrawer；新 section 必须已初始化。

`ArriveExistingMap` 只将未 Spawn 的 marker 交给 `Notify_LandingAreaConfirmationStarted`（060007E0）；后续回调选择 Designator_MoveGravship。测试实际落地须按设计器提交位置，而不能从 Map 实体列表里提前找 marker。

`Designator_MoveGravship.ValidGravshipLocation` 要求候选船体至少覆盖一格非 Space 地形，否则返回 GravshipMustBeConnectedToLand；这是已有地图重新选址的原版规则。不能为多层支持把完全悬空选址强行判为合法。142017 独立重放记录了该拒绝；使用小平台的 142255 同检查点重放通过。


## 任意层核心、基架与设施（2026-10-19）

基准仍为 1.7.4982 rev1280 / a46e6f45b6164261bb63a9471bf7402a。

- `TerrainDef.IsSubstructure`（17000250）依赖 Odyssey 与 `Substructure` tag，而不是仅比较 `TerrainDefOf.Substructure` 身份。上层专用基础层可保留原生基架承载能力。
- `TerrainGrid.SetFoundation`（0600137D）要求 `isFoundation`，拒绝已有 under terrain，并会移除临时覆盖。`TerrainAt`（0600136A）在 under 为空时返回 foundation，否则返回装修 top。转换旧楼板须局部保留 top/under/temp/color，不能直接在木地板下调用 SetFoundation。
- `GetConnectedSubstructure`（0600CB69）是原生二维 flood fill；`UpdateSubstructureIfNeeded`（0600AD6D）按原生支持量生成核心甲板集合。跨层船体服务应另建原始归属图，不能让该图反过来查询设施生效或 ValidSubstructure，避免延展器递归。
- 原生设施候选 `PotentialThingsToLinkTo`（0600996A）按真实坐标排序并共享 maxSimultaneous。`CanPotentiallyLinkTo_Static`（0600995B）、`CompGravshipFacility.CanBeActive`（17001D3C）、`PlaceWorker_InRangeOfGravEngine.AllowsPlacing`（0600D168）都含距离判定；只改建造 PlaceWorker 不会形成真实设施链接。原生 Notify_LinkAdded/Removed（0600A6F4/F5）维护 comp.engine，应保留链接生命周期。
- `CompGravshipThruster.CanLink` / `IsBlocked` / `IsOutdoors`（0600A704/A707/A708）仍检查排气净空、基架及室外资格，跨层不应直接强制放行。
- `CompPowerTrader.ResetPowerVars`（06007B05）在非飞船运输时会调用 flickable.ResetToOn；电梯的断网/重装需局部保留原开关状态。不能以清除全局电网规则解决。
- `AutoBuildRoofAreaSetter.queuedGenerateRooms` 保存 Room 引用。替换或回滚区域图必须同步处理此派生队列；追加第三层前清空天空列/房间缓存，不能保留两层长度的 bool[]。

RimLayers 当前以最低甲板为版本 2 载荷基准，另存 engineDeckOffset，核心可在任意层。原生二维边界使用逻辑投影；foundation 由原生载荷保存，实体没有第二套所有者。实际接口、迁移、逐层扩容与游戏结果见 [多层飞船与电梯](../Mods/RimLayers/Docs/DECKS-AND-ELEVATORS.md)。


### 单甲板载荷也必须走归一化恢复（2026-10-19）

RimLayers 的托管地图可能只带走高层的一层实际甲板，`layerCount=1` 不能作为“原版无扩展载荷”的判据。`rimLayersShip` 存在即按逻辑载荷恢复；只有实际层数大于1才要求目的地图扩层。最低甲板的 RL_ShipStructuralFloor 在原版投影副本中转换为 Substructure，不更改原始字典/共享 Def。落地保留 top/under/temp，原版唯一持有实体；不把普通支撑建筑带走。真实单甲板捕获和原版 PlaceGravshipInMap 在未扩层60×60地图通过（v1-ship-structure-20261019-171347）。原版 Gravship.DetermineLaunchDirection (0600DCFF) 在无推进方向时读取 pilotConsole.Rotation，隔离夹具也须有真正驾驶台。


## 起降捕获与多层显示（2026-10-19）

同一1.7.4982基准：GravshipCapturer.BeginGravshipRender 06008CDB 以真实核心甲板基架建立捕获范围/子相机；BeginTerrainRender 06008CDC 用于落地前背景。MapDrawer.DrawMapMesh 060014D4 及 DynamicDrawManager.ComputeCulledThings 0600119D 已将 WorldComponent_GravshipController.GravshipRenderBounds 并入范围，多层 section 过滤不能再次排除它。捕获期间不要将其他楼层的背景投影提交到同一捕获子相机。

GravshipRenderer.UpdateTakeoff / UpdateLanding 06008CE7/8 使用takeoffOrLandingPosition及takeoffOrLandingEnginePos；若相机位于另一个存储坐标带，需在纯展示作用域转换中心/特效，并在作用域结束后恢复。控制器 DrawTerrainCurtain 060007DC 也需同样转换提交。GenGrid.Fit(Camera,CellRect,float) 06003870 的第三参是相机y高度，不是额外边距。测试应从GUI阶段启动真实起飞，新建网格尚未呈现时直接在Game.UpdatePlay发起会使捕获夹具产生空贴图，一层对照也会失败。具体候选、原始失败和结果见mod Docs/SHIP-CUTSCENE-VALIDATION.md。


## 发射仪式中的二维查询旁路（2026-10-19）

同一1.7.4982基准。`StageEndTrigger_RoleArrivedAndSpectatorsOnSubstructure.ArrivedCheck`（06009004）在原版角色到岗检查和60tick节奏后，对spectators、pawnsToBoard、pawnsToLeave直接查询 `ValidSubstructure.Contains`，不会调用已适配的 `ValidSubstructureAt`（0600AD68）。`JobDriver_BoardOrLeaveGravship.CheckFailedToBoardOrLeave`（06004C90）也直接读取二维集合，可能将合法高层乘员误从登船名单移除。多层适配应局部替换这些消费者，不全局扩大原版二维基架getter，避免破坏渲染和载荷空间契约。

`TryFindSpotOnGravship` / `TryFindSpotOffGravship`（0600CB7B/7C）使用层内Region BFS；跨层可达服务存在并不代表这两个候选入口会找到跨层位置。保留原成功结果，在失败时补全真实楼面与可达查询。`RitualBehaviorWorker_GravshipLaunch.CanReachGravship`（060090B8）缓存同tick的找登船位结果，`PawnCanFillRole`（060090B9）对普通乘员使用此结果，对驾驶员仍查控制台InteractionCell可达。`IsOnboardGravship`（0600CB79）转调NewTemp（0600CB7A）。验证入口见 mod Docs/SHIP-BOARDING-VALIDATION.md。


## 高层停机坪与连续多船封装（2026-10-19）

同一1.7.4982/MVID基准。Designator_Build.Visible（1700218A）中的 requireInspectedGravEngine 查询的是全局 ResearchManager.gravEngineInspected，不是“当前选中引擎”；建造归属必须另行保存，不能把该字段当船舶身份。PlaceWorker_InSubstructureFootprint.AllowsPlacing（0600D15D）另查自然屋顶与 InsideFootprint，高层普通平台的起始基架可局部替换其范围判定，同时保留原版解锁。

Designator_MoveGravship.ValidGravshipLocation/IsValidCell/DesignateSingleCell（0600B8E7/E8/EA）分别负责投影占地、格合法性和提交真实marker位置；BeginLanding（0600B03C）先销毁marker再 InitiateLanding，拒绝应发生在销毁前。TakeoffEnded（060007D7）无锚会调用 AbandonMap，不因普通楼层留原图就自动保留地图。

RemoveGravshipTerrainUnsafe（0600137F）直接清foundation/under/temp；屋顶RemoveRoofUnsafe回调可能发生在完整源地图清理之前，覆盖地面还原必须等整个捕获清理结束。普通停机坪的完整地形、结构、区域/指令状态由地图保存，不能混进飞船实体或载荷。

Gravship.CheckAffectedGoodwill（0600DD07）开头清理静态 tmpFactionMembersKidnappedCount。归一化时不能把所有 tmp* Dictionary 均设null，只能清实例投影缓存；否则同进程第二艘船封装会在原版友好度处理中空引用。原始失败、修复和隔离证据见mod [共存验收](../Mods/RimLayers/Docs/SHIP-COLONY-VALIDATION.md)。

合法镂空上方的飞船楼板仍需真实屋顶。屋顶位于无楼面的中间格时，需单独保存支撑RoofDef并在实体Spawn前恢复，不把空格当甲板或收集其地形/实体。是否属于船、是否有原生屋顶支撑、是否位于本次载荷，必须分别判断。

## 1.7 停靠预览的红白格与错误报告分别驱动（2026-10-20）

Designator_MoveGravship.SelectedUpdate（0600B8ED）将 ValidGravshipLocation（0600B8E7）填入的 validCells 画为白色、invalidCells 画为红色；DrawMouseAttachments（0600B8EC）才读取 AcceptanceReport 显示原因。只在 Postfix 将报告改成拒绝，不会把已判为合法的占地改红。整船高度等全局约束拒绝时，需要同时清空合法格，并将投影船底占地填入非法格；合法落点保持原版两组列表。

RimLayers 的高度使用物理 Elevation 与完整载荷跨度（含镂空），不使用稳定存储 LayerId 或已分配楼层总数。约束与回归入口见 [共存契约](../Mods/RimLayers/Docs/SHIP-COLONY-COEXISTENCE.md) 和 [验收记录](../Mods/RimLayers/Docs/SHIP-COLONY-VALIDATION.md)。

## 地表轨道停机坪保留原版基础设施规则（1.7.4982，2026-10-20）

`Designator_MoveGravship.ValidGravshipLocation` 要求船底存在非 Space 地形；`IsValidCell` 检查边界、屋顶、迷雾、实体及 Substructure 的地形条件，并不一概拒绝已有 Substructure。多层补丁不能把“上层另一艘船的基架是障碍”扩展为“地表已有轨道停机坪也禁止降落”。地表仍执行原版合法性检查，高层保留独立占用检查。

## VGE 1.7 可选适配边界（2026-10-21）

本机 Chapter 1 的 GravshipHelper.GravEngineDefs 按 Building_GravEngine 类型收集定义，但包含特殊敌方引擎；不要因此无条件把所有子类认作可扩层玩家船。普通引擎以外，VGE_GravjumperEngine / VGE_GravhulkEngine 各有自己的 linkableFacilities：pilot cockpit / pilot bridge 不可与普通 console 混用。

VGE 的 ExecuteGravshipLaunch 会直接 DestroyTrees → ConsumeFuel → InitiateTakeoff；只拦原版控制台/CheckConfirmSettle 会漏掉仪式入口。楼层额度与分离确认必须在该入口扣费前执行。Gravship_CopyCellContents 把彩色真空屏障顶暂存在静态字典，ExtraGravshipDataComp.Initialize 随后接管；多层归一化要同步字典键，getter 还需按旋转/当前放置层投影。

VEF 的 GenerateLightingOverlay 后期 transpiler 查找 RoofGrid.RoofAt(int)。如果另一个补丁先替换该调用，VEF 报找不到目标指令。RimLayers 将逻辑屋顶补丁排在 Harmony owner OskarPotocki.VEF 之后，保留 VEF 对透明屋顶的阴影决策。

Chapter 2 的 IsCellJammed 使用 Verse.IntVec3Utility.DistanceToSquared（静态扩展方法），且保留电力/眩晕判断。楼层存储间距不能算作实际干扰距离；未分配的入站上层同样要检查逻辑 footprint。具体适配与验证见 Mods/RimLayers/Docs/VGE-COMPATIBILITY.md。
