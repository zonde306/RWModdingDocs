# 单 Map 多楼层：工作查询、连续取料与动态路径审计

核对：2026-10-16，RimWorld 1.7.4980 rev60；主程序集 MVID 8e962ecc7ac742d28f3b1b16cd24d2cd，SHA-256 EFF7D97949F919FB2B151EBFB906E9BE749B63410B7DBDA78EEEBB7972B245C3。

范围：当前原版的定向元数据/方法体，以及 [ProjectZ 源码参考](multilayer-reference-isorim.md) 的相关入口。本文区分源码事实、条件推导和建议接口；没有修改或运行参考 mod。多 Map 的转移与意图协议仍见 [routing](multilayer-routing.md)；本篇专门回答单 Map 是否自动解决选物、家具、工作台、多次取料和动态断路。

## 结论

同 Map 可以让同一 Job 保留最终对象、队列和原版预订，但不会自动修正空间查询。需要共同的坐标/查询契约，以及各个业务入口的适配。只接入 Region 和 PawnPath，仍会漏材料、漏格子工作、选错距离；把所有距离都改成平面距离，又会让不同层错误地发生近距离交互。

选候选、验证可用、求路、执行相邻动作和完成业务必须分别成立。“某 Thing 出现在同 Map 索引中”不能作为整个工作链可用的证据。

## 当前原版的明确过滤点

| 入口 | 1.7.4980 已核实事实 | 实现影响 |
|---|---|---|
| JobGiver_Work.TryIssueJobPackage | 保留工作优先级顺序；Thing 扫描通过 GenClosest，格子扫描自己计算存储距离 | 改 GenClosest 仍会漏格子工作；不可全局重新排序而破坏工作优先级 |
| 普通 scanCells 的 ProcessCell | 初始 closestDistSquared=99999f；非 Prioritized 分支要求距离平方更小，之后才 HasJobOnCell / CanReach | 距离较远的楼层连工作验证都不会进入 |
| GenClosest.ClosestThingReachable | 先 Region 搜索，受区域预算影响；不总进入全局回退 | 加楼梯伙伴不等于一定全图扫描，需尊重具体请求的预算和回退条件 |
| RegionProcessorClosestThingReachable.RegionEntryPredicate / ProcessThing | 区域边界到 root 与 Thing.PositionHeld 的存储距离用于半径/排序 | 只改 Region 连接仍保留错误几何 |
| FoodUtility.BestFoodSourceOnMap | 人形/玩家控制机体分支调用 SpawnedFoodSearchInnerScan，扫描食物全局列表；其他分支有区域数量策略 | 不能笼统说所有 Pawn 食物只搜索固定数量区域 |
| FoodUtility.SpawnedFoodSearchInnerScan | 以存储 Manhattan 距离传给 FoodOptimality，再验证可达、Spawned、饮食等条件 | 跨层食物可被看见却被距离严重扣分，存储带边缘也可能虚假接近 |
| RestUtility.FindBedFor / IsValidBedFor | 先医疗/归属/伴侣等规则，再按床 Def 次序寻找；有效性包含可达、容量预订、禁用与阵营 | OwnedBed 在同 Map 下可直接验证；不要将床简化成所有候选取最近 |
| JoyGiver_InteractBuilding.FindBestGame / CanInteractWith | 候选、集会区和全局可达查找；保留人数预订、禁用、迷雾、社会/政治、电力、环境条件 | 家具适配需保留多人容量与活动范围，不能只改路径 |
| StoreUtility.TryFindBestBetterStoreCellFor | 按储存优先级遍历组 | 跨层距离不能压过储存优先级 |
| StoreUtility.TryFindBestBetterStoreCellForWorker | 以物品 PositionHeld（无生成父对象时用搬运者）到格子的存储距离平方比较 | 同优先级目的地应按真实运输路线比较 |
| StoreUtility.TryFindBestBetterNonSlotGroupStorageFor | 容器也单独做存储距离、容量、在途量、预订与可达检查 | 改仓库格子仍会遗漏容器路线 |
| GenConstruct.CanGetResources_NewTemp | 调用 Map.itemAvailability.ThingsAvailableAnywhere 检查材料数量并按 tick 缓存结果 | “材料总量足够”不代表供料 Job 已找到可达具体堆栈 |

原版数据 Data/Core/Defs/WorkGiverDefs/WorkGivers.xml 的 BuildRoofs / RemoveRoofs 明确 scanCells=true。WorkGiver_BuildRoof 未覆盖 Prioritized，WorkGiver_Scanner 的基类属性返回 false，所以其自动任务也受上述普通格子扫描距离限制。

## 可复算的静态反例

对 250×250 地面，ProjectZ stride=255，地面编号 0：

| 场景 | 存储距离 / 条件 | 结论 |
|---|---|---|
| 地面与第三层相同局部 x/z | dz=510，距离平方=260100 | 大于 99999，普通格子扫描可在可达性之前排除该格 |
| 工作台和材料同局部位置但相隔两层，账单半径 30 | 510² 不小于 30² | 材料被账单物品半径过滤 |
| 地面与第五层同局部位置，账单半径 999 | dz=1020，1020² 不小于 999² | 即使接近最大半径也不能当作跨任意层的真正无限搜索 |
| 地面局部 z=249，第二层局部 z=0，同 x | 存储 dz=255-249=6，实际平面 dz=249 | 存储距离不仅漏掉远层，还会把不同层的远端当成近邻 |

以上数值按当前源码条件用只读 Python 复算，不是运行 RimWorld 的测试结果。ProjectZ 运行源码搜索未发现针对 JobGiver_Work 的格子扫描或 WorkGiver_DoBill 选料距离的专门补丁；不能把原型中搬运/睡床场景当作这些入口已适配的证明。

增加坐标带间隔只能缓解边缘近邻，不能解决跨层候选漏选，而且会增加网格面积。必须区分查询目的。

## 工作台选料的三个独立半径

WorkGiver_DoBill.TryFindBestBillIngredients 将 bill.ingredientSearchRadius 传入 TryFindBestIngredientsHelper。后者至少包含三处不同过滤：

1. 普通材料：baseValidator 检查 (t.Position-billGiver.Position).LengthHorizontalSquared < radius²，另有 Spawned、配方可用性、禁用和 CanReserve。
2. IHaulSource 容器：先对容器 Position.InHorDistOf(billGiver.Position, radius) 过滤，再递归看内容物。仅修正散落材料不会让架子/容器中的材料被搜到。
3. Region 进入条件：非约 999 半径时，使用原始 Region.extentsClose 与工作台存储 x/z 限制搜索。只改材料距离仍可能根本不访问远层 Region。

约 999 的分支仅放宽 Region 进入半径；baseValidator 的 radius² 过滤仍存在。不能把它解释为所有材料都无距离限制。

候选集合形成后，TryFindBestIngredientsInSet_NoMixHelper 用 PositionHeld 到 rootCell 排序；AllowMix 分支先按 IngredientValueGetter.ValuePerUnitOf，再按原始距离排序。适配要保留配方对材料种类、价值、混料和数量的规则，距离只替换相应排序项。手术药品、自动工作台内部材料和未完成物另有路径，不能用普通散落材料查询覆盖所有特殊情况。

## 同一个 DoBill Job 如何多次取料

当前 JobDriver_DoBill 的命名空间是 Verse.AI，不是 RimWorld。可复用的链条已核实：

```text
TryStartNewDoBillJob:
  A = 真实工作台
  B 队列 = 选定的 Thing
  countQueue = 对应数量
  bill = 当前账单

TryMakePreToilReservations:
  预订 A、交互位置；尽可能预订 B 队列

CollectIngredientsToils:
  从 B 队列取下一个目标
  → Goto 材料 / 其生成父对象
  → StartCarryThing（必要时把余量放回队列）
  → 可选顺路补拿同类材料
  → Goto A 的 InteractionCell
  → 找放料位置 / 放入自动工作台
  → 记录 placedThings 与物理交互预订
  → B 还有目标则循环
  → 回工作台执行配方
```

所以“一层拿材料、三层工作台、多次往返”可以由一个原版 Job 表达，不需要为了每个楼梯另造 Travel Job。前提是前面选料完整、所有 Goto 都经过跨层路径、放料/交互检查仍有效。

TryMakePreToilReservations 使用 ReserveAsManyAsPossible，不是所有材料必定成功的原子锁。途中堆栈、禁用、别人抢占或工作台状态变化仍可能让 Job 合法失败。所谓保目标是避免普通过层误终止，不是删除这些业务校验。

JumpToCollectNextIntoHandsForBill 的 64 距离平方条件只是顺路补拿优化：过远目标仍在队列中，通常以后再取，不应因这一优化被跳过就宣称整个 Job 失败。也不能把该条件改为不带楼层的平面 8 格，否则可能无意引入昂贵跨层绕路。

FinishRecipeAndStartStoringProduct 会消费原料、通知迭代完成；按储存设置，可能以 Succeeded 将 DoBill 替换成搬运产品 Job，并保留手持物。这是业务完成后的正常交接，不能为了“一个最终目标”禁止它，也不能在失败恢复时重放已提交的配方消费和产物生成。

## 放料与近距离交互必须留在本层

Toils_JobTransforms.IngredientPlaceCellsInOrder 先枚举工作台 IngredientStackCells，再从 InteractionCell 加原生 GenRadial 前 200 个偏移寻找备选。SetTargetToIngredientPlaceCell 选择这些位置；PlaceHauledThingInCell 的实际放置使用 Direct。

因此要在枚举/选择时过滤：格子有效、属于工作台的实际楼层、不是间隔行、具有有效楼面并允许该物品放置。不能只依赖 ProjectZ 对 GenPlace.Near 的同层过滤，它明确跳过 Direct 模式。边缘坐标带的静态近邻反例表明此类限制有实际必要。

近战、拾取、进食位置、桌椅邻接、材料落点等“物理相邻”不能使用普通平面距离代替。跨层连接只在明确的楼梯/洞口交互协议中授权，不让同一局部 x/z 的不同层直接 Touch。

## 建议共享查询契约

下表名字是拟议接口，不是原版 API：

| 查询 | 建议语义 |
|---|---|
| ToLogical(cell) / ToStorage(level,x,z) | 唯一双向转换，拒绝间隔行、越界和未知楼层 |
| PhysicalTouch(a,b) | 默认同层且满足原版相邻/交互格规则；特殊楼梯触达单独注册 |
| LocalArea(center,radius) | 同层局部范围，不跨坐标带；用于放料、落物、桌椅和局部工作组合 |
| CandidateScope(request) | 显式允许哪些楼层；包含所在组、权限、房间/活动范围和危险策略 |
| LogicalRadius(scope,center,radius) | 明确是本层半径、跨层平面投影半径或路线预算；不混用存储距离 |
| RouteCost(pawn,source,target) | 返回可达性、总路程/耗时及版本；不可达候选不靠近似距离冒充成功 |
| PickByBusinessPriority(...) | 先保留工作、储存、床类型、配方等规则，再比较路线成本；稳定保留已承诺目标 |

默认建议：工作台半径若允许多层，应定义为各允许层对工作台的平面投影范围，并加真实路线可达检查；取料次序按运输成本。它是明确的产品设计建议，不是已获用户指定的半径规则。若需要路线耗时上限，作为另一项设置表达，不能悄悄改变原有半径含义。

查询阶段可以用平面下界与缓存估计排序，再逐步求精。但剪掉的候选必须有正确淘汰依据；不能声称“最近”却固定只尝试前 3 个楼梯。复杂网络先采用端点 Dijkstra，等下界证明完成后再引入 A*。

## 动态断路：缓存失效不足以保护当前路径

ProjectZ StairGrid.Register / Deregister 调 Changed：清理 Reachability 缓存并 Dirty SlabPeel；它没有在此向所有已持有拼接 PawnPath 的 Pawn 撤销跨层步。TakeLanding 也明确保留原本存在的上层楼面，或某些已建物占用的落点。

当前原版的相关行为：

- PatherTick 每 30 hash tick 检查 NeedNewPath；不是每 30 tick 无条件重建路径。
- NeedNewPath 检查目标变化、剩余节点、接下来最多 20 节点的行走性、碰撞、危险、门和斜角阻挡；没有楼梯身份或拓扑代次。
- SetupMoveIntoNextCell 主要检查下一格行走性，随后消费节点。
- TryEnterNextPathCell 在阻挡/碰撞/门处理后写 pawn.Position=nextCell，没有先拒绝普通非相邻格。
- PatherTick 收到 NotFound 时调用 PatherFailed；不能靠“失败之后下次自动派工”承诺保留最终意图。

条件推导：若楼梯被删但原路径跨带两端仍可走，已持有路径可能绕过新图的不可达结果；样本 Links.Kind 失去楼梯后返回 None，并不等于拦截原版进入该格。此为静态风险，未声称已经游戏内复现。

建议给拼接路径挂运行时元数据：routeId、topologyVersion、各非相邻步的 connectorId / 两端 / 通行条件。路径结果认领前校验版本；每次执行非相邻步前，再原子地检查“当前格、下一格、连接器仍存在、方向允许、落点安全、Pawn 能力允许”。只有验证成功才能进入该步。

若不再合法：留在安全的当前格，废弃旧路径，按同一最终 Destination 重新求路；拓扑持续无路时按业务策略有限等待或失败/暂停，并保留需要恢复的高层意图。玩家取消、征召和必要需求仍拥有仲裁权，不可无限锁 Job。不要用 StopDead 之后放空当前 Job 作为隐式等待协议，必须明确后续如何恢复同一 Goto 和失败通知。

## 建议补丁清单与保存范围

| 范围 | 最小适配点 / 原则 |
|---|---|
| 工作 Thing 与家具 | GenClosest 的选中链及所需业务入口；保留 validator、优先级、可达和区域限制 |
| 格子工作 | JobGiver_Work 的 ProcessCell 距离/初始上限；保持优先级边界及 scanner.HasJobOnCell |
| 工作台 | 三处半径、材料排序、未完成物、放料同层、产品入库 |
| 食物与床 | 人形食物评分、床归属与类型规则、原生容量预订；动物/机体另测 |
| 仓库与容器 | 储存优先级、物品到目的地路线、容量/在途/预订；普通格与容器均覆盖 |
| 路径执行 | 请求/结果版本、连接器逐步验证、失败与取消、customizer 参数契约 |
| 读档 | 保存布局/连接器和任务语义；Job 队列由原版唯一保存；路径和 Region 读档重建 |

不要全局 patch IntVec3.LengthHorizontalSquared / InHorDistOf：没有 Map/查询语义，而且会破坏存储网格与物理相邻。若采用 Harmony transpiler，部分检查在编译器生成的 lambda / local function / iterator 内，patch 外层方法不代表已命中真正距离计算；要定位实际成员与匹配数量，版本不符则明确失效。

PathFinder.FindPathNow 有 IPathGridCustomizer 参数；ProjectZ 的跨层 Prefix / StairRouter 调用未转交该参数。未来实现若宣称保留调用者的全部路径约束，必须将其贯穿各层内 leg，或明确对非空 customizer 的不支持策略，不能静默丢弃。

Pawn_PathFollower.ExposeData 保存 moving、nextCell、步进成本、peMode、destination 等，不保存 curPath。TryResumePathingAfterLoading 在 moving 时重新 StartPath；ProjectZ 在 PostMapInit 将非相邻 nextCell 先重置到当前格。该原则可沿用：重建路线，不恢复旧 Region 或瞬时链接对象。DoBill 队列、bill 和 placedThings 由原版 Job/Driver 保存；业务意图层只保存自有所有权数据，避免同一 Job 深拷贝两份。

## 待执行验收与本轮证据范围

本轮已核实入口和计算条件，没有 mod 原型、构建或运行结果。必要场景：三层自动铺顶；不同层食物、OwnedBed、娱乐家具；散料与容器材料；连续多次取料和产品入库；多人争抢；楼梯删去但两端楼面保留；路径已求出后出口封堵；取消与读档；坐标带边缘不跨层错误落物。

方法 metadata 入口（均属上述 MVID）：JobGiver_Work.TryIssueJobPackage 06005636；GenClosest.ClosestThingReachable 060037CB、RegionwiseBFSWorker 060037CD、ClosestThing_Global_Reachable 060037CF；RegionProcessorClosestThingReachable.RegionEntryPredicate 060037D6、ProcessThing 060037D8；WorkGiver_DoBill.TryFindBestIngredientsHelper 060057C7、TryStartNewDoBillJob 060057C1、NoMixHelper 060057CD、AllowMix 060057CE；JobDriver_DoBill.TryMakePreToilReservations 060041A1、MakeNewToils 060041A2、CollectIngredientsToils 060041A3；FoodUtility.BestFoodSourceOnMap 06004A5F、SpawnedFoodSearchInnerScan 06004A63；RestUtility.FindBedFor 06005E87；StoreUtility.TryFindBestBetterStoreCellForWorker 0600564C；Pawn_PathFollower.NeedNewPath 060044B6、PatherTick 06004496、SetupMoveIntoNextCell 060044AB、ExposeData 06004493；IngredientPlaceCellsInOrder 06004283。其他所读成员可按正文完整类型/成员名重新 resolve，不能跨 MVID 复用 token。

后续已完成 [跨层战斗落地审计](multilayer-combat.md)，覆盖射位 AI、近战/爆炸/护盾与投射物持久化的接入；严格屋顶施工时序、失效传播及天空/出口入口已补到 [multilayer-structure.md](multilayer-structure.md)，垂直电力/设施和环境传播见 [公共系统审计](multilayer-services.md)，任务续行/声明与特殊 Pawn 见 [仲裁专题](multilayer-coordination.md)。各专题保留自己的证据范围和未运行状态。


## RimLayers 原型回验：1.7.4981（2026-10-18）

当前主程序集 MVID f1e6382600a84191bbc6d8a184fd6b57。PathRequest.ValidateInt（06001AD8）的 ByPawn 分支调用 pawn.CanReach，验证实际 Pawn.Position，而非 request.Start。单 Map 端点图中的假想层内路径必须在路由器内部子请求作用域改用 request.Start，并保持同一 Pawn、Map、TraverseParms、EndMode；全局跳过验证会放宽真实通行规则。

Pawn_PathFollower.ResetToCurrentPosition（按名解析）会丢弃当前请求/路径并在仍移动时重发同一目的地，适合作为跨层步执行前发现拆梯后的同 Job 重规划入口。Job 是池化对象，任务连续性验收至少捕获 loadID，不能仅比较对象引用。

WorkGiver_DoBill.TryFindBestIngredientsHelper（060057C7）可替换候选收集，同时继续调用原版传入的 foundAllIngredientsAndChoose；这样保留配方、混料和数量算法。高层候选查询仍需分别适配 JobGiver_Work、GenClosest、选料半径、食物评分及仓储成本。当前原型的具体通过场景见 [RimLayers 实施记录](../Mods/RimLayers/Docs/WORK-LOG.md)，不能据此泛称所有工作与需求已适配。

PathFinderMapData.ParameterizeGridJob（06001ABE）将 customizer.GetOffsetGrid 传给 PathGridJob；CostForCell（06001AC9）将 ushort 网格项作为附加成本。跨层边的落点需纳入同类规划成本，而不是把偏好成本变成实际爬梯等待时间。


## RimLayers 运行补证：1.7.4981 rev136（2026-10-18）

当前 MVID f1e6382600a84191bbc6d8a184fd6b57。WorkGiver_DoBill.TryFindBestIngredientsHelper（060057C7）对 IHaulSource 单独递归枚举未 Spawn 的内容物，必须保留来源开关、权限、数量和原版材料选择器。当前程序集实现者为 Building_Bookcase、Building_OutfitStand、Building_KidOutfitStand；普通 Shelf 的物品仍是 Spawned 格内实体。衣架的 IHaulSource.HaulSourceEnabled（17001E2D）读取 allowRemovingItems，SetAllowHauling（0600AAD8）更新开关及原版可搬运索引；衣架本身无 CompForbiddable，而原版 Apparel 有自己的禁止组件。

JobDriver_DoBill.CollectIngredientsToils（060041A3）使用 canGotoSpawnedParent:true、允许从 inventory/holder 取材料的 StartCarryThing；不需要给容器取料另建 Travel Job。其预订入口（060041A1）预订真实工作台、交互格及材料队列。HaulAIUtility.HaulToCellStorageJob（060040EC）保留原版仓储容量与数量设置，JobDriver_HaulToCell（MakeNewToils 060041B9）仍跟随真实 targetA，SplitOff 和最终堆合并由原版完成。

真实四楼梯/非单调、堵出口、移动目标、容器取料及携料跨层中重启均已通过；Job/Driver 与队列、原版预订和数量证据见 [实施记录](../Mods/RimLayers/Docs/WORK-LOG.md) M2 末节。原版自定义成本 PathGridJob.CostForCell（06001AC9）逐格加 custom[index]；实际 ushort 网格改变了跨层连接器选择。一次端点搜索的层内路径临时拥有并释放，不保存跨请求路径缓存，规模性能仍待整合。
