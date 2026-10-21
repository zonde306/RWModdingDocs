# 多层框架落地调研：转移、地图生命周期与结构楼面

核对：2026-10-16，RimWorld 1.7.4979 rev109。主程序集 SHA-256 A97A01C4319414414C0CA39F3565215E83300DC608DC94B69A27720095B4E751，MVID 425a593954d7432a8f44e3076b25fce9。依据为当时根目录 DLL 和指定原版 XML；未导出整程序集。

这是 [总体架构](multilayer.md) 与 [连续寻路协议](multilayer-routing.md) 的落地补充。以下区分已核实事实、推导与建议。未创建 mod、未编译原型、未运行游戏；读写档、携物、异步路径、DLC 关闭与兼容性均未通过运行验证。

当前 1.7.4980 的 [严格屋顶施工与事件隔离](multilayer-structure.md) 已补齐完工提交、直接删顶/坍塌两分支、整列天空检测和地面出口规则。本篇保留历史基础证据，不把这次审计解释为已实现原型。

## 后续参考与版本变化

后续已调查 [Baal.SecondFloor 本地实现](multilayer-reference-baal.md)，记录其不同的保留 Driver 转移方式、自定义 PocketMap 父对象、屋顶铺地板策略和财富风险。当前游戏已更新为 1.7.4980；本篇原始方法证据为 1.7.4979，有限重核结果见参考专题及 [sources.md](sources.md)。不把参考 mod 的存在当作本篇 P0 已通过。

新增 [IsoRim ProjectZ / IsoRim Waylith 参考](multilayer-reference-isorim.md)：同 Map 坐标带的过层不调用 DeSpawn/Spawn，因此本篇转移协议不适用于它的普通上下楼。空域、结构楼板、屋顶生命周期和验收原则仍有参考价值；其实际生成策略为一次扩图预分配，另需审计临时 Map.Size、网格修复与存储距离。本篇后文继续作为多 Map 设计分支。

## 本轮结论

1. 普通携物过层可以先验证“使用原 Pawn 与原 carryTracker，不另复制运输容器”的实现。离图不移除 carryTracker，但 Job 结束/开始可能主动放下物品；必须管住两端交接。
2. 过层会销毁并重建 pather 和 drafter。保留旧 PathFollower/征召控制器实例不可行；恢复征召后再建立新 Job。
3. GameComponentTick 位于实体 TickList 和全部 MapPostTick 之后、AutosaverTick 之前，可作为批量转移队列的首选原型时机。它不是异步寻路任务全部结束的证明。
4. 自定义楼层需要专用父对象/生成配置。原版 PocketMap 工厂固定使用原版父对象 Def；sourceMap 还参与财富汇总和地图销毁，不能当成任意邻接指针。
5. 原版已递归统计子 PocketMap 财富，再自行将各层财富相加可能重复；sourceMap 关系必须无环。
6. 空洞必须是真正不可通行地形，并与结构板和饰面分开。直接在不可通行地形上铺 layerable 地板可能生成 Sand 底层，拆除后会出现幽灵实地。
7. 屋顶事件存在原版批量旁路：当前主程序集直接调用 RemoveRoofUnsafe 的是 GravshipUtility.GenerateGravship。飞艇起飞属于必须处理或明确禁止的结构变化。

## 一、携带物、征召和 Job 的真实生命周期

### 已核实的调用链

| 成员 | 当前行为 |
|---|---|
| Pawn.DeSpawn | 有 curJob 时调用 jobs.StopAll；正常 Thing 离图后停止 pather，处理牵绳、思想、机械师等通知，清除各图预订，注销 mapPawns，再 RemoveComponentsOnDespawned。 |
| PawnComponentsUtility.RemoveComponentsOnDespawned | 将 rotationTracker、pather、natives、filth、roping、flight、caller、interactions、drafter 设为 null；此方法未移除 carryTracker、inventory、equipment、jobs。 |
| PawnComponentsUtility.AddComponentsForSpawn / AddAndRemoveDynamicComponents | 按种族/派系/DLC 等重建需要的组件，包括新的 PathFollower 和适用 Pawn 的 DraftController；不是恢复旧实例。 |
| Pawn.SpawnSetup | 正常注册地图并重建组件；非读档生成会重置寻路位置，还触发战争迷雾、游戏结束检查、人口记录等通知；有后续 ExecuteWhenFinished 回调。 |
| Pawn_CarryTracker.ExposeData | 通过 Scribe_Deep 保存自有 innerContainer；携物由其真实 ThingOwner 所有。 |
| Pawn_CarryTracker.TryStartCarry(item,count,reserve) | 限制容量和数量，用 SplitOff 后加入 innerContainer；可预订并更新资源计数。 |
| Pawn_JobTracker.CleanupCurrentJob | carryThingAfterJobOverride 未给出时使用 job.def.carryThingAfterJob；若不保留携物，TryDropCarriedThing。 |
| Pawn_JobTracker.StartJob（前轮已读） | keepCarryingThingOverride 与新 JobDef.dropThingBeforeJob 决定新 Job 开始前是否放下携物。 |
| Pawn_DraftController.Drafted setter | 改变状态会清优先工作及队列，将 FireAtWill 重设 true；在条件满足时结束/替换当前 Job，并处理 Lord、商队、受控动物等通知。 |
| Pawn_DraftController.FireAtWill setter | 关闭自由开火时可取消当前瞄准姿态。 |
| Thing.DeSpawnOrDeselect | 调 DeSpawn 或取消选择，并返回原先是否选中；不会把旧 drafter/pather 保存下来。 |
| Pawn_PathFollower.StopDead | Dispose 当前请求与路径、停止移动、重置当前格及移动状态。 |

由此可作出的有限推导：普通物品或被携 Pawn 若仍属于 carryTracker.innerContainer，可随原 Pawn 所有权链保留；不能仅因 DeSpawn 清组件就认定携带容器必然丢失。但 Job 清理、后续通知和第三方补丁仍可能改变携物，这不是已运行通过的结论。

### 建议的最小实现

使用本 mod 的交接/Travel Job，明确 carryThingAfterJob=true、dropThingBeforeJob=false；在调用已核实的 Job 交接入口时按需使用 keepCarryingThingOverride。所有正常、失败及取消分支都检查真实 CarriedThing、持有者与数量，而不是仅保存目标 Def。

物品继续由原 Pawn 持有；地图间移动的是原 Pawn 本体，禁止复制 Pawn/Thing 或直接改 mapIndex。只有持续多 tick 的电梯舱、无法入图后的独立安置等确需独立所有者的阶段，才引入可序列化 ThingOwner，并定义唯一所有权和恢复协议。

一次转移前记录稳定的任务/阶段/代次、Pawn 引用、源/目标 Map、落点、旋转、征召状态、FireAtWill、期望携带物/数量；不得长期保存 pather、drafter 或旧地图临时对象引用。

入图后先恢复适用 Pawn 的 Drafted，再恢复 FireAtWill，最后派发下一段 Job。顺序来自 Drafted setter 的清队列/打断行为；先启动下一 Job 再恢复征召会再次干扰任务。不能无条件给敌人、动物或不支持征召的 Pawn 创建控制器。

不要用 respawningAfterLoad=true 伪装普通转移以跳过通知。SpawnSetup 的人口/思想/迷雾/组件副作用需要分别测试；是否需最小范围修正，取决于实际不良结果和证据，不整体绕过基类。

## 二、在何处执行转移

### 当前 1.7 时序证据

TickManager.DoSingleTick 的相关顺序：

MapPreTick（每图） → Normal/Rare/Long TickList → 世界与事件系统 → WorldPostTick → MapPostTick（每图） → GameComponentUtility.GameComponentTick → LetterStackTick → AutosaverTick。

MapPostTick 内包含各本层系统及 MapComponentTick。因此在某张图的 MapComponentTick 中转移到另一张尚未 PostTick 的地图，仍会处在地图遍历内部；跨图队列由会话级组件统一收口更容易保证顺序。

MapPreTick 调 PathFinder.PathFinderTick。后者先 ForceCompleteScheduledJobs，再为本轮工作调度新的网格与路径 Job。所以“已经到 GameComponentTick”不代表没有异步任务；不能在该阶段随意释放路径网格/地图内存或后台改地图。

建议：Driver 到连接器后只进入本 mod 的待转移状态并入队，保留一个合法的本层执行状态；GameComponentTick 有界处理队列。执行时再次核对 ActiveTask 代次、当前本层 Job、地图、连接器、持有物和落点。若玩家/危险已抢占，丢弃旧请求。

在 GameComponentTick 中完成离图、入图、恢复控制状态和安全的新阶段派发，避开旧 Driver 的 tickAction 调用栈；不把 curJob 清空后再留一整个 Pawn tick 等待补发。是否直接 StartJob 或经预先接入的 ThinkTree 继续仲裁，应在小原型中选定并验证。

转移上下文只覆盖这一笔正常离图/入图，用来解释 StopAll/InterruptForced，不能吞掉同阶段发生的真正外部取消。对清理回调/其他 mod 的重入需要检查 task generation，不把一个全局布尔值作为所有 Pawn 的豁免。

若先 EndCurrentJob(startNewJob:false) 再 DeSpawn，注意 DeSpawn 仅在 curJob 非空时 StopAll；不能因此漏清旧 JobQueue。首个原型可让专用 Travel Job 保持当前状态，再由正常 DeSpawn 执行其清理，所有后续远程命令保存在自有意图队列。

同一次同步转移在自动存档入口前结束，是由当前 tick 顺序支持的设计；它不是对所有手动/第三方保存入口的原子性保证。持久状态仍需能识别准备、已入目标图和已提交，恢复时以真实 Pawn.Map/所有权为准。

## 三、落点和失败不能只靠 Spawn 返回值

GenSpawn.Spawn(Thing,...,WipeMode,respawningAfterLoad,forbidLeavings)：

- 默认 WipeMode.Vanish；相应分支会 WipeExistingThings，其他模式可能退款或移动物品。
- null Map、越界、占地越界等可能返回 null；already Spawned 时可直接返回原对象。
- 若有 holdingOwner，会先从持有者移除，再调用 SpawnSetup。
- 返回 newThing 不等于完整事务成功；SpawnSetup 可能因 Pawn 状态错误而销毁实体。

CanSpawnAt 可帮助检查原版生成约束，但不替代本 mod 的楼板、连接器权限、危险、落点容量/对向冲突与期望地图检查。普通站立检查也不能证明不会产生不允许的擦除。

建议在离图之前验证整个落脚占地与其结构支撑、目标图注册状态、排队声明和出口可用性；对会发生 Wipe 的情形重新选点或等待，不把 Spawn 当作找落点的方法。不要假设有一个未经核对的“绝不擦除”枚举值。

提交条件至少是：原 Pawn 未 Destroyed、Spawned、Map 等于目标图、位置合法且持有物一致。否则先检查实际所有权/生成状态，再按明确回退协议处置；不能因为 Spawn 返回 null 就直接再次 Spawn，导致双重登记。

源 Map 消失、目标过期或回退无合法格时，需要有持久安置方案并记录诊断。P0 首先限定在两个已加载、不会移除的测试 Map；不能以此原型通过代表地图销毁情境也安全。

## 四、楼层 Map 的创建、保存、身份和回收

### 已核实事实

| 入口 | 结果 |
|---|---|
| MapGenerator.GenerateMap | 一个 MapParent 不能已有另一张 Map；重复时记录错误并将本次 parent 清空。每层需要独立父对象。 |
| GenerateMap(...,isPocketMap:true) | 使用父对象 ID 参与种子；设置 isPocketMap 与独立 pocketTileInfo，从 pocketMapProperties 读取 biome/mutators；随后正常构造组件、AddMap、生成、FinalizeInit/MapGenerated 和父对象回调。 |
| PocketMapUtility.GeneratePocketMap | 固定 WorldObjectDefOf.PocketMap 创建 PocketMapParent，赋 sourceMap/mapGenerator，生成后加入 World.pocketMaps；接口不接受自有 WorldObjectDef。 |
| Core WorldObjects.xml 的 PocketMap | worldObjectClass 为 PocketMapParent；这段定义未配置自有楼层身份。 |
| PocketMapParent.ExposeData | 保存 sourceMap 引用和 mapGenerator Def；MapGeneratorDef getter 返回保存的 mapGenerator。 |
| World.ExposeComponents | Scribe_Collections.Look(ref pocketMaps, LookMode.Deep)。父对象应沿正确的原版容器保存，不要另 Deep 一份。 |
| MapInfo.ExposeData | 保存尺寸、isPocketMap、阴影配置及父对象引用。 |
| Map.IsPlayerHome | 普通分支检查父对象玩家派系和 def.canBePlayerHome，另有重力船落地/引擎条件；不是仅凭“有殖民者”或 sourceMap 就为真。 |
| Map.Tile / MapInfo.Tile / WorldObject.Tile | Tile 最终读取父对象保存的 tile；这些 getter 不自动把 sourceMap.Tile 替换进来。 |
| Map.TileInfo | 普通图取 WorldGrid[Tile]；PocketMap 取独立 pocketTileInfo。 |
| Game.DeinitAndRemoveMap | 调 MapDeiniter、移出列表、地图移除通知，按 sourceMap 与 destroyOnParentMapAbandoned 条件销毁关联 PocketMap，最后 Dispose。 |
| MapDeiniter.Deinit / PassPawnsToWorld | 居民被移交世界等处理，可触发丢失、被俘、思想、Quest LeftMap 信号，并移除 Tick/各系统关联。 |
| PocketMapUtility.DestroyPocketMap | 从 world.pocketMaps 移除父对象后，DeinitAndRemoveMap(notifyPlayer:true)，还改变世界渲染模式。 |

本轮没有发现上述 PocketMap 生成入口上的显式 DLC 激活检查，Core 也包含 PocketMap WorldObjectDef。因此可将 Core-only 作为原型测试组合；这不证明生成、温度、UI 和所有依赖均已在无 DLC 环境运行成功。

### 推荐的原型比较

候选 A：自有 LayerMapParent : PocketMapParent、自有 WorldObjectDef 和专用 MapGeneratorDef，通过自有工厂使用原版生成入口并按正确所有权注册。不能直接调用固定原版 Def 的快捷工厂后假装已创建自定义父类。

候选 B：普通自有 MapParent 与 worldObjects 注册。它需要继续核对同一 PlanetTile 上多父对象、世界查询/地图选择与事件目标语义；目前不能断言与原版单基地索引完全兼容。

优先验证 A 以利用现成的隐藏子地图关系和持久化，但要显式设定所有者、允许居住/清扫、biome、温度、生成步骤、Tile/时间/纬度来源和拆层策略。PocketMapProperties 有 canBeCleaned、temperature、canLaunchGravship、preventPrisonerEscape、destroyOnParentMapAbandoned 等字段；不要依赖未验证的默认值或遗漏其对需求的影响。具体使用位置尚需按选定玩法继续定向审计。

不要一次性把每层都配置成独立 PlayerHome 并承诺事件正确。需求/工作需要的本层身份与故事叙述是否把该图当袭击目标，是不同决策；若采用 home 身份，要审计事件选图、殖民者数和财富是否被重复计入。

普通拆层先停止相关任务、撤销声明、搬走全部 Pawn/物品、断开连接，再走原版回收。DeinitAndRemoveMap 是关闭地图，不是可透明恢复的卸载 API；P0 禁止通过关闭有人楼层来省地图额度。

生成地图会改变 ProgramState、使用全局生成工作数据并执行场景回调。应先在受控初始化阶段生成并完成结构同步，再接受转移；不要把整张地图生成塞进本轮同步过层提交。MapCommonBase 会带入地形、遗迹、动植物、场景步骤；上层不能直接复用 Base_Player 的完整生成链。

## 五、sourceMap 已经影响财富，不能复用成楼梯图

WealthWatcher.ForceRecount 会遍历 World.pocketMaps，对 sourceMap == 当前 map 的已生成 PocketMap 递归 ForceRecount，并加上其物品、建筑、Pawn、地板财富及 totalHealth。

因此：

- 原版 root wealth 已包含其子 PocketMap；若自有聚合再将 root 和所有楼层财富相加，会重复计数。
- sourceMap 不能形成自环或多层循环，避免递归重算无终止。
- 楼梯可以任意相连、有多条通路；结构支撑是紧邻上下层；sourceMap 是原版归属/统计关系。这三张关系图必须分开。
- 建议先把每个楼层的 sourceMap 指向同一个根地图，层号/下层支撑关系另存；这是工程建议，不是原版必须采用星形的要求。
- 若 sourceMap 链式归属，销毁传播与财富递归都要明确。地图组改根、搬家、废弃或地层数量变化不能只改邻接表。
- 根财富及各层 IsPlayerHome/事件评分的组合仍需运行与调用链验证，不能据此宣称袭击点数已正确。

## 六、空洞、结构板和饰面的真实地形约束

### 原版证据

PathGrid.CalculatedCostAt：terrain 为 null，或 passability==Impassable 且没有满足飞行豁免时，返回不可通行成本 10000。PathGrid.Walkable 比较成本是否小于 10000；GenGrid.Walkable 使用 Normal 和 FenceBlocked 网格。禁止只靠降低 affordance 或隐藏纹理制造“空洞”。

TerrainGrid.SetTerrain 不接受 null。若新 terrain.layerable 且 underGrid 为空，旧地形可通行时把旧地形放入 underGrid；旧地形为空或不可通行时，用 generatorDef.defaultUnderGridTerrain，未设置则 Sand。非 layerable 新地形清空 underGrid。

DoTerrainChangedEffects 会更新绘制/路径/房间等，还会检查植物、污物，并将不再能建的 Blueprint/Frame 以 DestroyMode.Cancel 销毁；它不是纯粹换贴图。它没有在这段代码中提供普通完成建筑/Pawn 的跨层坠落处理。

Core Terrain_Water.xml 的 WaterDeepBase 示例使用 Impassable/changeable=false，但带有水、灭火、冻结、思想等父链语义；不能直接复制深水当作空洞。WaterDeep 还明确允许飞行 Pawn 通行；自定义空洞需显式决定该行为。实际路径系统另有异步网格，应按正常修改入口及版本协议失效，不能直接改内部数组。

### 建议模型

1. Void：专用非结构地形，普通 Pawn 不可通行、无建筑承载；是否允许飞行单独定义。
2. SlabSurface：下层有效屋顶投影得到的结构表面，具有允许的通行和承载能力。
3. Finish：木地板、瓷砖、地毯等饰面；铺设前必须已有 SlabSurface。

保持“饰面下方永远是正确结构面”的不变量。可研究非 layerable 结构面 + layerable 饰面，以及合理 defaultUnderGridTerrain，但这个 fallback 不能代替实际支撑验证。不能先在 Void 上直接 SetTerrain 一个地板，否则拆地板可能露出 Sand 幽灵楼面。

支撑消失时，先收集受影响建筑、蓝图/框架、物品及 Pawn 与任务，再进入受控结构失效提交；提交顺序需考虑 SetTerrain 自动取消蓝图/框架的副作用，防止自有逻辑再次退款/清理。失效掩码立即阻止新寻路/施工，具体坠落/损毁协议完成后才算支持塌楼。

## 七、屋顶旁路与飞艇兼容

本次对 RoofGrid.RemoveRoofUnsafe 做当前主程序集直接调用者搜索，结果只有 GravshipUtility.GenerateGravship；读取该方法确认其批量移除地形和屋顶、清建/拆屋顶区域、更新绘制/路径，再检查塌顶。该路径没有调用 SetRoof 的 RoofChanged 通知。

这不是所有 mod/反射调用者的完整枚举。建议以结构变化批次接口接入已支持的原版旁路，并在加载/生成结束重建核对，而不是每 tick 全图扫描。

飞艇从承载楼层的根图起飞时，下层结构被移走，上层会失去楼面。首版必须明确选择：阻止携带上层依赖的结构起飞，或先实现整组地图随船迁移与依赖更新；不能默认屋顶事件会自动解决。完整重力船适配仍未设计/验证。

## 八、P0 原型与后续仍需调查的事项

本轮收敛出的 P0 顺序：

| 步骤 | 必须观察的结果 |
|---|---|
| 两张已加载测试 Map，无携物 | 原 Pawn 往返，mapPawns/Tick 注册正确；旧 pather 被清理，新 pather 可用。 |
| 携带一批物品、装备/背包、被救援 Pawn | 真实所有者/数量保持；Job 交接无落地、无复制；预订按协议重建。 |
| 征召并关闭自由开火，多次上下楼 | Drafted 和 FireAtWill 保持；恢复状态不再打断下一段 Job；不自动切玩家视图。 |
| GameComponentTick 执行队列 | 旧 Driver 不重入、同 tick 无重复执行、取消代次有效；异步路径返回旧结果不再应用。 |
| 专用地图父对象与生成器 | 能保存重启、独立创建多层、原版父对象无重复 Deep；无 DLC 与目标 DLC 组合明确记录。 |
| Void→结构面→饰面→拆饰面→拆屋顶 | 无沙地 fallback、无凭空行走/建造；蓝图/材料只清理一次。 |
| 根图财富与关闭地图实验 | 子层财富只计一次；原版回收通知和源图销毁策略符合预期；仅用独立测试档。 |

本篇当时的关键未决接口已在后续专题补齐：[任务仲裁/预订及牵绳、机体、Lord](multilayer-coordination.md)、[公共系统](multilayer-services.md)、[建造与事件入口](multilayer-structure.md)、[战斗](multilayer-combat.md)、[单 Map 生成/显示参考](multilayer-reference-isorim.md)。当前原版的准确入口与静态风险在相应专题记录，历史方法证据不自动算作当前全面回归。

推荐以单 Map 原型先满足用户主线，见 [调研收口](multilayer-readiness.md)。若改选多 Map，具体 PocketMap 玩法参数、事件选图与独立地图合成显示应按选定分支继续实现/验证，不能把同 Map 参考当作这些分支也已运行。上述 P0、兼容、性能和存档运行测试仍未执行；本轮仅完成实施前静态调查。

## 批量追加地质时延迟区域与虫巢创建（1.7.4982，2026-10-20）

替换区域网格后，批量 `SetRoof` 可通过 RoomTempTracker 的 RoofChanged 回调触发区域查询；若同时逐格生成岩体，会在未完成的地质上反复建立区域。RimLayers 的 250×250、五层地下测试复现了 `RegionLink.Register: >2 regions on link`。追加事务使用新的 `RegionAndRoomUpdater { Enabled=false }`，完成地形、屋顶、岩体和路径代价后再启用并完整重建区域。

虫巢生成必须在区域恢复后进行：原生 Hive 的 Pawn/Lord 创建会查询合法 Region，不能放在上述禁用阶段。保存该层洞穴格与独立随机种子，区域重建后执行原生虫巢生成；后续回滚仍记录并移除本批新建实体。成功证据为 `Mods/RimLayers/Tests/Artifacts/v1-settings-perf-20261020-232818`，13 项通过；其后巡检游标优化另行复测。
