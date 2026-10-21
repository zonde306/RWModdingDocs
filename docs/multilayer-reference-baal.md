# 本地参考：巴尔「楼上楼下」Baal.SecondFloor

调研日期：2026-10-16。范围仅为用户指定工坊目录的 About、Def、DLL 元数据和相关成员；没有运行 DLL、导出整程序集、复制资产或修改工坊内容。

## 样本身份与版本

| 项目 | 本地值 |
|---|---|
| 根目录 | G:/SteamLibrary/steamapps/workshop/content/294100/3820071953 |
| About 名称 / 作者 | 巴尔：楼上楼下 / 想成为作家的厨师 |
| packageId | Baal.SecondFloor |
| supportedVersions | 1.7 |
| About 声明适配版本 | RimWorld 1.7.4977 |
| 依赖 / 明确不兼容 | Harmony；telardo.MultiFloors |
| 加载布局 | 根 Assemblies/Baal.SecondFloor.dll 与 Defs/SecondFloor.xml；未发现 LoadFolders.xml 或源码工程 |
| DLL 大小 / MVID | 69120 bytes / fa38fb6d4e2149d5b216e112b6970b4b |
| DLL SHA-256 | EB8FEF852716E42E20ECA86D48985296110ED5DF8E36D3A11B2D8FC5A08AE65F |

本轮发现游戏已更新：Version.txt 为 1.7.4980 rev60，主 DLL 大小 15632384 bytes，SHA-256 EFF7D97949F919FB2B151EBFB906E9BE749B63410B7DBDA78EEEBB7972B245C3，MVID 8e962ecc7ac742d28f3b1b16cd24d2cd。前几篇调研是 1.7.4979 rev109，不能混用旧 metadata ID。当前运行时仍为 net10.0 / 10.0.10-waylithclr.dev。

当前版本只复核本轮涉及的 Pawn 转移、Job 交接、财富和 TerrainGrid 相关入口；其他旧专题不因此自动升级为 1.7.4980 全面验证。该 mod 是否正常加载和运行也未测试。

About 宣称地上多层、地下室、电梯、电网连接、跨层工作/搬运/进食/休息/医疗；明确没有跨层射击、坍塌或重力飞船甲板支持。下文的已读实现与宣传范围分开记录。

## 关键结构

| 类型 | 已确认职责 |
|---|---|
| UpperFloorParent : PocketMapParent | 保存独立 level，父类保存 sourceMap/mapGenerator；ShouldRemoveMapNow 返回 false。 |
| FloorMap : 地图组件 | 按根图/层号查地图、建新层、登记楼梯、检查地板支撑及同步部分环境。 |
| FloorIndex | ConditionalWeakTable 缓存地图组件，统一 Revision；地图新增/移除触发失效。 |
| TransitLeg / FloorRoute | 一条跨层边 from/to；完整路线保存 legs 与估计 cost。 |
| TripPlan / Trips | 保存目的图、段列表/legIndex、resume Job、放货位置和用途；按 Travel Job 关联计划。 |
| JobDriver_StairTravel | 同一 Job 内循环“走到入口→等待→转移→下一段”，全部完成后交接恢复任务。 |
| FloorOrderContinuation | 在 EndCurrentJob 的成功收尾点阻止普通重新分派，启动 plan.resume。 |
| FloorQuery / QueryMap / QueryPosition | 为指定 Pawn 临时提供另一地图及预测入口位置，用原版逻辑查询候选。 |
| FloorWork / CrossFloorNeeds | 工作优先级比较，以及食物/床/娱乐需求的远程查询。 |
| Supplies / FloorMedical | 建造、账单、仓储的跨层供料及异图救援。 |

这些类型证明它与当前需求高度相关，不代表其接口适合作为依赖或实现已经满足我们的全部保证。

## 多次跨层如何保持目标

Trips.Reachable 维护候选 (Map,entryPosition,FloorRoute)，每次取估计成本最小项展开。它用 Stair 端点集合/最佳成本限制重复，不是 visitedMaps；在 FloorQuery 中检查当前图内能否走到某个楼梯入口，再追加与连接出口对应的 TransitLeg。因此“一层→二层→三层”确实是一条多段路线。

同图段成本用直线距离，楼梯加固定估计 12，电梯加 3+层差。实际 Travel 等待另有 TravelTicks；这不是准确的总耗时/最优真实步行路线。值得借鉴端点图，但不能把该估计包装为全局最优。

TripPlan 保存完整 legs 与 legIndex。JobDriver_StairTravel 经过二层后修改当前 job.targetA/B 指向下一段，并跳回 approach Toil，不重新跑工作搜索；最后一段结束才置 crossed=true。因此中间层不会因正常路段完成而立即换目标。

FloorOrderContinuation 的 Harmony 目标是 Pawn_JobTracker.EndCurrentJob：仅在 Succeeded、当前 Driver 是该 Travel 且 crossed 时，取走 resume，并让 startNewJob=false；Postfix 再 StartJob(resume,keepCarryingThingOverride:true)。它解决了我们前文所说的“最终层交接前普通 ThinkTree 抢走控制”问题，是具体、可定位的参考。

但当前段连接不可用时，Driver 的 FailOn / 等待检查会结束行程；已读执行链没有“保留最终业务意图→自动重算剩余路线”的协议。成功的多段连续行走与动态断路后的目标级恢复应分开验收。

## 转移采用保留 Driver 的另一条路线

Trips.Transfer 的实际步骤：

1. 保存 curJob、curDriver、旧 drafter、源图位置/旋转及部分 Lord/选择状态。
2. 暂时将 pawn.jobs.curJob 与 curDriver 置空。
3. 调 Pawn.DeSpawn；因 curJob 为空，原版不走 StopAll，但其他离图通知与 ClearAllReservations 仍执行。
4. 用 GenSpawn.Spawn(...,WipeMode.Vanish,respawningAfterLoad:true) 入目标图，重置新 pather，并把旧 drafter 赋回。
5. finally 恢复原 curJob/curDriver；故原 Travel Driver 可继续下一段。
6. 对特定敌袭 Pawn 重新创建 AssaultColony Lord；选中的 Pawn 过层可切换视图。

该方案解释了它为何不需要每层重新创建 Travel Job，也解释了 Def 中 carryThingAfterJob=false 仍可能在中途保留携物：普通过层绕过了 Job 清理，正常最终收尾则按业务放货/恢复。

这与我们的当前建议“正常清理 + 自有意图 + 重建本层 Job”不同。它的优势是复用现有 Job/Driver 状态、适配代码较少；代价是依赖 Pawn_JobTracker 内部字段、恢复旧控制器、把移动当读档生成以及从 Toil 内同步转移。

当前 1.7.4980 已复核 Pawn.DeSpawn、SpawnSetup、CleanupCurrentJob、StartJob：上述字段清空确实避开 StopAll，其他离图清理仍在；respawningAfterLoad 标志确实影响 SpawnSetup 的多项通知。不能只因它在参考 mod 中这样做，就撤销我们对这些生命周期影响的审计要求。

首版可以做两条最小原型对照，但不混用：A 正常清理/重建；B 参考方案的完整保留。比较携物、征召、多段路径、原版通知、取消、读档与第三方补丁。无运行数据前，继续保留 A 为默认架构建议。

## 数据保存并非只靠静态字典

Trips.plans 是 ConditionalWeakTable<Job,TripPlan>。TripSerialization 在 Job.ExposeData 对 BSF_Travel 附加 Scribe_Deep 的 BSF_trip；TripPlan 保存 resume Job（Deep）、目的图/recipient（引用）、放货格、purpose、legs 与 legIndex。Driver 另存 crossed 与动画起始时间。

这说明它有实际的多段计划读写入口，不能因为计划表是 static 就断言无法存档。同时，也不能把这种做法等同于随意 Deep 当前正在运行的 Job：这里 resume 是计划持有、尚待恢复的另一个 Job。

可借鉴明确的段索引和关联序列化；仍需检查候选 Job 被舍弃、池化复用、取消后 resume 的所有权、失败清理和读档是否只启动一次。现有模型没有我们建议的独立 taskId/generation，因此旧回调隔离和目标级恢复还需另做验证/设计。

## 远程工作、食物与家具的查询手段

FloorQuery.active 是 ThreadStatic，作用域保存 previous 并在 Dispose 恢复，支持嵌套。QueryMap/QueryPosition Harmony 修改 Thing.Map / Thing.Position getter，但只对当前作用域命中的特定 Pawn 返回预测 Map/Position；并非永久改 Pawn 的真实字段，也不是直接切 Find.CurrentMap。

FloorWork 在作用域内重入原 JobGiver 查询，使原版看到“这个 Pawn 已在目标图入口”。FloorQuery.Busy 防止远程扫描补丁无限递归。这个技术能够同时复用原版路径、预订资格和大量 WorkGiver 条件；比全局无条件替换 Map 的范围小，但原版 worker 的缓存、随机数、建 Job/改账单等副作用仍须审计，ThreadStatic 也不能自动解决跨线程异步工作。

我们前文推荐显式参数适配器，是为避免隐含 Pawn 上下文。参考实现提供了可研究的折衷；不要误写成已经证明任意 WorkGiver 在查询作用域内都是纯函数。若采用，限定支持列表，禁止作用域跨 await/异步调度，并记录嵌套和异常恢复。

FloorWork.Select 比较 WorkType 设置优先级、自然优先级和 giver.priorityInType；同 rank 不一定重新按完整旅行成本择优。普通远程扫描有 90 tick 退避，紧急扫描有 60 tick 退避，需求扫描有 60 tick 退避；RouteQueryScope 在一次查询内复用路线结果。

FloorWork.Needs 只处理 GetRest/GetFood/GetJoy，优先考虑楼层组中的 OwnedBed，并对本地地面回退等条件尝试远程候选。这印证“不能只在原版返回 null 后补找床”的判断。它仍保留许多本地有效 Thing 结果，不等于所有需求都按跨层总成本做全局比较。

餐椅/餐桌完整多阶段选择、所有家具类型、药物/装备需求不在本轮已验证支持范围。About 的“进食/休息”不能扩写成任意需求自动跨层兼容。

## 供料、预订与敌人

Supplies.Find 的分支依次考虑：建筑蓝图/框架缺料、账单原料、较高优先级仓储。Bring 选择目标附近可达放货格，Travel 拿货跨层后落地；加工交给原版后续任务。这是“先供料再本层工作”的具体参考。

Incoming 汇总各层当前 Travel 的货物数量，减少重复供应；它是流入量统计，不是目标材料/工作台/容器容量的强租约。已读 Travel 的预订是当前货物及当前楼梯的原版预订；过层后原版仍清该 Pawn 的预订，下一段再申请入口。不能据此宣称已经解决长途目标被别人占用或所有并发数量竞争。

FloorMedical.Rescue 找到目标层床后，创建带 resume Rescue Job 的取人/跨层计划。跨层医疗有单独适配，不能概括为把所有医疗行为无条件自动转换。

HostileStairUse 在特定 AssaultColony Lord、当前图没有相应有效敌人的条件下，从其他层找敌方 Pawn 并派 Travel；转移时可能新建本图 AssaultColony Lord。它可防部分“上层永久安全区”，但不是跨层射击，且目标感知/战术继承比我们计划的完整规则窄。

## 上层屋顶建造：相似但不等价

UpperFloorGenerator：地上层初始化为 BSF_OpenSpace 并揭雾；地下层填粗糙花岗岩、厚岩顶和岩石。不是每层都生成自然殖民地图。

FloorMap.Supports：地下层或楼梯周围特殊落脚范围直接允许；普通地上层要求紧邻下层对应格 RoofAt == RoofConstructed。天然顶不在此普通地上判定中。

FloorPlacement patch GenConstruct.CanBuildOnTerrain，逐格处理建筑占地：位于 OpenSpace 时，只有 TerrainDef 且 Supports 为真才可通过额外检查。玩法因此是“在下层人工屋顶上铺地板扩建，再在已铺楼面上建建筑”，并有楼梯落脚例外。

关键区别：该检查针对 OpenSpace；已铺地板位置不在这里再次验证下层屋顶。已读 FloorMap.Tick 负责连接修复、天气和温度交换，没有屋顶失效→撤楼面/坠落处理。About 也明确无坍塌。因此不能直接视为满足我们的持续结构支撑要求。

BSF_OpenSpace 同时 Impassable、changeable=true，并给予 Light/Medium/Heavy affordances；是否能铺地板再由补丁控制。其 MapGeneratorDef 没有显式 defaultUnderGridTerrain。结合当前原版 SetTerrain 的不可通行旧地形 fallback，存在铺可叠地板后底层为 Sand 的静态风险；拆地板实测仍需验证。首版应继续采用独立结构面/饰面不变量。

## 地图创建、财富和环境

FloorMap.EnsureLevel 创建 UpperFloorParent，sourceMap 指 Ground，level 独立保存，Tile 设 Ground.Tile，派系设玩家，并复制根地图尺寸；专用 PocketMap 生成后注册 world.pocketMaps。它确实采用“原版归属为根、结构层号另存”的方案。地图数到原版阈值时拒绝继续。

特殊楼梯/电梯、天气与温度、电网还有相应模块；本轮只读取部分：FloorMap.Tick 将地上天气同步根图、地下清天，并在楼梯房间间交换温度；电力相关类型仅列出入口，未审计电网算法。

发现一个需要优先验证的财富问题：

- FloorWealth patch WealthWatcher.WealthTotal getter，在根图结果上再加 OtherFloors 的 WealthTotal。
- 当前原版 1.7.4980 的 WealthTotal 读取 ForceRecount 维护的字段，而 ForceRecount 已递归累加 sourceMap==根图的 PocketMap。
- 样本楼层正是 sourceMap=Ground 且注册 world.pocketMaps。因此在该补丁和当前原版路径都生效时，存在再次累计子层财富的明确静态风险。游戏面板、其他补丁和实际袭击影响尚未测量，不能冒充已复现的玩家问题。

## 对我们方案的取舍

| 可借鉴 | 需要保留不同设计或补齐 |
|---|---|
| 楼梯端点路线、TransitLeg 列表、legIndex | 完整耗时、动态断路后的目标保持与重规划。 |
| 最后一段成功后阻止普通重新分派并交接 | 明确的任务身份/取消代次、异常/失败状态及本层生命周期契约。 |
| 自定义 PocketMapParent、独立层号、专用生成步骤 | 结构失效、事件归属、财富去重及当前版本回归。 |
| OwnedBed 在本地地面回退前参与远程比较 | 家具/需求的完整使用资格、路程预算与抢占政策。 |
| 供料与加工拆分、查询级路线复用、扫描退避 | 强资源声明、数量与容量竞争、租期和公平性。 |
| 当前行程的显式序列化 | Job 池化/独占保存、读档去重及旧回调隔离。 |

不直接采用的默认做法：清空内部 Job 字段绕过清理、用读档标志移动、恢复旧 drafter、全类型工作查询虚拟 Pawn 上下文、额外重复汇总财富，以及只在铺地板时检查支撑。可把前四项作为有明确测试边界的替代原型，而不是从文档中抹去它们的存在。

## 下一步验证

1. 三层两段楼梯，给三层工作台明确命令，二层有普通工作；核对同一 Travel 的段索引和最终 resume。
2. 第二段途中拆楼梯/断电/目标消失，观察是停止、改目标还是保目标重算。
3. 携物、救援 Pawn、征召且禁止自由开火，逐段保存重启；检查 Job/Driver、控制器和所有权。
4. 多 Pawn 同抢材料/床位/工作台，检验 Incoming 统计之外是否还会过量供应/失去目标。
5. 屋顶上铺地板再拆地板、移除下层屋顶、楼梯落脚例外；检查 Sand 和悬空建筑。
6. 根图与一层已知价值物品比较财富，再加第二层；检查当前 getter patch 是否重复累计。

本轮只做静态参考调研，未启用 mod、未运行上述场景。没有修改玩家配置/存档、工坊内容或游戏运行时。
