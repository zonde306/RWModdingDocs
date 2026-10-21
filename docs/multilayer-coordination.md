# 任务仲裁、资源声明与特殊 Pawn 的接入

核对：2026-10-16，RimWorld 1.7.4980 rev60，主程序集 MVID 8e962ecc7ac742d28f3b1b16cd24d2cd，SHA-256 EFF7D97949F919FB2B151EBFB906E9BE749B63410B7DBDA78EEEBB7972B245C3。

范围：补齐早期 [foundations](multilayer-foundations.md) 中预订和 ThinkTree 的未决接口；针对当前原版方法和 Humanlike.xml / SubTrees_Misc.xml，提出最小作用范围接法。前半部分保留当时的接口调查，末节补充当前版本的实现和运行证据。单 Map 正常上下楼仍应优先保留原版 Job，详见 [queries](multilayer-queries.md)。

## 先区分两条执行路线

单 Map：普通 Job 的多个 Goto 都交给拼接路径，原版 Job/Driver/队列/预订保持；不需要每到楼梯就重派工作。只有复杂计划、异常恢复或业务子任务需要高层意图。

多 Map：正常 DeSpawn 清理后，需要由独立意图重建本地图段/业务 Job；不能假定主 ThinkTree 下一 tick 会自己找回目标。保留 Driver 的 Baal 路线是另一种实现，其兼容性和生命周期风险见 [参考](multilayer-reference-baal.md)。

以下协调服务适用于确实需要重派的阶段，不应把它扩展成“所有 Pawn 每 tick 无条件接管”。

## 当前派发链与精确优先级位置

Pawn_JobTracker.DetermineNextJob 先调用 DetermineNextConstantThinkTreeJob，有有效结果立即返回；否则调用 MainThinkNodeRoot.TryIssueJobPackage，并传递 ignoreQueue。TryFindAndStartJob 接到有效 ThinkResult 后保留 SourceNode、Tag、FromQueue 等信息启动 Job。

HumanlikeConstant 的安全/常驻逻辑包含逃避爆炸、寻找氧气、登离重力船、自动商队、敌对反应及 Lord 常驻任务。主 Humanlike 还包含倒地、着火、精神状态、玩家队列、征召/危险、Lord、紧急工作等分支。不能为了“保目标”屏蔽整棵 ConstantTree 或主树。

MainColonistBehaviorCore 在 SubTrees_Misc.xml 中定义：ThinkNode_PrioritySorter 的同级候选包括 GetFood、GetRest、药物/特殊需求、娱乐、冥想、装填和普通 JobGiver_Work。PrioritySorter 按节点 GetPriority 选择有效结果，不是简单按 XML 先后取第一个。

当前 JobGiver_Work.GetPriority 使用时间表：Anything=5.5、Work=9、Sleep=3、Joy/Meditate=2；无法工作时为 0。需要在原工作节点的位置放置自有工作适配器/包装节点，保留此优先级与业务 Tag，而不是另插一个固定高优先级续行节点。

**Humanlike_PreMain 不是普通工作续行的默认正确位置**：它位于 MainColonistBehaviorCore 之前，会绕过后者的正常食物/休息等优先级比较。Humanlike_PostMentalState 更早，也不能无条件用于普通任务强制续行。

建议普通工作包装节点在无有效意图时走原版流程；有可继续的工作意图时返回该阶段候选，同时继承原工作 GetPriority / 时间表与休息仲裁。远程食物、床和娱乐应各自在对应需求节点/查询中竞争，不把它们全部冒充工作。紧急工作分支的 emergency=true 需单独保留。

玩家明确下达的跨层命令走自己的有序意图/命令入口，仍服从取消、征召和安全规则，不依赖普通殖民者工作节点；敌人、动物和机体则按其相应 ThinkTree / JobGiver 适配，不能把 Humanlike XML 修改当成全体 Pawn 已接入。

## SourceNode、continuation 与候选所有权

CheckForJobOverride 通过 DetermineNextJob 重新获得候选，判断 ShouldStartJobFromThinkTree 和 minPriority 后才 StartJob；不接受的候选会 ReturnToPool。

ShouldStartJobFromThinkTree 对同 JobDef、非队列且 curDriver.IsContinuation(newJob) 为真时仍比较 SourceNode 与 curJob.jobGiver。JobDriver.IsContinuation 基类默认 true，所以：

- 自有 Driver 按 taskId / stage / generation 判断 continuation，不能只比较 JobDef。
- 同一意图的候选返回稳定、正确的 SourceNode，不能每次构造一个新节点导致反复打断。
- 查询候选时不提交运输所有权、不真正扣材料或绑定永久声明；被丢弃/池化的 Job 不能仍代表有效事务。
- 成功进入实际 StartJob / 预订阶段后再确认阶段绑定；用 taskId、代次和状态确认启动，不用对象哈希当持久身份。

ThinkNode_QueuedJob 会 Dequeue 能开始的任务；还可能清理不能开始的队首。跨图纯格子命令没有额外 Map 就不能安全进入原版队列。对已支持的跨层命令先转换为自有目标/阶段队列，保留玩家顺序；不要让 StopAll 清队列后才试图从已池化 Job 推测命令。

## 阶段结束与最终任务续接

EndCurrentJob 默认进行 CleanupCurrentJob 并可能自动选下一 Job；错误路径还可能先 Wait 250。到入口/某层只是路线推进，不能把每段成功都当成整个任务完成。

当前 JobDriver 提供 protected SetFinalizerJob(Func<JobCondition,Job>)；GetFinalizerJob 取得候选。EndCurrentJob 会先取得 finalizer，再清理旧 Job；startNewJob=true 且满足原版条件时启动 finalizer，并处理携物。这是自有 Driver 在**当前地图上合法完成后**续接下一业务 Job 的候选接口，可减少全局 EndCurrentJob 补丁范围。

它不是跨 Map 生命周期替代品：

- startNewJob=false 时 finalizer 不启动；Pawn 条件也可能阻止 finalizer。因此 factory 应只产生候选，不提前标记下一阶段已提交。
- finalizer 必须在当前实际地图可合法启动；不能在旧图完成时返回一个尚需到新图才能预订的原版 Job。
- factory 委托不能作为持久状态保存；由 Driver 重建 Toils/初始化时，依据已保存的任务阶段重新注册。
- 下一阶段真的启动后才确认交接；若因合法抢占未启动，则意图暂停/重新仲裁，不能无条件强制覆盖。

多 Map 的建议受控转移：

1. 自有移动阶段到入口后标记 transferPending，保存最终目标与代次；当前 Driver 进入明确等待状态，避免重复排转移。
2. 在已核实的确定性组件阶段处理票据，重新确认 Pawn、入口、出口、目标和代次。
3. 结束旧本地图段时禁止普通自动重新分派，按明确携物策略正常清理；执行 DeSpawn/Spawn。
4. 在同一次受控处理内绑定下一图的移动阶段，或在最终图创建业务 Job；整个意图未完成前不经普通工作重新选择。
5. 每个准备/已转入/已绑定阶段都可由真实 Pawn.Map、持有物和代次恢复；取消、死亡、出口失效或保存时不能复活旧票据。

这是实施协议，需要原型验证重入/保存边界；不是在 DeSpawn 清理回调里重入 StartJob。原版队列、携带物和 Lord 等状态必须按所有权明确转交，不能靠整体关闭清理掩盖问题。

## 原版预订为何不能直接用于远程预订

当前 ReservationManager.CanReserve / CanReserveStack 直接拒绝：claimant 未 Spawned、claimant.Map 不是该 manager 的 map、有效 Thing 的 MapHeld 不匹配。它们还检查实体/数量、physical interaction、ReservationLayer、MaxPawns 和 RespectsReservationsOf。不能给远程 Pawn 临时伪造 Map 后声称已完成正式预订。

单 Map 下普通 Job 可以真实预订其他层对象，通常不需要第二套远程租约。多 Map 计划或尚未启动的物流计划才需要自有 Claim/Lease。读原生状态可用公开 ReservationsReadOnly；不要直接插入私有 reservations 绕过验证和事件。

## 最小声明覆盖范围

| 入口 | 建议覆盖 | 不能做的事 |
|---|---|---|
| CanReserve | 只在原版允许后，按声明补充容量/排他限制 | 把原版 false 全部改 true |
| CanReserveStack | 将原生可用量与未转成正式预订的其他有效声明协调 | 忽略数量接口只 patch bool |
| Reserve | 在实际提交前后校验声明代次；成功后绑定/转换所有权 | 只看候选查询结果就提前占物 |
| Release / ReleaseClaimedBy / ReleaseAllClaimedBy / 目标销毁 | 清理相应本地绑定；由意图状态决定是否仍有有效全局声明 | 转移中原生清预订便无条件删除仍有效的运输计划 |
| Enroute / 储存/容器容量 | 使用原版在途/空位与自有尚未绑定声明的统一口径 | 同一运输同时计入原生在途和自有声明而双扣 |

声明至少包含 leaseId、taskId、generation、Pawn/目标、地图/层、数量或槽位、用途/ReservationLayer、状态与期限。Thing 拆堆/并堆、拿起、放下、Blueprint→Frame 转换都需重新绑定，不能继续锁住已经不存在的实体。

数量口径应区分原生预订、本 mod 尚未绑定的声明、已经映射到原生预订的同一声明；只有最后一类需要去重。maxPawns/容量与原版保持同一语义，不能用“总数量足够”替代床槽位或工作台排他。

声明按同一确定性阶段检查并提交；多资源按稳定顺序取得，失败回滚或等待，避免 A 锁木材等钢、B 锁钢等木材。期限由有效任务心跳更新，暂停、死亡、手动取消和读档重建各有清理规则；不能依靠永不释放的软锁保目标。

## 强制命令与阵营规则有特别分支

Reserve 在 playerForced 的 Job 上可再次调用 CanReserve(ignoreOtherReservations:true)，成功后抢占并结束受尊重的其他原生任务。这个强制分支直接加入 reservation，却没有走普通成功分支的 Notify_ReservationAdded；所以只监听 MapEvents.ReservationAdded 会遗漏它。

覆盖应在 Reserve 的实际成功结果处处理：玩家强制预订成功后使冲突自有意图失效/暂停，释放相应声明并更新代次；失败则不能先取消别人的有效声明。CanReserve 的 ignoreOtherReservations 与 Reserve 的 playerForced 分支应协调，不让“保目标”使玩家命令失效。

RespectsReservationsOf 不等于所有 Pawn 一律尊重所有锁：它考虑同阵营、敌对、HostFaction 等。声明层要采用相同或明确的阵营政策，不能用玩家物流租约让敌人无法攻击/占用场地。优先级不由本 mod 另造一个不透明的全局锁顺序替代。

## 牵绳、机体和 Lord 的具体约束

| 系统 | 当前源码事实 | 所需适配 |
|---|---|---|
| Pawn_RopeTracker.RopingTick | 死亡/倒地/征召/着火等会断绳；牵引者若非 JobDriver_RopeToDestination 也会断；失去 Touch 可达会断 | 单 Map 保留原牵引 Job；多 Map 的 Travel 必须明确保留牵引语义与整组过层，不能随意恢复断绳 |
| Pawn_PathFollower.RopeeWithStretchedRopeAtNextPathCell | 以 ropee.Position 到 nextCell 的原始 8 格范围限制前进 | TryEnterNextPathCell 移动后才检查；当前原型保留其落脚等待与原生 FollowRoper，运行证据见末节，不直接改短为平面距离 |
| MechanitorUtility.InMechanitorCommandRange | 要求 mech 与 overseer MapHeld 相同，再调用 CanCommandTo | 多 Map 需显式楼层组控制策略 |
| Pawn_MechanitorTracker.CanCommandTo | target 在原 Map 内且原始距离平方 <620.01 | 单 Map 也需修正控制距离，否则其他层可见却无法下令 |
| Lord.Map / AddPawnInternal | Map 来自 LordManager；禁止 Pawn 同时属于两个 Lord，并更新该 Map 的攻击目标缓存/职责 | 单 Map 保持身份但适配 duty 的几何；多 Map 用显式队伍/子 Lord 协调，不能只改 Pawn.Map 后假定控制自然正确 |
| Lord.Notify_PawnLost | 可能移除 Pawn、更新损失、触发状态转移甚至销毁 Lord | 正常过层不应误触发“已离队/战损”；死亡、被俘和真正离图仍应通知 |

这些不是任意 Pawn 已通过运行测试的保证，但关键特殊身份已定位到实际成员。动物、机体、敌方与殖民者共享路径不意味着共享任务仲裁；各自的命令/ThinkTree/能力约束仍应由相应适配器保留。

## 存档和验收边界

原版 Job/Driver/队列由原版唯一保存，自有意图与未绑定声明由本 mod 保存。运行时将 Job 与 taskId / generation 关联，读档后对照真实 Pawn.Map、当前 Job 阶段、携带物和原生预订重建，不能由两处 Deep 保存同一个 Job。

必要验收：正常中间层无重选；安全/需求抢占和玩家取消有效；同 JobDef 不同 task 不误判续行；候选被池化不留租约；多 Pawn 分数量预订、床槽位、强制命令抢占；转移中清原生预订而有效意图不丢；读档不重启两次；牵绳双方上下楼不无限等候；机体跨层控制范围正确；敌方上下楼不丢 Lord 或复制成员。以上是 2026-10-16 调研时列出的实现阶段测试；已执行项与具体范围见末节。

当前 metadata 证据：ReservationManager.CanReserve 06004558 / CanReserveStack 0600455A / Reserve 0600455B / RespectsReservationsOf(Pawn,Pawn) 0600456B / ReservationsReadOnly 属性；Pawn_JobTracker.DetermineNextJob 060044FE / TryFindAndStartJob 060044FB / EndCurrentJob 060044F3 / CheckForJobOverride 060044F9 / ShouldStartJobFromThinkTree 06004501；ThinkNode_Priority 060045CF、PrioritySorter 060045D8、QueuedJob 0600462A；JobGiver_Work.GetPriority 06005635；JobDriver.SetFinalizerJob 06004211 / GetFinalizerJob 06004212 / IsContinuation 0600422A；Pawn_RopeTracker.RopingTick 060088CB / IsStillDoingRopingJob 060088CC；RopeeWithStretchedRopeAtNextPathCell 060044A2；MechanitorUtility.InMechanitorCommandRange 060000C2 / Pawn_MechanitorTracker.CanCommandTo 060085EF；Lord.AddPawnInternal 0600484A / Notify_PawnLost 06004858。ThinkTree 原版 XML 入口为 Humanlike.xml 与 SubTrees_Misc.xml 的 MainColonistBehaviorCore。


## 1.7.4981 的机体控制接入与回验（2026-10-18）

当前基准为根 Assembly-CSharp MVID f1e6382600a84191bbc6d8a184fd6b57。前述 1.7.4980 的调研历史保留，本节仅更新已重核和实际运行的范围。

CanCommandTo（060085EF）先检查 target.Cell.InBounds(pawn.MapHeld)，再以距离平方严格 <620.01 判断。InMechanitorCommandRange（060000C2；全局 MechanitorUtility 类型）先取得真实 overseer，要求双方 MapHeld 相同后调用它。DrawCommandRadius（060085EE）只在操控者在场且选中了其已征召机体时，以原生半径 24.9 和 CanCommandTo 谓词绘制。

RimLayers 只替换前者的距离运算为逻辑 x/z，拒绝布局隔离带，不改原生阈值；后者仅在显示入口将中心投影到查看层。受管地图外走原始距离，没有全局距离函数替换。Hediff_Mechlink.PostAdd（0600218D，Verse 类型）调用 PawnComponentsUtility.AddAndRemoveDynamicComponents 创建原生 tracker；PawnRelationWorker_Overseer.OnRelationCreated（060084D9）调用原生 AssignPawnControlGroup（060085DE）。控制关系、组和 Job 仍全部由原版保存。

真实 Core+Biotech 与全 DLC 的机体植入、控制范围 24/25 边界、三层 Goto、同一 Driver、二层途中保存/新进程续行及组成员唯一性已通过；详见 [WORK-LOG](../Mods/RimLayers/Docs/WORK-LOG.md) M5。该证据不替代牵绳/Lord 验收。

牵绳当前程序集重读仍确认：RopingTick（060088CB）会因倒地、征召、火焰、原生 roping Job 不再成立或真实 Touch 不可达而断绳；RopeeWithStretchedRopeAtNextPathCell（060044A2）直接以原始 InHorDistOf(nextCell,8) 限制领队。后续运行确认原生落脚等待与 FollowRoper 可完成三层牵引，详见下节；不能仅把跨带平面距离改短后声称绳路合法。


### 指挥圈的调用与画面不能合并判定

原版 Pawn.DrawAt（06002300）调用 mechanitor.DrawCommandRadius；远层操控者被裁剪后，仅改绘制中心不能恢复这条调用。补画采用已核实的 MapComponentDraw（06001483）入口，由 MapInterfaceUpdate → MapComponentUtility.MapComponentOnDraw（0600C546 / 0600148B）调度。SelectedPawns 是可复用列表，DrawCommandRadius 又会通过 AnySelectedDraftedMechs 读取它；调用前要先快照去重后的操控者，避免嵌套查询使遍历失效。

控制/三层路径/保存和远层指挥圈画面均已通过。最小 Biotech `v1-mechanitor-20261018-165050`、全 DLC `165140` 生成 17 / 读档 15，通过的实际远层图以操控者仍在地面、查看二层、局部镜头 (54,54)、size 28 拍摄，完整边界可见。相同镜头下的原生地面和原生二层为对照。

此前“远层圈缺失”的负面结论撤回：近景 (77,54)、size 8 把圈心放在屏外，新增同镜头原生对照也不显示边界。保留原生裁剪，不据此增加全局渲染补丁。几何提交的 1941 格/196 条边和像素验收分别记录，详见 [WORK-LOG](../Mods/RimLayers/Docs/WORK-LOG.md) 显示专项。


## 1.7.4981：牵绳和 Lord 的真实运行结论

### 原生跟随可完成过层，先不要替换绳长规则

TryEnterNextPathCell（060044A9）先写入 Pawn.Position，再调用 RopeeWithStretchedRopeAtNextPathCell（060044A2）；拉绳时给领队 60 tick 冷却，而不是在旧层禁止迈出楼梯。原生 FollowRoper 以真实领队为目标并使用实际路径，原生 RopingTick 继续检查 Touch 可达与合法牵引 Job。实际双 Muffalo 的 1→2→3 与二层途中重启已通过，故 RimLayers 没有替换该绳长检查、另建旅行 Job 或恢复已合法断开的绳。

原生 PenMarkerState.HasOutsideAccess（1700032C）只检查 Enclosed / Calc().ImpassableDoors，当前唯一调用方为 CompInspectStringExtra（060099C2），不直接决定能否分配牵引工作。WorkGiver_TakeToPen.JobOnThing / MakeJob（06005689 / 0600568B）通过 GetPenAnimalShouldBeTakenTo（06001076）保留原生过滤与预订，后者的 CheckUseAndReach（06001075）用真实 roper + WithFenceblockedOf(animal) 评估未来路线。原生围护内的 FindPlaceInPenToStand（0600107C）仍生成真实目标 B。

RimLayers 为畜栏补充两项规则：可用楼梯跨越畜栏 Region 边界也算可牵引出入口；未被真实牵引且按原生 fenceBlocked 通行的 Roamer 不能单独走楼梯。Pawn.FenceBlocked（170006DF）由 Roamer / 当前 Job.ignoreFenceBlocked 决定，不是任意动物一律阻挡。人的预先查询、PassDoors 和真实牵引保持通行，畜栏仍不合并上下层 Region。

ClosestSuitablePen / PenIsCloser（06001073 / 06001078）的原始距离使用存储坐标；受管地图现以实际预期牵引路径成本排序，不改变营养平衡模式。两条真实合法路线的 Core 成本为高层 420 / 地面 836，全 DLC 为 420 / 960；原生工作候选与最近畜栏查询均选择高层目标。双动物送达、保存和征召/取消/断路的具体证据见 [WORK-LOG](../Mods/RimLayers/Docs/WORK-LOG.md) M5 末节。

### 正常过层不需要拆建 Lord

LordMaker.MakeNewLord（060048C0）负责注册、成员、SetJob 和初始 GotoToil。LordJob_Travel / LordJob_DefendPoint 位于 Verse.AI.Group，不在 RimWorld 命名空间。原生 TransitionAction_SetDefendLocalGroup.DoAction（06004950）直接选一个真实成员的 Position 作为防御点，不计算跨坐标带的平均格。原生 Travel 图的临时 DefendPoint 在无伤害 1200 tick 后会恢复 Travel；永久改派不能只改该临时防御点，应通过同一 Lord 的原生 SetJob / GotoToil（06004845 / 06004851）明确业务职责。

三个实际敌方 AncientSoldier 用原生完整 AI 经二层抵达三层，二层中途重启后 Lord ID、精确成员集合和最终职责保持；真实伤害触发原生防御转移，持久防御改派后全员实际返回一层。正常过层不移除成员、不累计暴力损失；实际死亡继续经 Notify_PawnLost（06004858）移除成员并累计一次损失。Core 与全 DLC 各生成 13 / 重启 13 项通过，RimLayers 当前没有 Lord 生产补丁。该结论针对实际运行的 Travel / DefendPoint，不扩展成所有 DLC 仪式、商队和攻城 Toil 已验收。


## 1.7.4981：单 Map 步行离图和加载后的 Toil

私有 ExitMapGrid 只解决一部分边界：家园 MapUsesExitGrid=false，而 JobDriver_Goto 最终到达还用 Position.OnEdge(map)。RCellFinder 的 Best/Random 出口直接按 Map.Size 取样和投影，Near 委托 CellFinder.TryFindRandomEdgeCellNearWith；CanReachMapEdge 原生只遍历 Region 并测试 District.TouchesMapEdge，不认识楼梯边。单 Map 坐标带应定向接入这些世界边界入口，不能全局改 Map.Size 或把所有 OnEdge 都解释为楼层边缘。

普通步行提交方包括 Goto.TryExitMap、JobDriver_Flee.ExitMap、TakeAndExitMap 的离图委托；均需用逻辑地面计算到达与出口方向。Pawn.ExitMap 本身还被飞行、Farskip、运输和商队调用，不可用无条件“仅地面”前置补丁阻断它们。保持原生伤员、携带、Lord/阵营/任务通知和一次性世界提交。

家园普通 Goto 不创建商队；CanExitMapAndJoinOrCreateCaravanNow 先要求 MapUsesExitGrid，后者还受 Pocket、真空和 FormCaravanComp 等规则影响，并有单 tick 缓存。家园实际组队由 StartFormingCaravan / LordJob_FormAndSendCaravan 驱动，Leave 阶段到出口附近后调用 FormAndCreateCaravan。CaravanMaker 可能已先移除地图/Lord 身份；在该方法开始时观测到达，而不是错误要求后续 Pawn.ExitMap 入口仍 Spawned。

读档时 Driver 重建 Toil 可早于 MapComponent.PostLoadInit 的布局恢复。若在 MakeNewToils 判断 Managed 后才包装到达检查，加载后的 Toil 会永久漏装。可定向替换实际执行委托的几何调用，在执行时读取布局，保留原生 Toil 和仲裁；不需要再存一套 Job。RimLayers 的真实三层 NPC 撤离、原生家园商队及二层中途重启证据见 [WORK-LOG](../Mods/RimLayers/Docs/WORK-LOG.md)。
