# 连续跨层寻路、需求选物与目标保持

核对：2026-10-16，RimWorld 1.7.4979 rev109。主程序集 SHA-256 为 A97A01C4319414414C0CA39F3565215E83300DC608DC94B69A27720095B4E751，MVID 为 425a593954d7432a8f44e3076b25fce9，属于 [sources.md](sources.md) 的历史基准；当前安装为 1.7.4980，有限重核另列。总体架构与屋顶建造约束见 [multilayer.md](multilayer.md)。

状态：已完成定向元数据/方法体及原版 XML 调研。本文协议、类型名、算法接法和参数策略均为待实现建议；未编译实现、未启动游戏测试。目标是正常经过中间层不会丢失意图，同时保留玩家取消、危险与需求抢占。

## 架构适用范围更新

本文任务协调和离图续接协议主要面向一层一 Map。后续 [IsoRim ProjectZ 调查](multilayer-reference-isorim.md) 确认另一条路线：多层存于同 Map 的不同坐标带，将层内路径与楼梯节点拼成一条 PawnPath，最终 Destination 不变，普通过层无需结束/替换 Job。该分支的有限选物/选料半径、存储距离、拓扑漏路与读档恢复已在 [queries](multilayer-queries.md) 进一步核实，具体修复仍待实现。保留业务意图适用于复杂多阶段工作；不应为单 Map 的每个楼梯强制派发 Travel Job。

单 Map 分支的进一步当前版本审计见 [multilayer-queries.md](multilayer-queries.md)：原版 DoBill 已有材料/数量队列和多次取料循环，可保留同 Job；选料半径、格子工作、材料落点和连接器逐步验证仍需适配。正常产品入库可能将已成功 DoBill 换成 Haul Job，属于正常业务交接，不应被“保目标”协议阻止。

## 需求与保证范围

用户明确场景：Pawn 在一层，需要三层的物品、家具或工作台，必须经过二层；部分任务还需要多次上下楼。在二层经过或短暂等候，不应被重新分派普通工作。工作台在三层、材料在一层时，去取材料与回工作台属于同一个生产目标的不同阶段。

“目标不中断”应分成四项可验证保证：

1. 正常过层不会取消最终目标；中间层只是路线的一部分。
2. 同一次任务的目标与资源不会因为每到一层就重新全局评分而反复改变。
3. 断路时先重规划路线；可替换资源失效时按规则换资源；最终对象失效才重选任务或失败。
4. 用户取消、征召或必须让步的情境仍可结束/暂停任务，旧回调不能将其复活。

不能保证任意 mod Job 的任意 Toil 都能跨图原样续跑。稳定的是任务意图与已提交的业务进度，本层 Job 是否能恢复由具体适配器和原版契约决定。

## 原版已核实事实

| 当前成员 | 核实内容 | 对设计的影响 |
|---|---|---|
| Pawn_JobTracker.JobTrackerTickInterval | interval 路径按 hash interval 30 检查 ConstantThinkTree；处理当前 Job expiry；curJob 为空且 Pawn 存活、mindState Active 时 TryFindAndStartJob | 只等下一 tick 补 Job 会有普通重新分派窗口；也不能全面关闭常驻安全行为。 |
| Pawn_JobTracker.DetermineNextJob / TryFindAndStartJob | 先 ConstantThinkTree，再主 ThinkTree；有效结果通过 StartJob 启动 | 活跃跨层任务需要进入派发仲裁，不能只在当前地图扫描外另开一个无约束调度器。 |
| Pawn_JobTracker.EndCurrentJob | 默认 startNewJob=true；清理后可能启动 finalizer、1 tick Wait_MaintainPosture 或重新选 Job；ErroredPather/Errored 默认进入 Wait 250 | 本层 Job 成功、短暂站定或寻路失败，都不等于最终任务完成。 |
| Pawn_JobTracker.StopAll | 根据条件清理当前 Job，始终 ClearQueuedJobs | 原版 jobQueue 不能独自承担跨图计划。 |
| Pawn_JobTracker.StartJob / SuspendCurrentJob | 只有请求恢复且 def.suspendable 时才暂停入队；暂停把 curJob 放队首；新 Driver 执行预订后才 SetupToils | resumeCurJobAfterwards 不是跨图持久保证；到目标层才能重新完成本层预订和执行前检查。 |
| Pawn_JobTracker.CheckForJobOverride | DetermineNextJob 后按 ShouldStartJobFromThinkTree 和 minPriority 决定替换；另有队列及 Job 池回收 | 不能将全局 override 一律跳过来保目标。 |
| Pawn_JobTracker.ShouldStartJobFromThinkTree | 同 JobDef、非排队且 Driver.IsContinuation 为真时还比较 SourceNode/jobGiver | 共享 Travel JobDef 的阶段身份需要自己定义，不能认为同 Def 就是同任务。 |
| JobDriver.IsContinuation | 基类默认返回 true | 自有 Driver 应按任务/阶段身份定义 continuation，或使用明确的阶段交接协议。 |
| JobDriver.Notify_PatherArrived / Notify_PatherFailed | 前者对 PatherArrival Toil 调 ReadyForNextToil；后者 EndJobWith(ErroredPather) | 到楼梯或本层路段终点只是 Toil 推进；失败应由本 mod 路段上报路由层。 |
| Pawn_JobTracker.TryTakeOrderedJob | 涉及 playerForced、打断标志、清队列/入队，以及立即预订 | 玩家新命令/排队命令需要转换为相应跨层意图；不能把远程原版 Job 直接排进去就期待延迟预订。 |
| Pawn_JobTracker.ExposeData / JobDriver.ExposeData | 原版保存 curJob、curDriver、jobQueue；Driver 保存 Toil 索引/剩余时间等，PostLoadInit 重建 Toils | 不可把同一原版 Job/Driver 再由协调器 Deep 一份；读档时匹配现有 Driver 与任务意图，避免重复启动。 |
| JobGiver_GetFood.GetPriority / TryGiveJob | 需求阈值及是否应由别人喂食影响优先级；选食物后也可能生成狩猎、采收、补料、从他人背包取物或 Ingest | “需要物品”不总是一个 Goto；各类满足方式需明确支持范围。 |
| FoodUtility.TryFindBestFoodSourceFor / BestFoodSourceOnMap | 比较背包与地图食物，地图索引/可达性依赖 getter.Map；包含饮食、禁用、社会关系、预订等条件 | 不可把异图物品加入列表后继续用单图距离和 validator。 |
| JobGiver_GetRest.GetPriority / TryGiveJob | 作息、疲劳、Lord 等影响优先级；FindBedFor 不成功可回退地面睡眠 | 远程有效床需要在地面回退之前参与选择；不能只等该 JobGiver 返回 null 再补跨图候选。 |
| RestUtility.FindBedFor / IsValidBedFor | 床型/医疗需求、OwnedBed、伴侣床和其他床各有规则；有效性含 traveler.CanReach / CanReserve、床位数量、权限与禁用 | 床的归属与床位容量应保留；完整原版 validator 不能直接用于异图 Pawn。 |
| JoyGiver.GetSearchSet / JoyGiver_InteractBuilding.FindBestGame / CanInteractWith | 候选来自 pawn.Map；最近可达搜索使用本图；检查参与人数、禁用、迷雾、社会/政治条件、电力、屋顶等 | 娱乐家具要显式接入多层候选与到达后的本层复核。 |
| JobGiver_GetJoy.TryGiveJob | 根据娱乐耐受、可用性等加权尝试 JoyGiver | 不应每过一层重新抽娱乐类型。 |
| Toils_Ingest.TryFindChairOrSpot | 在 pawn.Map 搜座椅，检查可坐位置、预订、派系、禁用、危险及邻接进食桌面；返回裸 IntVec3 | 跨层就餐位置必须补 Map；进食地点属于可选子目标，不能让寻椅压过紧急进食。 |
| WorkGiver_DoBill.JobOnThing | 检查账单当前需求、工作台可用性、工作台/交互格预订、燃料和燃烧；可能返回 RefuelJob，随后 StartOrResumeBillJob | 到台子不保证账单能开工；工作台目标、账单目标与补给阶段分开。 |

前文已核实的 Pawn.DeSpawn、ClearAllReservations、JobGiver_Work、WorkGiver_DoBill 选料和本层 PathFollower 约束仍适用。这里只补新增方法与解释，不导出整程序集。

原版 XML：Data/Core/Defs/ThinkTreeDefs/Humanlike.xml 的 SatisfyingNeeds 分支包含 GetFood、GetRest、GetJoy 等；Joy/JoyGivers.xml 按娱乐种类定义相应 giver。此次只读参考，未修改原版 XML。

## 一、把目标、子任务、路线和本层执行分开

建议拥有关系：

- TaskIntent：最终目的、来源需求/命令、主体 Pawn、目标替换策略、优先级语义、状态、已提交进度。
- TaskStep：当前取物/供料/就餐/睡眠/加工阶段，其前置条件、成功条件和必要资源。
- RoutePlan：当前阶段源/目标的跨层路线、拓扑版本、计划代次及段索引，可被替换。
- LocalExecution：本层移动或动作 Job 的身份，及关联 taskId/stepId/generation；由原版保存自己的 Job/Driver。

示例 A：

一层 → 本层楼梯入口 A → 二层出口 A' → 二层楼梯入口 B → 三层出口 B' → 工作台交互格 → 做账单。

最终目标和 taskId 不变。中途切换地图只推进 routeSegmentIndex。二层若有两个被墙隔开的入口，必须判定失败或绕路，不能只凭“二层连接三层”宣布路线存在。

示例 B：

二层 Pawn → 一层取钢 → 三层工作台附近交付 → 回二层取另一类材料 → 三层加工。

这是一个目标的多段子任务计划；每个取物/交付阶段内部又可有多次跨层。多个 Pawn 协作供料时，用可等待的供应任务/需求单连接，不能让两个任务彼此等待并各占一半资源。第一版仍建议先把材料供应到工作台所在图，再执行原版账单。

物品是否可换由目标语义决定：同类钢材可按数量和过滤器替换；玩家指定的独特物品、已绑定账单/工作台、OwnedBed 默认保持身份；若失效则明确失败/重新仲裁，不能静默换成无关目标。未完成品/产物等身份与进度由各工作适配器按实际原版契约处理。

## 二、路线搜索：允许经过同一层多次

搜索节点用 (map/levelId, endpointId, traversalState)，不是只有层号。地图内部入口间的连通/成本来自本层通行图，连接器提供有方向的跨层边。起点与目标交互/接触格加入临时节点；允许同层目标经其他层绕行，也允许路径重复进入同一张图的不同端点。

visitedMaps 会错误裁掉合法路线，禁止用它代替搜索状态。Dijkstra/A* 保存端点状态的 bestCost，非负边权及重复状态淘汰限制环路；与能力、携带限制相关的状态必须进入键。无需任意规定“只能跨一次/两次”，实际应有计算预算、重试和无进展限制。

不能把不同 Map 的裸 x/z 相减作为距离。总成本至少包含本层行走、楼梯/电梯耗时及等待估计；物品阶段还应考虑空手/携带移动速度、交互耗时、后续运输目的地。段内真实路径可以按需计算，但缺失路段不能默认为可达。

每段终点都验证 Map + Cell/Thing + PathEndMode。最终成功以“到达目标地图且完成阶段动作/满足业务条件”为准；走到二层出口、跨完一次层、甚至到三层地图，都不等于已经使用工作台。

动态目标/门/屋顶楼面/危险变化失效相应路线段。普通断路先保目标、重算剩余路线，不回起点重做全部任务；不能通过尚在等待拆顶处理的无支撑格。已完成且可验证的取料/交付阶段保留进度。

## 三、目标选择要覆盖各类需求

统一候选接口应显式接受源地图/位置、Pawn 能力、需求及最终使用目的；返回目标身份、地图、交互位置、成本区间、条件、可替换性和可用容量，不直接创建或启动远程 Job。

| 需要什么 | 选择时需保留的条件 | 到达后怎么执行 |
|---|---|---|
| 食物/药物/装备/材料 | 允许种类、数量、品质/状态、饮食/药物策略、权限、来源容器、有效性和完整路线 | 按类别适配器在本层拿取/使用；先明确支持地面、仓储容器或他人背包哪些来源。 |
| 床 | 归属、床型、医疗/囚犯等资格、床位槽容量、危险与可达性 | 到床所在图后本层复核与 LayDown；不能因过二层就就地睡地板。 |
| 餐椅与桌面 | 座位位置、桌面邻接、进食目标、权限及绕路预算 | 可保持已携食物跨层就餐；紧急时允许取消座位子目标并就近进食。 |
| 娱乐家具 | 娱乐类别/耐受、设施状态、电力、参与人数、社交限制、操作位 | 保持本次活动选择，到图后创建对应本层 Job；不保证所有 JoyGiver 自动支持。 |
| 工作台 | 工作类型/技能、账单、燃料/电力、材料供应、交互格及台子占用 | 必要补给完成后再尝试原版工作；不能在到台后随便重扫到另一张账单。 |

候选先按原版资格与需求优先级过滤，再比较整个行动成本。几何最近不一定路程最近；更高品质食物或床不应无限抵消跨层耗时。远处的优选资源超过需求安全预算时，选择能及时满足需求的备选。

这不是只改 WorkGiver：需要分别接入需求 JobGiver/utility 与工作 scanner。仅做“原版返回 null 后找远程目标”会漏掉有效远程床被地面睡眠抢先、较差本层候选提前返回等场景。应在相应回退之前比较，或由该类别适配器统一构造候选与策略。

不临时修改 pawn.Map/Position，不对 Find.CurrentMap 作全局伪装，不使用假 Pawn 试跑 getter；原版方法含本层预订、缓存、Toil 前置与有状态逻辑。可复用已确认无副作用且不依赖本层上下文的资格判断，其余拆为显式检查；抵达后复核也要绑定预选目标，不能重新调用会任意另选目标的全局搜索。

## 四、正常过层的连续派发协议

转移执行时机、携带物保留、征召控制器重建与地图回收的后续静态证据见 [实现前置调研](multilayer-foundations.md)。GameComponentTick 可作为统一队列的原型候选，但不是异步寻路已全部结束的屏障。

建议由每 Pawn 的协调器管理 ActiveTask，并在适用 ThinkTree/Job 派发仲裁中提供“继续当前阶段”。保留原版常驻安全行为及明确的抢占条件；各类任务沿用其需求/工作来源的优先级语义，不能给所有跨层移动一个永远最高的固定优先级。

正常交接：

1. 验证 task/step 仍有效，确定下一段并准备资源声明。
2. 本层 Travel Job 走到连接器；只有当前段完成条件满足才准备过层。
3. 进入带 taskId、stepId、generation、transferId 的转移上下文，保存可恢复状态。
4. 原版执行正常清理及离图/入图；将期间的 StopAll/InterruptForced 识别为本次转移，不作为业务取消。
5. 实际地图/落脚点确认后，提交过层结果并只推进一次段索引；在安全派发点提供下一段 Job。
6. 最终到图后，绑定选定目标建立本层 reservation，创建动作 Job，动作成功才提交阶段完成。

这要求在原版可能自动重新派发的位置接入，而非把 curJob 清空后等待组件下一 tick 抢回控制。具体入口已补到 [任务仲裁专题](multilayer-coordination.md)：普通续行保持原工作节点的 PrioritySorter 位置，常驻安全树保留，当前地图合法完成可使用 SetFinalizerJob；跨 Map 仍用受控转移。实现重入和保存时序仍需原型验证，不能从 DeSpawn 清理回调内重入 StartJob。

只处理本 mod 的 Travel/动作适配 Job 和活跃 Task；不全局禁止 JobTrackerTick、CheckForJobOverride、EndCurrentJob 或 StopAll，也不通用吞掉异常。若需要 continuation，比较同一 task/step/generation 和合法路段，不能依赖基类恒 true 的行为。未采用的临时 Job 按原版池化契约释放，不把回收到池中的 Job 引用当作长效任务 ID。

每个异步路径结果/Driver 完成回调携带任务代次，应用前检查：仍是该 Pawn 的 ActiveTask、任务未取消、step/segment 匹配、实际地图符合预期、generation 未变化、结果未提交。过期结果丢弃。用户取消或新命令替代时推进代次/写终止状态，使旧的抵达通知不能复活旧目标。

## 五、目标保持与允许中断的边界

| 事件 | 对最终目标的处理 | 对当前执行的处理 |
|---|---|---|
| 经过二层或等待一段通行 | 保持 | 继续下一段；允许有限等待。 |
| 普通本层清扫/同优先级更近资源出现 | 默认保持 | 不仅因为切图重新择优；设切换收益阈值/冷却避免往返。 |
| 路段堵塞、楼梯销毁、楼面失效 | 优先保持 | 安全停止，重算剩余路线；无路则退避/失败。 |
| 某堆可替换材料被搬走/拆堆 | 保持需求和数量 | 修复引用/声明，换合法资源并重算相应阶段。 |
| 指定工作台/床等主体失效 | 按身份策略失败或重新仲裁 | 停止依赖阶段；不悄悄另选无关对象。 |
| 紧急需求/危险/更高优先级事件 | 暂停或取消 | 清理本层 Job 与不再必要的声明，必要时保存恢复检查点。 |
| 玩家取消/非排队新命令/征召 | 明确终止或按命令语义替换 | 不自动恢复；Shift 排队按自有意图队列保留次序。 |
| 死亡、任务权限消失、地图移除 | 失败/终止 | 清理所有相关任务与运输持有关系。 |

JobCondition 不是完整业务原因：过层 StopAll 和用户打断均可能表现为 InterruptForced；expiry 导致 Succeeded 也不证明达成目的。通过操作来源与受控上下文判断原因，外部 mod 发起且原因不明的打断要保守让步并诊断，不能一律自动重启。

软中断可以暂停任务目的，但原版任意 Job/Toil 无通用的可逆执行保证。只在已验证检查点恢复；已消耗物品、已交付数量或已完成账单不能重放。进食、睡眠、娱乐等需求满足后，原工作任务恢复前需重新检查时间表、权限与资源，不无限堆积互相打断的任务栈。

长路途先估计需求恶化和可接受行程；候选评分与路上检查结合，避免 Pawn 跨多层去吃饭途中饿倒。到各层不等于必须重选目标，但危险/需求变化仍能及时让步。数值阈值待游戏测试，不沿用未经验证的旧版常量。

## 六、资源声明和多阶段运输

远程声明应由 taskId/stepId 持有而不是由某个短命的 Travel Job 持有；正常 DeSpawn 清原版预订时，它仍按协议存在。声明必须参与本层原版竞争校验，不能只在自有字典中“看起来锁住”。

- 对物品锁定数量和来源身份，并记录已取得/已交付数量；SplitOff、合堆和容器变化重新定位。
- 对床、座位、娱乐家具和工作台分别使用合适的槽位/参与人数语义，不能把整栋多床位床视为单人独占。
- 长途目的地可先做短期意向，临近抵达再升级为使用声明和本层 reservation；若采用强排他声明，必须让本层任务尊重且有租期，不能无限霸占工作台。
- 锁定后仍可能失去电力、被摧毁或权限改变；抵达/交接时复核，不能把声明当作对象不变的保证。
- 正常移动或合法排队可续租；无进展、长期睡眠、取消或失效时释放/降级。排队需要更新有限等待证据，不能只把“仍在任务中”当成永久续租理由。
- 多资源按稳定顺序取得，失败撤回；未必要同时独占全部未来阶段。当前携带物由真实持有者拥有，暂停/换任务前明确继续携带或安置位置。
- 入库、取物和过层提交使用状态/回执去重。数量/物品所有权和步骤完成在受控提交点一致，避免读档或重复回调扣两次、交两次。

先完成单物品/单家具跨三层访问，再加入取料→供料→工作台的多阶段计划。最优多目的地批量路线不是首版必需；应声明使用启发式及计算预算，不能承诺全局最优。

## 七、存档恢复与诊断

自有 TaskIntent/TaskStep 保存 dataVersion、稳定 taskId、来源、目标策略、阶段、已提交数量、恢复检查点及必要的声明。已有 Map/Pawn/Thing 用引用；账单/未完成品的身份按其实际序列化契约确认，不以列表下标或物品 Def 替代身份。路线成本、Region 和真实路径不持久化。

原版仍保存现有 curJob/curDriver；协调器仅保存关联标识，不重复 Deep。交叉引用完成后先核对实际地图、目标、携带物和提交状态，再匹配现有本层 Driver：一致则接管继续；不一致则安全清理并从检查点重建。新一代路由结果替代旧结果，避免同时存在读档恢复 Job 和新派发 Job。

记录 taskId/stepId/generation、目标层与实际层、路线段、来源 JobGiver、暂停/取消/换目标原因、声明数量/到期和最近进展时间。界面报告应显示“前往三层工作台，经二层”，而不是只显示没有业务上下文的上楼。检测同一状态来回切换、反复进出同一入口或长时间无进展，有限重试后解除锁定并给出原因。

## 八、必须通过的游戏内场景

| 场景 | 成功标准 |
|---|---|
| 一层去三层工作台，经二层；二层有普通待办 | 不改最终目标；到交互格后才执行工作。 |
| 二层两个楼梯隔墙、不连通 | 不误报三层可达；能找到实际绕路或明确失败。 |
| 合法路线重复经过同一层不同端点 | 不被 visitedMaps 剪掉，也不无限循环。 |
| 三层床与一层地面睡眠回退同时存在 | 按床/需求策略选择，不因原版回退提前丢失远程床。 |
| 去远层食物途中需要就餐位置 | 保持食物数量/持有关系；椅子可选，紧急时不为找椅饿倒。 |
| 一层材料、三层台子；连续多次取料与交付 | 已交付不重复搬/扣，阶段推进不会重选账单或丢物。 |
| 多 Pawn 争物、双人床、多人娱乐及繁忙工作台 | 数量/容量正确；本层原版 Job 也尊重声明，无无限霸占。 |
| 上二层后楼梯损毁/下层屋顶坍塌 | 原地安全处理，保目标重规划；不穿空洞。 |
| 物品拆堆合堆/换图、工作台断电、账单取消 | 只修复允许的目标；无效主体正确暂停/终止。 |
| 等电梯、饥饿抢占、征召、取消、Shift 排队 | 合法续行或明确让步；旧回调不会复活被取消任务。 |
| 每次过层及取物/交付前后保存，退出重启读档 | 不重复生成任务、复制物品、重放动作或遗留预订。 |
| 本地任务/其他 mod 不参与楼层组 | 行为保持其本层契约；未适配任务可诊断，不声称自动兼容。 |

本次没有实现原型，因此上述都是待验证验收条件。文档格式/链接检查不等于游戏行为或性能验证。

## 本地实现对照

[Baal.SecondFloor 实现参考](multilayer-reference-baal.md) 已确认用完整 TransitLeg 列表、同一 Travel Job 的 legIndex 和 EndCurrentJob 成功续接避免在中间层重新分派；它也提供 OwnedBed 和供料的实例。但它保留 Driver 过层、虚拟 Map/Position 查询的路径不同于本文默认方案，且当前段失败后的目标级重规划仍需补齐。参考样本/当前游戏的新版本基准单列于专题，不能沿用本文旧 metadata ID。

## 后续最小实现顺序

1. 自有调试任务，仅移动：一层→二层→三层，带持久 taskId 和取消代次。
2. 加入端点路由、回到同层的绕行和断路重规划；确认中间层不会进入普通分派。
3. 加入目标物品数量/声明/持有、床或单个工作台访问，验证到图后的绑定复核。
4. 接入需求优先级、合法抢占与恢复；检查原版队列、expiry、continuation 和 Job 池行为。
5. 加入多阶段材料供应及家具组合，最后扩展到指定自定义 Job。


## 单 Map 实现增量：可达性不能每次生成完整路线（2026-10-18）

当前1.7.4981 rev136的原生 Reachability.CanReach（060016E1）先处理Immediate/同District、解析目标模式，再通过Region及其缓存查询；PassAllDestroyable等特殊模式和NoPassClosedDoorsOrWater另有格级连通性分支。不要把它们一概描述为纯Region判断，也不要把每个CanReach替换为FindPathNow。

FoodUtility.SpawnedFoodSearchInnerScan（06004A63）在候选食物的FoodOptimality不差于当前最佳时调用CanReach。因此食物多、较优候选不可达时，查询可能重复很多次。RimLayers 0.1.2现场两次非侵入主线程栈均命中 GetFood→FoodSearch→LayerReachPatch→LayerRouter→PathFinderJob.Execute；布尔查询调用完整Dijkstra及每段格级路径是直接热点。

0.1.3实现将层内边委托给原生CanReach，以原始TraverseParms和最终PathEndMode做楼梯端点连通搜索，复用原生层内缓存，不创建PawnPath。端点而非楼层作为访问键，允许同层借其他层绕回、反向跨层与不在最近几个之内的楼梯；方向、禁止、围栏与门权限仍由合法连接边控制。内部原生调用独立防重入，原生递归模式转换保持原样。不新增持久化或跨查询连通缓存。

实际移动与明确按真实路线成本比较的候选继续使用完整LayerRouter，路径自定义网格、原生Job/Driver和动态跨层提交复核不变。这个单Map实现不引入前文跨Map方案中的TravelJob或远程预订。可达性与完整路径有不同职责，不能从布尔连通性推导实际成本。

同进程同输入原生食物扫描：250×250三层、130候选，Core为636.6508→0.2827ms，全五DLC为822.1789→0.2949ms；完整格级路径任务均1675→0、选择保持同一真实食物。不是整帧/TPS或任意mod组合的速度结论。三层真实进食、动态断路、0→1→2→1→0、原生连续取料、牵绳/畜栏、战斗AI及重启记录见 [0.1.3验收](../Mods/RimLayers/Docs/REACHABILITY-VALIDATION.md)，失败记录也保留。
