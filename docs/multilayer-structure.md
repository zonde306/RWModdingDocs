# 严格屋顶建造：提交点、结构失效与地图事件隔离

核对：2026-10-16，RimWorld 1.7.4980 rev60，主程序集 MVID 8e962ecc7ac742d28f3b1b16cd24d2cd，SHA-256 EFF7D97949F919FB2B151EBFB906E9BE749B63410B7DBDA78EEEBB7972B245C3。

用户约束：高层只能建在紧邻下层已经完成且有效的屋顶上。本文追查当前原版蓝图→Frame→提交、屋顶变化和坍塌，以及 ProjectZ 的事件隔离参考。建议流程尚未实现、构建或游戏验证；其他基础见 [foundations](multilayer-foundations.md)，跨层路径/工作见 [queries](multilayer-queries.md)，爆炸见 [combat](multilayer-combat.md)。

## 用户规则与样本规则的边界

ProjectZ 的 PlaceWorker_UpperFloor / SlabSupport.Supported 允许在下层“可支持屋顶”而未必已有屋顶的格子建楼板；建成后 FloorRoof.Changed 自动生成 PZ_FloorRoof，Normalize 也可在读档时补顶。它不能直接作为用户严格规则的实现。

建议先采用“已有屋顶允许建设上层结构楼面”的规则，保留 ProjectZ 的楼层坐标、结构楼板和施工方式，但必须增加已有屋顶的前置条件。另一种“屋顶建成便自动得到可走楼面”的方案也可行，二者应由明确产品设计选择，不混用收费和拆除语义。用户只指定必要条件，未要求额外重量模拟或特定 RoofDef 白名单。

固定对齐下，建筑旋转后的实际占地每一格必须满足：

```text
level > 0
⇒ 对应的 level-1 格子在布局内
  && 已有 RoofDef（不是 BuildRoof 区域、规划或蓝图）
  && 屋顶当前仍有效，未标为待失效
  && 该 RoofDef / 山体策略允许使用
  && (本次建设的是结构楼板 || 上层已有有效结构楼面)
  && 工作者有合法工作与落脚位置
  && 原版本层建造条件仍成立
```

上层局部有屋顶，不意味着再上一层完整可用；只建立逐格、逐层依赖。厚岩顶是否可用是附加规则，不能像样本一样默认薄化替换山体；若上方是实际山体，需先处理其占用空间。

## 原版提交链已查清

| 阶段 / 当前成员 | 已核实行为 | 所需检查 |
|---|---|---|
| GenConstruct.CanPlaceBlueprintAt_NewTemp | 建造预览/放置入口；样本也在此接入高度和地面专用限制 | 全占地已有下层屋顶、楼面、高度/本层规则 |
| Blueprint.TryReplaceWithSolidThing | 先检查 FirstBlockingThing，然后 MakeSolidThing，WipeExistingThings、销毁蓝图、Spawn 新物 | 在任何破坏旧物/蓝图前复核结构；失败要返回 createdThing/jobEnded 一致结果 |
| Blueprint_Build.MakeSolidThing | 创建 Frame，复制样式，并把 enroute reservations 从蓝图转交 Frame | 供料目的地已变化，不能继续把旧蓝图当唯一目标 |
| JobDriver_ConstructFinishFrame.MakeNewToils | GotoBuild 后按 delta 增加 workDone，FailOn !GenConstruct.CanConstruct；满足工作量后 CompleteConstruction | 施工中复核；不能只检查任务最初派发时的屋顶 |
| Frame.CompleteConstruction | 先发送完成相关 Quest 信号、销毁资源内容和 Frame，然后 Spawn Building 或 SetFoundation/SetTempTerrain/SetTerrain | 最终提交 guard 必须在方法副作用前 |
| TerrainGrid.DoTerrainChangedEffects | 刷绘制/路径等；按本层 TerrainAffordance 取消不合地形的蓝图/Frame | 不是下层屋顶检查，也不会自动处理所有已建建筑和 Pawn |

关键结论：只在 TerrainGrid.SetTerrain 拦截楼板写入太晚。Frame.CompleteConstruction 在进入地形分支之前，材料与 Frame 已销毁，且可能已发完成信号。必须在最终提交前验证结构，不能先扣材料再发现没有屋顶。

同样，单纯 Prefix 跳过一个 void CompleteConstruction 也不完整：调用者可能继续 ReadyForNextToil 并把任务视作成功。失败需要明确反馈施工 Job，保留/取消 Frame 与材料的策略一致，释放合适的预订，并防止无效 Frame 每 tick 重复尝试。

Blueprint.TryReplaceWithSolidThing 对 Pawn.pather 调 NotifyThingTransformed；跨层任务意图、候选和资源声明应同步跟踪 Blueprint→Frame→建筑对象身份，不能把所有实体转换都当成目标被毁，也不能直接把旧实体引用“复活”。

## 共用校验服务，而非分散补丁各写一套

建议 StructuralSupportService 提供 Placement、Delivery、Work、Commit、EnterCell 等用途明确的检查。校验结果携带失败格、下层/屋顶类型、结构版本与原因，用于 UI 和重新派工；不要只返回无法诊断的 false。

- 预览与自动重建调用相同空间规则，避免手动能造、自动重建悬空。
- 运料之前检查是否仍需该材料、Frame 身份和支撑；到达后复核。
- 施工期间与完成前复核。最后一次检查到提交放在同一确定性执行阶段，期间不让本 mod 的结构队列重入修改依赖。
- 已有屋顶但无楼梯：结构合法、施工暂不可达；不要误报“缺屋顶”或为了供料凭空建路。
- 无屋顶的提前规划只能保存自有计划，依赖完成后才转换为可执行蓝图。
- 神模式、安装/重装、自动重建和第三方直接 Spawn 属于额外入口；公开共用校验 API，并对本 mod 自己的直接创建调用强制执行。不能凭 UI 一个补丁承诺任意外部 mod 都不会越过限制。

结构楼板本身允许从上层已有平台的合法邻格施工，不能要求目标格预先已有楼板而形成循环依赖。当前 Toils_Goto.GotoBuild 先用 TryFindGoodAdjacentSpotToTouch 找邻格，找不到才尝试 OnCell；其 tick 条件检查 Touch 与当前工作格预订。需保证楼梯落脚平台和扩建施工邻格有效，不能把目标空域当作可站工作格。没有合法邻格时属于暂不可施工，不是自动铺实整片空域。

只移除空域地形的 Light/Heavy affordance 不够：原版某些实体无地形承载需求。是否存在有效楼面和下层屋顶，应独立于原版 TerrainAffordance。

## 结构事实、屋顶镜像与保存

建议保存稳定布局与每格结构事实：下层/上层身份、源 RoofDef、结构楼板状态、必要的原屋顶恢复信息、dataVersion。Def 用 Scribe_Defs，原版实体引用用 Scribe_References，自有对象按所有权保存；不 Deep 复制 Map 或 Def。

如果用 PZ_FloorRoof 一类自定义屋顶表示“上方有楼面”，转换前保存被替代的有效屋顶类型与所有权版本；拆楼板不应无条件替换成 RoofConstructed，更不能把天然顶悄悄变成人工顶。若屋顶后来由其他行为替换，不能恢复已经过期的旧快照。

装修层与结构层分离：撤掉地毯/木地板不撤承重；撤结构层显式变为空域。ProjectZ 的结构楼板 layerable=false、显式 Slab.Remove 和从远到近拆除顺序可以参考；不要让 TerrainGrid 的默认 under-terrain 回退产生沙土地面。

RoofChanged 不应自动反向造屋顶来“修复”上层楼面。严格规则下，下层屋顶缺失就是结构失效，应使上层进入不可用/坍塌或显式恢复流程；读档 Normalize 不能偷偷创造已不存在的支撑。保存的布局与实际网格尺寸不匹配时保留原始数据并报告，不能自动填实空洞掩盖损坏。

## 屋顶变化和坍塌有两条路径

当前 RoofGrid.SetRoof 写数组后通知 District、绘制、光照和 MapEvents.RoofChanged；可通过地图组件订阅。TerrainChanged、BuildingSpawned/Despawned、RegionsRoomsChanged、门/路径变化等事件可分别使结构与通路缓存失效，不把所有事件都当成重新扫描整图的理由。

RoofGrid.RemoveRoofUnsafe 仅写数组 null，不通知。当前程序集的直接调用者查询只找到 GravshipUtility.GenerateGravship；已读片段同时移除重力飞船地形、屋顶、区域和气体。这里必须在批量移动边界处理上层依赖，不能假定 RoofChanged 会补通知。第三方反射/直接数组写入不在这次调用者查询范围，生成/读档后的结构一致性检查仍必要。

RoofCollapseCellsFinder.ProcessRoofHolderDespawned 有两种处理：

1. removalMode 且符合条件时直接 SetRoof(null)；
2. 其他情况 MarkToCollapse，随后 RoofCollapseBufferResolver 调 RoofCollapserImmediate.DropRoofInCells 处理缓冲中的格子。

所以仅监听实际落顶动画/缓冲不够，直接移除分支也要触发支撑失效。样本 SlabCollapse 用 finding/removing/resolving 作用域及队列区分两者，这个拆分有参考价值；不能简单把所有 SetRoof(null) 都禁止，否则房屋失去墙后结构会永久悬浮。

RoofDef 当前字段只有 canCollapse、isNatural、isThickRoof、残留物/音效等，没有现成楼层重量/承载能力。用户未要求重量模拟，默认无需再造一套质量求解器。

## 失效事务与跨层连锁

建议状态是 Stable → InvalidPending → Resolving → StableVoid / Recovered，进入 InvalidPending 立即使通行、施工和射线缓存失效。耗时的破坏/坠落可以按稳定顺序排队，但待处理格不能继续当作安全支撑。

队列的键为稳定楼层/格和结构代次，同一代次去重。每次处理先重新读当前事实；拆掉一根墙引起的后续屋顶移除、上层建筑倒塌及更高层支撑变化继续入队，不在事件回调里无限递归修改。

处理顺序建议：

1. 标记失效列与受影响占地，阻止新 Pawn/Job 进入，取消或更新相关声明。
2. 取消/保留自有规划；按策略取消蓝图和 Frame 并安置已有材料。不能重复退款。
3. 多格建筑以 Thing 身份去重，整体处理一次；先收集受影响列表，再执行破坏，避免遍历时修改原集合。
4. 计算 Pawn、携带 Pawn、物品和残留物的最近合法下落/安全安置位置。通过逻辑列寻找第一个实体支撑面，不能用存储 z 减 1 当作下一层。
5. 对 Pawn 显式结束/恢复受影响 Job、路径和预订；事故伤害和正常拆除挪开分开。携带物持有关系不能重复 Spawn。
6. 失效传播到更高层后继续按队列处理，批量刷新地图网格与几何/拓扑版本；保存时保留尚未提交的必要状态，读档重建队列。

正常拆除可要求先清空上层对象，另提供安全施工顺序；敌袭、承重墙破坏与脚本改顶仍必须有事故流程。完整坠落伤害、物品损坏和是否允许坍塌重击下一层是产品策略，不能把样本的固定数值当作用户已选规则。

## 天空落点与离图边界必须使用逻辑列

单 Map 中，游戏原生 WholeMap、AllCells、随机格和最北边界都指整个扩大的存储矩形。ProjectZ 的 CellFinderPatches 将部分随机/地图边界查询限制为地面，ExclusionPatches 修正 ExitMapGrid 与 Region.touchesMapEdge，并为 DropCellFinder 单独开放可用的上层落点。

当前原版 ExitMapGrid.Rebuild 遍历 CellRect.WholeMap；不适配会把扩图最北端当成离图边界。建议仅地面逻辑边缘连接世界；上层 Pawn 逃跑、撤退、俘虏押送和动物离图必须通过楼梯到地面出口，不能在上层坐标带边缘消失。

DropCellFinder.CanPhysicallyDropInto 只检查当前存储格的可走、真空/水、该格屋顶及阻挡物，没有从天空穿过所有楼层的判断。Skyfaller.HitRoof 也只处理其落点 OccupiedRect 附近的屋顶；Impact 可能先爆炸再 SpawnThings，内容物 Near 放置后清空容器。

因此天空事件需要 ColumnTraceFromSky：对实际占地逐列从最高实体面向下检测，选择合法命中/落点，决定能否穿普通薄顶、能否破坏结构板及碰撞中止点。不能只检查落点格是否有屋顶。

ProjectZ DropRules.Blocked 主要检查该格有效楼面和 FloorRoof.Kept（直接对应的上方楼面/楼梯井），RandomDropSpot 在各层抽样；这不能自动证明所有上方实体面都已排除。例如中间层该列为空洞、但更高层存在由周边承托的屋顶/楼板时，应按整列最先碰到的实体面裁定，而非只看紧邻一层。

多格 shuttle/陨石还需全部占地一致，不可有一部分在层 A、一部分误落到层 B。事件选点、飞行过程中结构变化、命中爆炸、内容物放置和“找不到合法落点”的回退要用同一规则，不能回退到任意 AllCells 随机格。

## 环境与世界系统的明确边界

| 系统 | 单 Map 默认事实 / 需要的明确规则 |
|---|---|
| Map / storyteller / 财富 | 一个 Map 的原生殖民地与统计不因查看层复制；不要每层再加一次同一 Map 的总财富 |
| 房间与温度 | 查询期楼梯连接不永久合并 Room；垂直热交换单独处理，ProjectZ FloorHeat 可参考 |
| 雨雪、天空、照明 | 自己的屋顶与所有更高结构面不是同一个概念；室外露台、楼下室内、天井需用列规则解释 |
| 火焰、气体、声音/惊扰 | 原版二维邻接不自然跨层，也可能在坐标带边缘发生伪邻近；按作用类型选同层或竖向连接 |
| 电力/管网/设施 | 同 Map 不意味着不同存储带电网自动相连；垂直连接必须显式建模，不能只修正寻路 |
| 地面专用建筑 | 地质扫描/深钻/坟墓/桥/飞船基础等需明确限制或专门适配；样本提供 GroundOnlyDef 列表 |
| 任务/Lord/动物/囚犯 | 身份在同 Map 保持，但活动范围、聚集和逃跑距离仍可能用存储坐标，需要按支持列表逐项验证 |
| 地图生成与存档 | 新图扩容、旧图不随设置随意 Resize；布局版本、网格尺寸和楼层映射属于保存契约 |

环境与电网这一表是设计边界；后续 [公共系统专题](multilayer-services.md) 已定向核实电网、设施、气体、温度、火、天气与光照的关键入口，[仲裁专题](multilayer-coordination.md) 核实预订、ThinkTree、牵绳/机体/Lord。它们各自记录事实和建议，不宣称穷尽第三方调用链或已运行验证。

## 证据、验收与尚余调查

方法入口（当前 MVID）：Frame.CompleteConstruction 0600A94F；Blueprint.TryReplaceWithSolidThing 0600A8CF、Blueprint_Build.MakeSolidThing 0600A8E8；JobDriver_ConstructFinishFrame.MakeNewToils 06004C19；TerrainGrid.DoTerrainChangedEffects 06001385；RoofGrid.SetRoof 0600180D / RemoveRoofUnsafe 0600180E；RoofCollapseCellsFinder.ProcessRoofHolderDespawned 060017EE / RemoveBulkCollapsingRoofs 060017F0；RoofCollapseBufferResolver.CollapseRoofsMarkedToCollapse 060017EC；GravshipUtility.GenerateGravship 0600CB6D（读取相关批量移除片段与直接调用者结果）；DropCellFinder.CanPhysicallyDropInto 0600C8FE；Skyfaller.HitRoof 0600B3AB / SpawnThings 0600B3AC / Impact 0600B3AD；ExitMapGrid.Rebuild 060012BD。RoofDef / MapEvents 元数据完整列举相关字段/事件；参考源码为 ProjectZ Slabs.cs、FloorRoof.cs、Stairs.cs、HeightRule.cs、GroundOnly.cs、CellFinderPatches.cs、ExclusionPatches.cs 与对应 Def。

待执行场景：部分屋顶下的旋转多格建筑；最后一个施工 tick 前拆顶；蓝图转 Frame 后断路；已供料 Frame 支撑失效；屋顶直接移除与延迟坍塌；多格物体/携带 Pawn 的连锁坠落；装修撤除不生假地面；天空物体遇到洞口和多重屋顶；上层逃跑经地面出口；保存失效队列后重启。它们是原型验收清单，不是本轮运行结果。

本轮已补齐严格屋顶规则的最终提交点与失效传播设计；已读事件入口足以说明为何必须有整列天空几何和地面出口规则。公共系统的垂直电力/设施、环境传播及其隔离与保存已补到 [services](multilayer-services.md)。静态调研收口见 [readiness](multilayer-readiness.md)；性能和第三方组合兼容需要原型之后实测，不能靠源码保证。


## RimLayers 原型回验：1.7.4981（2026-10-18）

当前 MVID f1e6382600a84191bbc6d8a184fd6b57。Frame.CompleteConstruction（0600A94F）仍先销毁材料和 Frame，再创建建筑/地形；JobDriver_ConstructFinishFrame.MakeNewToils（06004C19）的施工回调在 CompleteConstruction 后继续 ReadyForNextToil。提交入口 guard 与施工回调的失败反馈须共同实现。

RoofGrid.RemoveRoofUnsafe（0600180E）只执行 roofGrid[index]=null，不调用 SetRoof 通知；结构失效必须覆盖这一入口。TerrainGrid 的原生寻路成本更新早于自有地形 postfix 事实登记，新增有效楼板事实后需再次刷新相应 PathGrid，不能只修改 Walkable 查询返回值。

原版 GenLeaving.GetBuildingResourcesLeaveCalculator（0600CB31）对 Frame 的 Cancel 返回其资源容器实有材料；已建建筑 KillFinalize 使用原版事故残骸比例。不能按完整配方给未供满 Frame 退款。事故中断后的原版 Job 清理可能把携带物的同一实例放到落点，这是所有权转移，不应额外复制/Spawn 一份。

对应真实原型验收包含最后施工 tick 拆顶、材料/预订保护、邻格楼板扩建、直接移顶全列失效，以及保存待坍塌状态后新进程继续并只退款一次；范围和证据见 [实施记录](../Mods/RimLayers/Docs/WORK-LOG.md)。正常拆除、天空落物、完整结构边界仍需后续验收。


### 当前天空落物调用方与失败路径（1.7.4981）

2026-10-18 对当前 MVID f1e6382600a84191bbc6d8a184fd6b57 重核：Skyfaller.Tick（0600B3AA）剩余 15 tick HitRoof，0 tick Impact；反向离图分支另行处理。DropPodIncoming/PassengerShuttleIncoming/LavaRockIncoming 的 Impact 覆盖均有额外副作用，ShuttleIncoming/DropPodIncoming 的 SpawnThings 也有分支；同图 Skyfaller_FlyingPawn 不应被当作外来天空落物。

GenSpawn.Spawn 的完整 Thing 重载（060038B0）先按 loc 执行 WipeExistingThings，再设置 Rotation/Position、移出 holdingOwner 并 SpawnSetup。落点重定向必须早于擦除旧物，不能在 SpawnSetup 后才修改坐标。

Skyfaller.SpawnThings（0600B3AC）没有检查 GenPlace.TryPlaceThing 的 bool；Impact（0600B3AD）随后 ClearAndDestroyContents，Destroy（0600B3A6）也清空持有物。因此只给放置增加有效楼面 validator 不够：找不到位置时会销毁尚未落地的实体。1.7 类型是 ActiveTransporter，不是旧版 ActiveDropPod；其 PodOpen（0600B35D）同样在放置循环后清空容器，且可按配置提前 DeSpawn，Tick（0600B35B）用 age/openDelay 开舱。

Skyfaller.ExposeData（0600B3A3）已保存内容物、倒计时和 hasHitRoof/hasImpacted/hasLeftMap。完整实现需统一整列选择、多格占地、空域放置资格与失败持有/恢复事务，保留原生阶段事实，避免重复爆炸或内容物丢失。该调查之后 RimLayers 已实现天空整列路由和失败持有事务，当前运行证据见后节及 [实施记录](../Mods/RimLayers/Docs/WORK-LOG.md)。


### RimLayers 天空落物事务回验（2026-10-18）

SkyfallService 按来袭物及直接内容物占地选择统一的最高接触楼面，生成擦除之前重定向，原生 Tick 前复核；反向离图和同图 FlyingPawn 不重定向。CanPhysicallyDropInto 与 RandomDropSpot（0600C8EF）的两个 RandomCellWith 回退分别接入；后者不会调用前者，单补物理检查仍会漏掉非站立模式的隔离区候选。

GenPlace.PlaceSpotQualityAt（060038A5）可以把会擦除建筑的格当 Awful，TryFindPlaceSpotNear（060038A3）最终仍接受 quality>0。安全 Near 因而必须独立排除建筑/Frame 覆盖及不可站立 Pawn/物品落点。GenSpawn.CheckMoveItemsAside（060038B5）先 DeSpawn 旧物品，放置失败则 Destroy；mod 采用原生近处候选做完整挪位计划，成功后移动同一物品，不使用删除旧货物的失败兜底。

Core Things_Special.xml 的 ActiveDropPod.category 是 Item。恢复失败开舱的舱壳时，原生 GenSpawn 容量处理会再次挤出已卸货物；只对这一原实例恢复调用抑制该分支，不修改全局游戏状态。已经在实墙中 Spawn 的舱壳还会在读档 Wall 恢复时被原生 SpawningWipes（060038B8）的 Item 规则擦除；生产路径应保持整个舱壳在来袭物 holder 中，直到落点可站立，而不是把存档保护放到擦除之后。

原版 Skyfaller.hasImpacted/hasHitRoof/倒计时和 ActiveTransporter.age/contents 是唯一阶段事实；未成功放置的内容物不清空，已撞击重试不再 Impact。穿梭机 SpawnThings 的蓝图清理改为准备提交时执行，失败重试不预先取消蓝图；Frame 退款仍交给原生 Cancel。Native Notify_TerrainDestroyed（06001388）先检查 CanRemoveTopLayerAt，不能把它当成总会剥掉结构楼板的操作。

真实 Core 与全 DLC 覆盖在途/已撞击/部分卸货重启、满格原货物、实墙乘客、完整 5×3 穿梭机及 Frame 材料保护；具体输入、通过数、失败反例与范围见 [实施记录](../Mods/RimLayers/Docs/WORK-LOG.md) M3 末节。天然岩顶和第三方落物框架仍按原计划另行适配；首版不模拟重量或贯穿结构楼板。


## 2026-10-19：山体天然楼板与矿脉（1.7.4982）

用户已确定：薄顶岩只提供人工楼板支撑，不自带楼面、不填岩；厚顶岩轮廓向上延伸，每个新增层有不可拆天然楼板、真实可采岩体和独立矿脉。天然楼板单独保存，不因下层塌顶移除。原版厚岩顶的支撑和塌顶伤害保留，开采只移除体积占用，不挖穿下一层。

山侧可以通过同层人工平台直接进入：原版 Mine/Touch 任务站在真实可达邻格开采，目标岩体自身不要求可行走。内部楼梯和单层电梯在 Frame 完工前准备上一层；出口有 Mineable 时保留框架和材料，由采矿工作从下层挖通，再交给建造完成。不允许 Spawn 成品免费清岩。

矿物参数来自实际地图的 GenStep_RocksFromGrid，选择沿用 GenStep_ScatterLumpsMineable.ChooseThingDef，形状沿用 GridShapeMaker.IrregularLump；必须按逻辑单层面积计算数量，并限制矿簇全部格在本层山体内。原版 CountFromPer10kCells 假设方形、直接使用 mapSize²，不能拿扩容 Map 的面积当单层面积。天然岩石的 isNaturalRock 对矿物也可能为 true，识别矿物应看 isResourceRock，不能用 !isNaturalRock。

原版 Mineable 采空后可能将石面变为 RoughHewn，装修拆除应恢复实际铺装前石面，而非强制改回 building.naturalTerrain。竖向开采复用 JobDriver_Mine.DoDamage，并保留 Notify_TookMiningDamage/DestroyMined 的实际收益累计；只改变接近位置。

PathFinderMapData.GatherData 每 tick 最多采样一次。隔离测试在同一回调里大量造景、故障回滚、追加道路后，应结束造景批次再下达任务，避免读取同 tick 的旧原生网格；不能据此替换正常游戏寻路或改成传送。

本轮实现及有效运行记录见 [山体使用与验收](../Mods/RimLayers/Docs/MOUNTAINS.md)。既有上层只通过显式预览迁移，矿脉方案和逐格已处理状态保存，重复执行不能刷新已采矿物。


## 2026-10-21：新站出口还需要真实施工邻格（1.7.4982）

当前 MVID `a46e6f45b6164261bb63a9471bf7402a`。`GenConstruct.CanTouchTargetFromValidCell`（0600CB11）直接调用 `RCellFinder.TryFindGoodAdjacentSpotToTouch`（0600C924）；后者枚举 `GenAdj.CellsAdjacent8Way(touchee)`，两次邻格查找均失败时返回 false，不把目标格自身作为合法施工邻格。`GenConstruct.CanConstruct` 的 bool 重载（0600CB07）和 `WorkGiver_ConstructDeliverResourcesToBlueprints.JobOnThing`（060056C8）均在派工前要求此条件。

因此只打通一个可走出口，即使跨层 CanReach 为 true，原版供料/建造也可能不派工。RimLayers 垂直井扩展保留一个相邻施工位：地下出口与施工位均开采真实岩体；人工上层逐格检查已有下层屋顶。新站继续走原版蓝图、材料、Frame 和完工流程，不通过放宽全局施工邻格规则解决。实现、实际反例和游戏回验见 [垂直井验收](../Mods/RimLayers/Docs/SHAFT-VALIDATION.md)。
