# IsoRim ProjectZ / IsoRim Waylith：单 Map 多楼层实现参考

核对日期：2026-10-16。当前游戏为 RimWorld 1.7.4980 rev60，主程序集 SHA-256 EFF7D97949F919FB2B151EBFB906E9BE749B63410B7DBDA78EEEBB7972B245C3，MVID 8e962ecc7ac742d28f3b1b16cd24d2cd。

范围：只读用户指定的两份本地源码、元数据、Def 和少量测试入口，定向复核当前原版寻路成员。没有构建、启动游戏、执行作者测试或更改参考项目；源码分析不证明随附 DLL 与源码逐字一致，也不证明所有功能已经可用。总体方案见 [multilayer.md](multilayer.md)，另一种真实多 Map 实现见 [Baal.SecondFloor](multilayer-reference-baal.md)。

## 样本与职责

| 项目 | IsoRim ProjectZ | IsoRim Waylith |
|---|---|---|
| 本地目录 | Mods/IsoRim-ProjectZ | Mods/IsoRim-1.7 |
| About packageId | Peren.IsoRimProjectZ | Peren.IsoRim17 |
| About 版本 / 依赖 | 1.7；Harmony、Peren.IsoRim17 | 1.7；Harmony |
| Git HEAD | 66db5b6，2026-10-16 | 1f8f5c2，2026-10-15 |
| 检查时工作树 | git status --short 无输出 | git status --short 无输出 |
| 项目 | Source/IsoRimProjectZ.csproj，net10.0 | Source/IsoRim.csproj，net10.0 |
| 随附 DLL 大小 | 484352 bytes | 561664 bytes |
| DLL SHA-256 | A399156B7451945289735FBDBA4744F170D09918D2CA3EE5FA45FF48C7F789E5 | 3740005D6C22A60B02D25A6E169754913887F5B89006620ACB91F2EEFB848050 |

ProjectZ 的 LoadFolders 对 1.7 加载根目录、1.7 目录，并按 IsoRim 包是否启用加载 IsoRim 资源目录；IsoRim 加载根目录和 1.7。ProjectZ 负责楼层游戏规则；IsoRim 提供三维绘制和交互，并通过 IFloorLayout 接受游戏模组的楼层布局。不是两套独立的跨图工作系统。

ProjectZ README/About 明确标为开发中的原型，不适合普通游玩；有楼层的存档不能安全移除。IsoRim 本体的“纯视觉、可移除”说明不能套到 ProjectZ 上。

两个根 LICENSE 均提供 MIT 授权；实际复制或再分发应保留相应版权与许可声明，并另查使用资产的来源。本次没有复制代码或资产。作者在 README 注明路径拼接思路来自 MigCorp - Skiptech，并声明未使用其代码。

复现构建前注意：ProjectZ csproj 的 IsoRimDll 默认指向相邻的 IsoRimModelsWT/1.7/Assemblies/IsoRim.dll，本机该路径不存在。应显式传入本次 IsoRim-1.7 下的 DLL 路径，先确认其中具有 Api / IFloorLayout；不要因 About 依赖正确就认为项目默认引用也正确。

## 核心结构：一张扩大的 Map，多个坐标带

这两份代码所展示的是“单 Map 多楼层”。视觉上楼层重叠，存储上都在同一张 Map，Thing.Map 不随上下楼改变。不存在把多个原版 Map 直接叠起来就自动获得交互的机制。

ProjectZ.cs 的 AnnexMapComponent 保存 groundZ、stride、floors。地面层编号为 0；楼层 k 的存储格为：

```text
stride = ceil((groundZ + 1) / 17) * 17
storageCell = (x, 0, localZ + k * stride)
logicalCell = (floor = k, x, localZ)
map.Size = (width, 1, floors * stride)
```

每个坐标带之间至少留一行不可通行空域，stride 对齐 17 格 Section，宽度不变所以地面原有格子索引保持为前缀。GroundCellOf / CellOnFloor / FloorOf 统一转换；间隔行的 FloorOf 为 -1。不要将这当成原版 IntVec3.y 支持楼层。

ProjectZSettings 默认总共 2 层，最多 5 层，生成地图时一次预留全部楼层。250×250 地面对应 stride=255，5 层分配 250×1275=318750 格，为原地面格数的 5.1 倍；这只是格子数量计算，不是测得的内存或耗时倍数。

MapGenPatches 的过程是：先按全部楼层分配组件网格；暂时将 Map.info.Size 缩为地面尺寸供生成步骤使用；FinalizeInit 前恢复真实尺寸并填充空域。GridFixer 通过反射修复部分按临时尺寸创建的网格，河流流向数组因索引布局不同另行重排。此流程是生成兼容性的主要成本，有限深度的反射修复不能保证覆盖任意第三方私有缓存。

## 绘制、点选与坐标契约

IsoRim 的 [Floors.cs](../Mods/IsoRim-1.7/Source/Floors.cs) 声明 IFloorLayout：GroundRows、FloorCount、Floor(height, zShift)、PointerFloor 和 ThingOffset。ProjectZ 的 [IsoRimBridge.cs](../Mods/IsoRim-ProjectZ/Source/IsoRimBridge.cs) 注册 FloorLayout，锁定三维视图与墙高，并隐藏用于游戏规则的楼梯井屋顶。

Section、动态物体、选中框、拖拽和建筑预览按存储行应用位移：z 减去该层偏移，y 加上显示层高。可隐藏当前层以上的内容，并通过 ThingOffset 绘制爬楼和跳落动画。仅给地形做位移不够，所有绘制入口需要共享同一转换。

PointerFloorAt / ScreenToMap 将鼠标射线与楼层平面相交，再加回存储偏移。持有建造等 Designator 时指针固定在查看层，允许点空域铺楼板；普通选择和命令可透过空域落到下面有楼面的层。Interaction.cs 还负责实体拾取与高物体交互。

必须区分存储坐标、逻辑楼层坐标、渲染坐标。ProjectZ.Virtual 提供游戏几何坐标；战斗以墙高 1 为单位（Storey=1.02），显示墙高固定 1.65、层高 1.67，属于不同单位。复用时应明确换算，不能把两种高度直接混算。

## 寻路与目标保持

[PathPatches.cs](../Mods/IsoRim-ProjectZ/Source/Patches/PathPatches.cs) 的 StairRouter 将路线分成层内普通路径和跨层连接。层内调用 FindPathNow，保留原版门、代价和避让参数；楼梯斜坡使用固定节点，楼梯跨带一步连接两个不相邻的存储格。Join 最终生成一条 PawnPath，Destination 与 PathEndMode 始终是最终目标。

GenerateNewPathRequest 后置补丁取消刚提交的原生请求，生成并立即 Resolve 拼接结果；FindPathNow 的同步调用入口也接入相同路由。原版需要新路径时重新经过该入口。因此 0→1→2 可以作为同一个 Job 中的一次 Goto 执行，经过中间层不会主动派发普通工作。

本轮复核当前 1.7.4980 的 Pawn_PathFollower.TryEnterNextPathCell（metadata 060044A9）：在阻挡、碰撞和门处理后直接赋值 pawn.Position=nextCell，此方法没有先拒绝非相邻格。源码正是利用这个行为；它不是原版承诺的传送接口，其余跟随器校验和 mod 补丁仍需实测。SetupMoveIntoNextCell 补丁给楼梯/跳落单独耗时；移动后只重置 Drawer.tweener，不调用会结束 Job 的 Pawn.Notify_Teleported。

同一个 Map、同一个 Pawn、同一个 Job，携带物及原版 Map 内预订不需要每上一层就进行 DeSpawn/Spawn 和重建。不能把跨 Map 方案的 Travel Job 与分段重派协议机械套到这里。

保存中途跨带移动时，Pawn.PostMapInit 补丁将非相邻 nextCell 重置为当前格，使原版恢复流程重新求路；并不是将整条 PawnPath 另存一份。取消、目标毁坏、需求抢占仍遵守 Job 生命周期；保留 Job 不代表任何情况下都不得中断。

## 可达性、房间与路径搜索边界

[ReachPatches.cs](../Mods/IsoRim-ProjectZ/Source/Patches/ReachPatches.cs) 在查询时向 Region 搜索补充楼梯伙伴，以及向下跳落的单向伙伴，不把它们永久写入 region.links。构建 District 的两个 flood 入口用 suppress 禁止这些连接，避免不同楼层被合并成同一房间；普通区域搜索则可以发现跨层工作目标。

跳落是单向边，不能复用原版把两个 District 排序后的对称缓存结果。DirCache 使用有向键并纳入原版缓存清理；具体私有字段和位标记约定属于版本耦合点。

现有 StairRouter 仍是有剪枝的递归试探：

- StairCandidates 按平面距离排序，每步最多保留 3 个；找到一条成功路线便返回，不保证最优，也可能漏掉第 4 个才通的楼梯。
- 不同目标层时主要只朝目标层方向走；同层路径失败才尝试下楼再上楼或反向绕行。若去更高层必须先下楼绕到另一栋建筑，这个约束可能漏路。
- depth 受总楼层数限制，不能代表任意拓扑下允许的连接次数。端点循环、同层多次绕行不能只用层数界定。
- Reachability 可看到更多连接，但路径路由有额外剪枝，可能出现“可达查询为真、路径失败”。作者 TestDeadEnd 覆盖较近死路楼梯与另一条可行楼梯，不代表所有此类拓扑已解决。

上述漏路属于代码条件推导，未在本机运行复现。建议保留同一 Job + PawnPath 拼接，改用统一的端点有向图搜索；以端点而非仅层号判重，失败缓存带拓扑版本，使可达与实际求路共享通行规则。不能全局替换 IntVec3 距离：它仍被存储网格、相邻判断和本层范围使用。

## 物品、家具、工作台与多次取料

单 Map 方案的直接收益是：各层物品、床、工作台和仓库都进入同一 Map 的索引；Job 可以一直持有真实 Thing 和目标队列，普通原版预订也作用于同一管理器。它减少了跨 Map 候选查询与预订迁移问题。

但索引可见、可达、被选中和任务完成是四件不同的事。有限搜索半径、原始存储距离、区域数量上限、账单选料范围与工作评分，仍可能忽略其他楼层或产生不合理排序。VirtualPatches 明确采取逐个几何决策修正的做法，并未提供通用工作台/材料搜索的空间语义。

用户的“一层材料、三层工作台、多次取料后回台”应优先尝试让同一个原版 DoBill Job 连续执行多个 Goto，由拼接路径承接每次过层；前提是原版选料能找齐需要的对象，且每个 Toil 的距离与相邻假设都成立。需要补给物流或复杂工序时，仍可使用 [任务意图与阶段协调](multilayer-routing.md)，但正常上下楼不必结束当前 Job。

本次看到 TestM1b 有跨层搬钢材、上层床睡眠的断言；TestDeadEnd 有搬木材上楼造床；TestSave 与 PathPatches 涉及读档恢复。测试源码存在不等于测试已通过。本次未取得工作台多层往返的运行证据，不能由床和搬运场景推断全部工作自动兼容。

## 屋顶、楼板、拆除与坍塌

[Stairs.cs](../Mods/IsoRim-ProjectZ/Source/Stairs.cs) 的 PlaceWorker_UpperFloor.Check 调用 CoverOk 和 Supported；后者沿屋顶/计划楼板的连通格检查附近 holdsRoof 建筑，使用原版屋顶支撑距离。GenConstruct.CanConstruct 补丁在实际施工阶段再检查支撑。厚岩顶、楼梯井和下层过高物体另有限制。

关键差异：它要求“下层可以支撑屋顶”，没有要求该格预先存在已建成屋顶。楼板建成后，FloorRoof.Changed 在下层写入 PZ_FloorRoof；拆掉楼板后恢复 RoofConstructed。它实现的是“楼板就是下层天花板”，不是用户要求的“下层先有屋顶，才能造上层”。

若保持用户规则，应在放置、施工开始/完成和支撑变化时，统一检查紧邻下层的屋顶类型及可承重状态；未完成屋顶蓝图不得算有效屋顶。保留普通屋顶与结构楼面的关系或将其合并，需在保存格式和拆除规则中明确。不要悄悄沿用 ProjectZ 的自动补顶规则。

[Slabs.cs](../Mods/IsoRim-ProjectZ/Source/Slabs.cs) 已有值得借鉴的完整流程：

- 空域不可通行且只有楼板专用 affordance；结构楼板本身 layerable=false，普通装修层可另铺其上，移除结构层显式恢复 PZ_OpenAir。
- 正常拆除阻止上方建筑、蓝图/框架、楼梯落点及更高楼层；按到楼梯的行走距离从远到近拆，避免先切断施工通道。
- FloorRoof 保护楼板对应屋顶不被普通拆顶或部分即时落顶入口移除；承重失效的屋顶坍塌则联动上方楼板。
- SlabCollapse 处理上层建筑破坏、蓝图取消、区域清理、Pawn 与物品坠落，设置空域并更新相关索引；正常拆除的 MoveOff 与事故坠落区分处理。

这比“放蓝图时检查一次屋顶”更接近完整生命周期。仍需验证多层连锁坍塌、跨格建筑、携带 Pawn、拆除同时改路及绕过 RoofGrid.SetRoof 的写入；原版 RemoveRoofUnsafe 风险见 [foundations](multilayer-foundations.md)，本轮没有证明该样本已覆盖所有旁路。

房间保持各层独立；FloorHeat 按上下层共有楼板格补充温度交换，并修正原版薄屋顶向室外散热部分，缓存随屋顶、地形、房间变化失效。垂直温度是专门规则，不会因共享 Map 自动形成。

## 跨层战斗与目标选择

[Combat/Heights.cs](../Mods/IsoRim-ProjectZ/Source/Combat/Heights.cs) 与 [CombatPatches.cs](../Mods/IsoRim-ProjectZ/Source/Combat/CombatPatches.cs) 提供了比仅放宽 CanHit 更完整的参考：

1. 从同 Map 的 attackTargetsCache 枚举其他层候选；按敌对、威胁、validator、Lord、迷雾和必要可达性过滤，以逻辑平面距离判断射程。
2. 高处一端必须位于面向另一端的楼板边缘，或可越过的低矮掩体边缘；楼板内部不能随意俯射。射线检查各层楼板/屋顶平面和墙体，洞口与楼梯井需要特殊规则。
3. 比较跨层候选与原版同层选择的分数，修正存储距离和高度掩体影响；若当前位置已经能射击，CastPositionFinder 可返回当前格。它没有由此提供完整的“寻找跨层最佳射位”算法。
4. ShotReport 使用实际水平距离与高度掩体，高射低可加射手命中因子（默认 +0.2，封顶 1）；旋转朝向和 AI 目标保留距离也有适配。
5. 弹丸将起点平移到目标楼层坐标带，按真实平面距离重算飞行 tick；额外高度状态检查飞行途中楼板平面、所在层实体和友伤。打偏落到空域会继续下落，部分高空炮弹命中也有向顶层重定向。

这证明“候选、射程、视线、掩体、命中报告、实际弹丸、朝向和 AI”需要一起处理。视觉叠层只解决显示与拾取；同 Map 也不自动产生正确三维弹道。当前实现采用平面射程和离散高度模型，不应称为原版完整三维物理。

明确限制：Flights.table 是未序列化的 ConditionalWeakTable，代码注明在途弹丸读档后退回普通弹丸；CombatGate 检测到 Combat Extended 后关闭本 mod 的这些战斗补丁。这是避让冲突，不能视为 CE 已支持跨层。后续已在 [战斗专题](multilayer-combat.md) 独立审计爆炸传播、近战提交及两类护盾的关键入口，样本仍需相应实现；不把普通同 Map 爆炸误当成上下层都会受影响。

## 后续当前原版审计

新增 [工作查询、连续取料与动态路径审计](multilayer-queries.md)：已把原先“可能存在距离限制”细化为当前 1.7.4980 的具体方法和条件。包括普通格子扫描 99999 距离平方上限、账单三处半径、食物评分、床/娱乐/储存规则、DoBill 原生多次取料队列、Direct 放料同层过滤，以及旧路径在连接器删除后需独立校验。数值反例只做静态复算，未运行本 mod。

新增 [跨层战斗落地审计](multilayer-combat.md)：进一步核对原版射位范围、目标保持、近战实际伤害、两类护盾、弹丸和爆炸保存。IsoRim Projectiles.cs 已有独立跨层绘制高度缓存，但需与游戏轨迹采用共同持久快照。当前格能射击之外的射位搜索、范围盾三维几何和爆炸逻辑仍需补齐。

## 方案选择与下一步验证

| 维度 | 单 Map 坐标带（ProjectZ） | 一层一 Map（Baal / 原总体方案） |
|---|---|---|
| 上下楼保 Job | 保留同 Job/Driver，以路径表达过层 | 要处理离图生命周期及意图/Job 续接 |
| 普通预订与物品索引 | 同 Map 可复用，仍有距离/半径限制 | 需要远程查询、声明与到图预订 |
| 现有地图与容量 | 新图预分配所有层；网格面积增长 | 可逐层建图，消耗 Map 数量与管理器 |
| 空间语义 | 大量原版距离/边界依赖要适配 | 层内坐标自然；跨图动作要适配 |
| 叠层显示与点选 | 现有 IsoRim API 是直接参考 | 不能直接套 IFloorLayout，需额外地图绘制方案 |
| 建造与战斗 | 已有支撑、坍塌、射击原型 | Baal 样本没有同等完整的直接跨层射击 |
| 兼容成本 | 网格生成、私有字段、查询入口与渲染 | 地图生命周期、工作扫描、任务和跨图战斗 |

应调整此前“一层一 Map 为唯一推荐”的表述。针对固定楼层、连续原版 Job、屋顶建造和俯射的需求，单 Map 方案值得优先验证；若目标强调普通 Map 独立生命周期、按需建层或不接受这种坐标与渲染侵入，则保留一层一 Map 路线。当前证据不足以无条件定案。

建议先做同一组独立测试，之后再决定正式存储架构：

- 三层路线；同层绕行；去高层前必须先下层绕行；第 4 个楼梯才通；中途楼梯被毁；可达与实际路径结果一致。
- 一层食物/材料、三层床/工作台，有限选料半径、连续多次取料、多人抢料、途中取消与保存重载。
- 严格“下层已有屋顶”规则，屋顶拆除、柱子破坏、正常拆楼板和多层连锁坍塌；空域不产生沙土地面回退。
- 高处边缘选低处目标、楼板遮挡、低掩体、敌方 AI、在途弹丸存读档；单独确认 CE 支持范围。
- 250 地面 × 2/3/5 层，测地图生成、峰值内存、路径请求耗时、tick 与绘制；同时覆盖河流、随机落点和边界事件。

这些是待执行验收，不是本轮通过结果。参考项目和随附 DLL 保持只读，玩家存档、设置和启用列表没有更改。
