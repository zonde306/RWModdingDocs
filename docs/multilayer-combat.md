# 跨层战斗落地：目标、射位、命中、护盾与爆炸

核对：2026-10-16，当前 RimWorld 1.7.4980 rev60；主程序集 MVID 8e962ecc7ac742d28f3b1b16cd24d2cd，SHA-256 EFF7D97949F919FB2B151EBFB906E9BE749B63410B7DBDA78EEEBB7972B245C3。

范围：当前原版目标/射位、弹丸、近战、护盾和爆炸的定向方法体，以及用户给出的 [IsoRim / ProjectZ](multilayer-reference-isorim.md) 源码。下面的服务、保存格式和算法是实施建议，不是现成原版 API；该调研阶段没有构建或游戏运行结果，2026-10-18 的原型增量与实测见末节。共同坐标约束见 [查询审计](multilayer-queries.md)，总体设计见 [multilayer.md](multilayer.md)。

## 玩家选择目标与实际攻击分开处理

无需为“高处瞄准低处”把实体复制到射手楼层。单 Map 下可保留目标真实 Thing 与存储格；多 Map 下目标必须带 Map/稳定楼层身份。渲染/鼠标把屏幕位置转换为真实目标，战斗几何把射手与目标转换为逻辑三维位置。显示地图、射手地图和目标地图的职责仍需分开。

命令应保留目标引用，允许查看别层；下达与实际每发开火前都检查实体、敌对/强制攻击规则、武器可用性、射程、视线和友伤。玩家切换查看层不改变 caster、清空命令或将同局部格子的其他 Thing 偷换成目标。目标移动或换层后重新验证；同一位置不代表同一目标。

用户没有要求必须采用某种显示风格。可选切层点选、半透明下层或完整三维叠层；IsoRim 已提供后一种可参考的绘制与拾取接口，但它的 IFloorLayout 专为同一 Map 坐标带设计，不能直接承诺多 Map 绘制兼容。

## 当前样本缺少“尚不能打到时寻找射位”

ProjectZ 的 AttackTargetFinder_Best_Patch 在跨层候选中要求 verb.CanHitTargetFrom(s.Position,target) 成功。因此它扩充的是“从当前格可直接命中的其他层目标”，并没有搜索未来站到边缘后才能命中的目标。

其 CastPositionFinder_Patch 也只在当前格已经能打时返回 caster.Position；否则回到原版逻辑。当前原版 Verse.AI.CastPositionFinder.TryFindCastPosition：

- 以 req.target.Position 的存储坐标构建目标射程矩形。
- 将 caster / locus 的存储范围与其求交，按 Region 预算筛选。
- EvaluateCell 再做原始距离、禁区、可达、目标命中、目的地预订和危险检查。
- CastPositionPreference 用原始距离与二维掩体评分。

一般敌人的 JobGiver_AIFightEnemies.TryFindShootingPosition 通过该入口，maxRangeFromTarget=verb.EffectiveRange；其他防守/能力类有覆盖。因此仅放宽跨层 CanHit 并不能让敌人走到楼上边缘或合理选择另一层射位。

JobGiver_AIFightEnemy.FindAttackTarget 使用 NeedReachableIfCantHitFromMyPos 等 flags；ShouldLoseTarget 仍要求 Touch 路线可达和 keep radius。ProjectZ 对后者修正距离，却保留 CanReach 条件。一个可以隔层射击、但没有步行路线到敌人的射手，仍可能失去目标；若希望保留这种远程交火，需要明确定义 acquire/keep/追击三者的不同条件。

## 建议的目标与射位协议

拟议 CombatQuery 返回状态：CanFireNow、CanMoveToFire、CanPursue、Unreachable、Forbidden / Invalid；不要把“当前不能射击”直接等同“不是战斗候选”。

1. 从允许感知的层枚举敌方候选，保留原版威胁、隐身、迷雾、非战斗人员、Lord / duty、武器与自定义 validator。敌人不能凭查看层或全局物体索引获得所有未知目标。
2. 先检查当前格的射程与三维视线；可命中则评估保持当前格，通常避免无谓跑动。
3. 对需要找射位的目标，在射手可用楼层上枚举候选：楼板边缘、开口、掩体格和允许的相邻层端点；将每层逻辑射程包围盒转换到本层存储带，分别搜索。
4. 对每格检查有效楼面、Pawn 能力/区域限制、路线、危险、目的地预订、武器最小射程和精确三维射线。履行原版的 maxRangeFromCaster、locus、maxRegions 和 validator，不能只保留视觉射程。
5. 按威胁、射击收益/掩体、被反击风险、移动与楼梯等待成本排序，输出目标 + 射位 + 路线。缓存按几何/拓扑版本失效；保留目标与射位身份，避免双方一移动就反复上下楼。
6. 到位和每发开火前重新验证；失去 LOS 可寻找新射位。无步行路线但能射击不必自动判失败；必须追击时则要求目标相关路线可达。保持原版超时、危险和玩家取消仲裁。

候选预算只能作为分帧/继续搜索状态，不能把“本帧没搜完”缓存成永久不可达。若要优化，可先当前层、再邻层，但应明确搜索完整性和收益停止条件。

## 共用三维几何，保留不同武器契约

逻辑空间定义为 (x, height, z)，楼层高度与楼板厚度统一。射线从枪口/眼高到目标可命中体积；视线和实际弹丸使用同一碰撞几何，但目标可见并不保证武器能够射击。近战、短距离能力、曲射、爆炸和穿透分别声明规则。

建议射线/线段沿逻辑格柱遍历，精确计算楼板水平面和墙/建筑体积的相交参数，返回最近碰撞：类型、层、格、Thing、交点和法线。跨多层时按线段时间顺序处理，不能简单按楼层编号先后命中。洞口不生成楼板碰撞面；普通屋顶与结构楼板的遮挡在同一几何服务里解释。

ProjectZ 的 Heights 用墙高归一单位、Bresenham 格柱和插值高度检测楼板/屋顶/墙，并对高处一端加“必须朝外沿边缘”的规则，是可用原型参考，不应当成任意模型的精确三维体积碰撞。

“高处必须在边缘”“高度是否影响射程/精度”“弹丸是否可穿楼板”属于产品规则，用户尚未逐项指定。建议初始规则为楼板和完整屋顶阻挡直射；洞口可通过；射程沿用明确的水平距离，附加高度参数。参数应集中配置，避免选目标和实际命中采用两套规则。

当前 Verse.Verb.TryFindShootLineFromTo 对 IsMeleeAttack 或 EffectiveRange<=1.42 的分支直接调用 ReachabilityImmediate.Touch，先于普通远程 LOS；不应将此分支无差别改为三维直射规则。原版其他 Verb override 和能力需按名单审计，CE 使用独立适配而不是关闭本 mod 补丁后宣称兼容。

## 弹丸：轨迹、命中与绘制只有一个事实来源

当前 Projectile.TickInterval(delta) 取本 tick 前后 ExactPosition，交给 CheckForFreeInterceptBetween，再移动 Position 并在到达时 ImpactSomething。因此几何处理必须覆盖整段位移，不能假定一次回调等于 1 tick，或只检查终点格。

ProjectZ 把弹丸起点平移到落点层的存储带，保持水平飞行成本，再用 Flights.Flight.hs / ht / last 检查途中高度。这个办法可沿用作存储投影，但额外逻辑状态必须持久化。

Projectile.ExposeData 已保存 origin、destination、ticksToImpact、usedTarget、intendedTarget、launcher、equipment、命中标志、landed、lifetime 等。ProjectZ 的 Flights.table 是未序列化 ConditionalWeakTable，读档后缺失高度；单靠原版字段无法恢复完整发射时的逻辑高度事实。

IsoRim Projectiles.cs 另外有绘制用 Flight 缓存：记录枪口高度、目标高度、launcherRow / targetRow，插值跨层显示高度。这说明它已有跨层弹丸绘制，不应误判为只把弹丸画在目标层；但绘制与 ProjectZ 游戏碰撞采用两份数据。绘制缓存 Begin 在首次绘制时读取 launcher 当前位置，重载后也会重建，不适合充当固定的发射时快照。

建议为每颗需要跨层语义的 Projectile 保存以下自有状态，引用 Projectile 本身仍由原版保存：

| 数据 | 用途 |
|---|---|
| dataVersion、逻辑发射点/高度、逻辑落点/轨迹参数 | 发射时固定的事实，不从射手后来的楼层猜测 |
| 当前轨迹参数/飞行进度及上一个逻辑点 | 重载与 delta>1 的连续扫掠 |
| 弹丸存储投影所在层、实体目标引用及命中/友伤策略 | 投影映射与原版命中契约 |
| terminal / impactCommitted 状态 | 防止同一碰撞被原版及自有流程处理两次 |
| 必要的延迟引爆阶段 | 区分飞行、已落地等待、已爆炸 |

可使用选定 Projectile 子类/Comp，或对已有弹丸 ExposeData 附加受作用域限制的保存入口；普通缓存 CWT 可继续作为运行时索引，但不是持久化来源。读档后重建索引，不能重调 Launch 或重掷散布随机数。

显示应读取同一个逻辑轨迹快照再换算到显示单位；枪口、火花、尾迹、阴影与命中位置也同步。只恢复碰撞而让绘制重新猜发射层，会产生看见子弹穿板却没有命中的错位。

## 实际命中沿用原版伤害管线

Bullet.Impact 使用原版 DamageInfo、武器品质、护甲穿透、额外伤害、战斗日志及 TakeDamage。跨层碰撞确定命中对象后应进入相应 Projectile 的一次原版 Impact，保留这些契约；不能自己 TakeDamage 一次又让原弹丸继续飞到目标再伤害一次。

Projectile_Explosive.Impact 可能即时 Explode，也可能设置 ticksToDetonation 后 landed=true。其 ExposeData 保存引爆倒计时；不能把所有命中都简化成 Destroy 后自建一次爆炸，丢失延迟、武器及目标信息。

撞上楼板/墙时确定实际逻辑命中层和格，之后才提交 Impact / 爆炸。终结状态必须在调用有副作用的命中流程前确定，处理回调中目标被毁、地图切换或二次触发的情况。保存/读档不能再次重放已经提交的伤害。

## 两种护盾是不同接入点

RimWorld.CompShield.PostPreApplyDamage 在 Pawn 的伤害管线内处理 EMP、远程/爆炸类型和能量吸收。沿用正常 TakeDamage 即可经过个人护盾；不要跨层命中时绕过 Thing/Pawn 正常伤害入口。

范围护盾 CompProjectileInterceptor 不同：

- Projectile.CheckForFreeInterceptBetween 先枚举 Map 中 ProjectileInterceptor，再调用 CompProjectileInterceptor.CheckIntercept；之后才执行格子的 CheckForFreeIntercept。
- CheckIntercept 使用 parent 的存储 x/z、半径和线段/圆周相交判断，没有跨层高度几何。
- 因此 ProjectZ 对 CheckForFreeIntercept 的高度补丁不足以覆盖范围护盾；弹丸仍可能被投影层护盾误拦，或错过实际飞经楼层的护盾。

需要定义护盾是单层圆、有限高度圆柱还是穹顶/球体，再在同一逻辑线段上求首次相交，同时保留敌我、入射/出射、高空弹、EMP、充能和耐久逻辑。结果应与楼板/墙等碰撞按先后次序竞争，不能为了让护盾工作先拦截后再判断更早已撞墙。

## 近战不能只信候选阶段的范围

当前 Verb_MeleeAttack.TryCastShot 在 CanHitTarget 失败时仅写出“from out of melee position”警告，随后仍可能执行近战伤害。这个事实不等于原版通常会隔空近战：通常上层 Job/Verb 负责保证触达；但扩展路径和楼层以后，必须在实际攻击提交处维护这个不变量。

默认只允许同层合法 Touch 近战，或通过明确注册的楼梯相邻交战边；站在上层同局部坐标不能隔楼板砍下层。攻击蓄力期间目标换层、连接器拆毁、倒地/被携带或楼面坍塌时重新验证，失败应重新追击/结束攻击，不仅记录日志后继续伤害。

沿楼梯跨边交战是否允许是待选规则。若允许，应有攻击双方的逻辑位置/高度和连接器状态，不能直接把两端数百格的存储距离一律视为邻接。

## 爆炸：overrideCells 不能单独解决跨层

当前链条为 GenExplosion.DoExplosion → Explosion.StartExplosion → DamageWorker.ExplosionCellsToHit / ExplosionAffectCell。原版使用存储平面半径和二维 GenSight.LineOfSight。空域地形不可走不代表阻挡视线：GenGrid.CanBeSeenOverFast 只检查格上的 Edifice 是否挡视线。

所以单 Map 坐标带存在两种相反风险：正常对齐楼层相隔数百格，爆炸无法自然传播；相邻带的两端却可能只有数格，爆炸半径可穿过空域间隔误伤逻辑上远处的另一层。ProjectZ 已读战斗代码没有为此建立爆炸楼层几何。

Explosion.StartExplosion 支持 overrideCells，但其他逻辑仍取相对爆心的原始距离：GetCellAffectTick、GetDamageAmountAt、AffectCell 的排除半径/气体/起火，以及 DamageWorker 的效果/融雪等。仅将远层格加入列表会导致到达时间、衰减和副作用错误；applyDamageToExplosionCellsNeighbors 也需要楼层边界约束。

建议建立一个逻辑 ExplosionEvent：

1. 保存逻辑爆心、高度、半径、伤害/归属、爆发时间、传播规则和事件 ID。
2. 在逻辑几何里决定各层受影响格及到达时间；楼板默认阻挡，洞口可传递。是否因本次爆炸破坏楼板而继续传播，要定义同一事件的顺序，不能靠副作用偶然决定。
3. 映射到存储格后复用原版 DamageWorker / 伤害入口，但到达时间、衰减和效果位置都从同一逻辑距离获得；为特殊 DamageWorker 保留适配名单。
4. 在一个事件内共享受伤 Thing 的去重集合，覆盖跨格建筑及爆炸传播期间移动的 Pawn；不同事件可以各自伤害。
5. 起火、气体、推力、地形、视觉和声音分别提交到实际受影响楼层，避免每层子爆炸重复触发全局通知和声音。

如果采用“每个受影响层一个局部 Explosion”重用原版，也必须共享根事件和去重/传播状态，保留高度衰减；简单在每层复制完整半径爆炸会穿透楼板、放大伤害和副作用。初版可以明确只支持同层爆炸并严格过滤坐标带泄漏，但这应列为功能阶段限制，不能声称已经完成三维爆炸。

当前 Explosion.ExposeData 保存 cellsToAffect、damagedThings、ignoredThings、startTick 等；跨层事件也应保存已处理格/对象和待传播状态，读档不能重新 StartExplosion 清空历史并重伤一次。

## 补丁/服务接口与验收

| 子系统 | 需要的统一能力 |
|---|---|
| Targeter / 鼠标 | 返回真实 Thing 或显式楼层格；验证与显示分离 |
| AI acquire / keep | 区分可立即射击、可移到射位、需追击，保留 duty / 威胁规则 |
| CastPositionFinder | 多楼层候选与逻辑范围；路线、危险和目的地预订 |
| Verb / ShotReport | 同一射程/LOS/掩体模型；支持名单和实际开火复核 |
| Projectile | 发射时逻辑快照、扫掠、首碰撞、保存和一次命中 |
| Interceptor / Shield | 范围盾几何接入；个人护盾走原版伤害管线 |
| Melee | 提交时触达；连接器与楼面变化失效 |
| Explosion | 逻辑传播/衰减、受伤去重、所有副作用及保存 |
| Renderer | 轨迹快照换算，拾取/命令/枪口/命中位置一致 |

必要验收：高层边缘俯射/低层仰射；中间楼板遮挡和洞口；从楼板内部走到边缘再开火；无步行路线的远程交火；目标在蓄力/飞行中换层；友伤、倒地目标、炮塔；个人盾和上下不同层范围盾；delta>1 与在途/延迟引爆/传播中保存重载；坐标带边缘不发生伪邻近爆炸；跨格建筑与移动 Pawn 在一个爆炸事件中只受预期次数伤害。上述场景在 2026-10-16 调研时未执行；后续原型的具体通过项见末节，未覆盖项继续未验证。

当前证据入口（同一 MVID）：CastPositionFinder.TryFindCastPosition 0600467A、EvaluateCell 0600467B、CastPositionPreference 0600467C；JobGiver_AIFightEnemies.TryFindShootingPosition 060052E3、JobGiver_AIFightEnemy.FindAttackTarget 06005318 / ShouldLoseTarget 06005317；Verb.TryFindShootLineFromTo（按全名 resolve）；Projectile.TickInterval 06002A29、CheckForFreeInterceptBetween 06002A2A、ExposeData 06002A25；Bullet.Impact 0600B334；CompProjectileInterceptor.CheckIntercept 0600A215、CompShield.PostPreApplyDamage 0600A369；Verb_MeleeAttack.TryCastShot 0600CF29；DamageWorker.ExplosionCellsToHit 06000A02 / ExplosionAffectCell 060009FE；Explosion.StartExplosion 0600285D、ExposeData 06002863；GenSight.LineOfSight 0600387C、GenGrid.CanBeSeenOverFast 0600386C。参考源码入口为 ProjectZ Combat/Heights.cs、Combat/CombatPatches.cs、Patches/VirtualPatches.cs，以及 IsoRim Source/Projectiles.cs。

本篇将战斗的主要接入链和已发现缺口落实到成员级；没有把设计当成运行验证。严格屋顶施工、结构生命周期及天空/出口入口已补到 [multilayer-structure.md](multilayer-structure.md)；垂直电力/设施和环境传播已补到 [公共系统审计](multilayer-services.md)，原始需求收口见 [readiness](multilayer-readiness.md)。


## 1.7.4981 增量与 RimLayers 原型（2026-10-18）

本节基准为根目录 Assembly-CSharp.dll，MVID `f1e6382600a84191bbc6d8a184fd6b57`。上文 1.7.4980 的调查日期和范围保留，不能把新增原型结果扩展为全部组合兼容。

新核实的接入细节：

- `CoverUtility.CalculateOverallBlockChance` 的第一参数是 `LocalTargetInfo`，不是 `IntVec3`；转译器反射匹配须保留目标类型。原版 AIFightEnemy 更新目标时，即使旧目标尚可保留，也会尝试重新选目标；未找到且 `chaseTarget=false` 时会丢弃旧目标，不能为跨层而抹掉守卫/追击策略。
- `Projectile.ArcHeightFactor`（17000878）由 Def 系数和水平距离上限决定；`DrawAt`（06002A2D）把它作为平面北向绘制偏移。若已有三维轨迹决定当前物理楼层，不能再无条件重复这层二维偏移。原版手雷 Def 链见 `ThingDefs_Misc/Weapons/RangedIndustrialGrenades.xml`。
- `Projectile_Explosive.Explode`（06002A38）先记录 Map 再 Destroy，随后调用 GenExplosion；物理命中点应在入口捕获。延迟引爆继续使用原版 landed/ticksToDetonation，不重新发射。
- `Explosion.ExposeData`（06002863）已保存 cellsToAffect、damagedThings、startTick 等，**没有保存 postExplosionGasAmount 和 postExplosionGasRadiusOverride**。需要自有保存的这两个参数必须在实体引用恢复后写回，否则气体波在读档前后会改变语义。
- `DamageWorker_Vaporize.ExplosionAffectCell / ExplosionDamageThing`（06001D80 / 06001D81）各有 2.9 距离截断；基类的融雪/颜色改成逻辑距离后，还需处理这两个派生分支。当前程序集没有 ExplosionCellsToHit 原始虚重载的派生覆盖；Vaporize.ExplosionStart 调用基类后再触发热浪效果。

RimLayers 采用同一个原版 Explosion，额外保存物理爆心和上述未保存参数；不为每层另造完整爆炸。覆盖在引爆时冻结，沿用原版队列与受伤实体集合；同一波破坏遮挡不追溯扩展覆盖。默认候选使用三维球形距离与楼板/墙体几何；显式 overrideCells 保留调用者形状，只做逻辑边界过滤与去重。排除半径、起火衰减、气体范围、伤害/穿甲与传播时间同源。

实际证据统一见 [RimLayers 实施记录](../Mods/RimLayers/Docs/WORK-LOG.md)：真实开火/飞行保存、敌人射位、盾与 EMP、多格目标、动态楼板/目标换层、传播中与延迟引爆中重启，以及 Vaporize/气体释放。爆炸热量的分配、危险通知与环境接入已有后续实现和实测，详见实施记录与 [公共系统专题](multilayer-services.md)；具体友伤、特殊体型、炮塔和第三方组合仍未全部验收。


### 原生护盾返回边界、空域屋顶与手雷警戒补证

当前同一程序集的 CompProjectileInterceptor.CheckIntercept（0600A215）先处理 Active、弹丸类型、阵营、入/出射和圆周相交，只有成功分支才扣原生 currentHitPoints 并返回 true。友方发射者可以在早期返回 false。RimLayers 先固定接触点，再用匹配当前弹丸/拦截器的调用作用域，在原生返回后的 Postfix 保存最终布尔值；失败不会遗留成功收据。该收据覆盖已扣耐久、尚未调用 Projectile.Impact 的返回边界，不扩展为任意第三方回调顺序的兼容保证。

原生 Projectile.TickInterval（06002A29）先扣 ticksToImpact，再扫描本次完整线段。大 delta 可能让原生时钟已经到达终点，而自有 Sweep 只处理到中途护盾；保存的 LastProgress 必须是已消费的接触进度，不能在调用外部入口前预写整段终点。放行后读档继续后段墙体检测；成功则只提交一次原生 Impact，不重新调用护盾扣耐久。

真实绘制还经过 DynamicDrawManager.ComputeCulledThings（0600119D）：非 Pawn 的 cell 取 Thing.Position，coarseBounds 取 GenAdj.OccupiedDrawRect（0600048A），而后者也直接使用 Position。因此只改 Projectile.DrawPos 不足以显示保存时尚未完成的扫掠接触点；原型只在此裁剪入口为受管弹丸换成对应 DrawPos 的格与平移后的占地框，不改变 Position 或其他物体的裁剪。

LayerShotScene 的 Roof 查询现在直接检查真实 RoofGrid，不依赖同层 HasSurface。人工屋顶能跨在无楼面空气上；这种屋顶仍须挡射线、弹丸、爆炸与危险警告。新增 combat-roof 验证自下而上的实际弹丸在该屋顶下表面终止，目标不受伤。

Core 的 Proj_GrenadeFrag 链（ThingDefs_Misc/Weapons/RangedIndustrialGrenades.xml）半径 1.9、延迟 100 tick。Impact（06002A37）设置 landed/countdown 后发原生危险通知，派生 TickInterval（06002A36）在同次回调末尾再减去 delta；因此一 tick 落地回调后的剩余值是 99，不是重置成 100。combat-grenade-flee 在这个已落地状态保存，生成与独立读档都保持原始 99 tick 期限；上层 Pawn 真实冲刺离开，下方实际爆炸覆盖其旧格但未伤到已撤离者。Core 与全部官方 DLC 的具体结果及命令见 [实施记录](../Mods/RimLayers/Docs/WORK-LOG.md)。


### 原生友伤与自动炮塔补证（2026-10-18，同 MVID）

Projectile.CanHit（06002A2F）按 IntendedTarget / NonTargetPawns / NonTargetWorld 决定候选资格；CheckForFreeIntercept（06002A2B）对沿途 Pawn 使用体型、姿态、距离和 friendlyFireChanceFactor，并仅为非敌对阵营应用 preventFriendlyFire。ImpactSomething（06002A30）对明确目标保留另一条原生命中分支及倒地远距离修正。因此 preventFriendlyFire 保护沿途误伤，不取消明确指定的友方目标。

RimLayers 的 combat-policy 现在以真实弹丸/原生 TakeDamage 观测上述开关、同局部位置但低于弹道的 Pawn、松鼠/牦牛/机体与倒地目标；Core 与全 DLC 各 28 项通过。瞄准事实必须在发射时记录，不能在原生伤害改变姿态或致死后重算原发射高度。

Building_TurretGun.TryFindNewTarget（0600AA44）对非高空弹使用 BestShootTargetFromCurrentPosition；Tick（0600AA41）、TryStartShootSomething（0600AA43）与 BeginBurst（0600AA47）驱动实际蓄力和原生 Verb。真实 MiniTurret、Battery、导线和炮管在 Core/全 DLC 中都完成了无步行路线的跨层自动选敌、飞行与伤害，各 9 项通过。该证据只覆盖具体测试的原版迷你炮塔，不扩展为所有第三方炮塔或战斗框架兼容。具体输入、失败夹具修复和目录见 [WORK-LOG](../Mods/RimLayers/Docs/WORK-LOG.md)。


## 1.7.4982 三维掩体与有人迫击炮接入（2026-10-20）

基准根目录 Assembly-CSharp.dll，MVID `a46e6f45b6164261bb63a9471bf7402a`。本节是后续实现，前文未定规则属于早期调研；当前 RimLayers 规则与验收见 [三维战斗](../Mods/RimLayers/Docs/COMBAT-3D.md)。

- `Verb_LaunchProjectile.TryCastShot`（06003AB6）保留本次 Projectile Def 后调用 `CompChangeableProjectile.Notify_ProjectileLaunched`，再处理强制散布、ShotReport 精度及二维 PassCoverChance。三维暴露若另抽一次，就必须在该次发射作用域跳过原二维掩体分支；不能全局改原版报告 getter 或重新实现整套消耗。
- `Building_TurretGun.OrderAttack`（0600AA40）自己检查原始距离和 EffectiveRange；只补 Verb 不足以支持跨层强制目标。`TryFindNewTarget`（0600AA44）的普通分支调用 `BestShootTargetFromCurrentPosition`，后者先用 EffectiveRange 限候选半径，需覆盖新增射程。敌方迫击炮另有随机建筑候选闭包，使用自己的 DistanceToSquared。
- `Building_TurretGun.TargSearcher`（0600AA45）原版有人炮台返回操作员；三维射击物理原点必须仍取炮台 caster。`Tick`（0600AA41）、`TryStartShootSomething`（0600AA43）继续负责供能、操作资格、蓄力与冷却。
- `JobDriver_ManTurret`（02000A55）先取炮管/炮弹，使用 `GenClosest.ClosestThingReachable`（搜索半径 40）、原生预订与搬运；`LoadShell` 后到 `PathEndMode.InteractionCell`，只有到位后的 man toil 调用 `CompMannable.ManForATick`。跨层供弹应扩展真实候选/路径，不直接伪造操作员到岗。
- `Projectile.TickInterval`（06002A29）对 overhead 也执行全段拦截；三维曲线扫掠无需另造炮弹生命周期。`ProjectileProperties.arcHeightFactor` 默认 0；原版 grenade 模板为 2。仅判断 flyOverhead 会漏掉投掷曲线。
- `CastPositionFinder.CastPositionPreference`（0600467C）防护查询传候选格与敌方 Position，而不是候选处真实 Pawn；九点防护评分需要携带候选 Pawn 的体型/姿态，不能把无人候选格默认成低矮点目标。

原版沙袋、路障 fillPercent 均为 0.55，不能作为独立物理形状来源。RimLayers 用 DefModExtension 配置逻辑高度，同层概率不变。实现与新运行证据在目标 mod，不修改原版数据或 DLL。


弹药消耗边界补证：`CompChangeableProjectile.Projectile`（17001A94）在 `Loaded=false` 时返回 null；`Verb_LaunchProjectile.Projectile`（17000B72）此时回退 `defaultProjectile`，而原生迫击炮没有配置该默认弹。`VerbUtility.ProjectileFliesOverhead`（06003A95）依赖当前弹药，因此不能单独用它识别空炮或扣弹后的武器类别。RimLayers 在原版扣弹前固定本次 Def，并以 `VerbProperties.isMortar`（040007AF）识别原生迫击炮；空炮预览和实际消耗后的几何不能临时变回直射/高度加成。


## 袭击投掷兵：射位可达不等于自动开火可用（2026-10-20）

1.7.4982 的 `JobGiver_AIFightEnemy.TryGiveJob`（06005311）用 `Verb.CanHitTarget` 判断可否返回 Wait_Combat，也可能通过 CastPositionFinder 选中当前格；`JobDriver_Wait.CheckForAutoAttack`（06004156）随后使用 NeedLOSToAll / NeedThreat / NeedAutoTargetable 重新索敌。跨层曲线可能越过楼板，但人物看不见目标实体。因此自动射位必须同时满足实际可见性和曲线可达，不能把所有“抛物线可行”的格子都当成自动开火位置。玩家明确指定的区域投掷保持独立规则。

RimLayers 原战场保存复现 5 名投掷兵长期 Wait_Combat：canHit=true、canSee=false、auto=null。修复当前格及候选格后，同一保存中的 5 人离开无效射位，并有原生投掷发射。具体证据见 [袭击补验](../Mods/RimLayers/Docs/RAID-COMBAT-VALIDATION.md)。

`IncidentWorker_Raid.TryExecuteWorker`（0600714E）先生成威胁，再通过 raidStrategy.Worker.MakeLords 建立原生队伍；仅创建几名敌人并给 AttackStatic 不能替代真实袭击验证。`PawnGroupMakerUtility.TryGetRandomPawnGroupMaker`（06008287）使用 parms.seed 临时随机状态挑选合法编组；种子变化可能生成纯近战，测试枪械分支应确认实际兵种和装备，而不是从袭击点数推断。

## 原版跳跃载体：不能只放开跨层 CanHit（2026-10-21）

在 1.7.4984 rev1242 / Assembly-CSharp MVID 484efdc5765b463ca9cc54a6131e38f0 定向核实：

- Verb_Jump → JumpUtility.OrderJump → CastJump → DoJump → PawnFlyer。OrderJump 不要求步行可达，但 CanHitTargetFrom 仍用存储距离和二维 GenSight。不能将其解释成无条件穿墙。
- Verse.Verb.TryStartCastOn 在有预热时另查 TryFindShootLineFromTo。多层项目若在此统一拒绝非 Verb_LaunchProjectile，跳跃即使选点通过仍无法起跳。
- PawnFlyer.RecomputePosition 直接插值 startVec / DestinationPos 并写 Position；TickInterval 检查终点却不扫掠楼板。必须让逻辑轨迹负责模拟位置，不能只改 DrawPos 或把存储隔离带算入飞行距离。
- CheckDestination 会在 3.9 范围选替代落点，只验证目标格，不验证改道是否穿板；跨层接管时需重新检查完整剩余路径。
- RespawnPawn 先 RestoreCapturedJobs，再 CheckForJobOverride；真实夹具确认先恢复的 Goto 可被 Wait_Combat 覆盖。应在保留原生着陆流程的基础上协调恢复顺序，保持一次交付。
- 原生 TryStartCarry(Thing) 的 bool 重载面向已脱离地图的对象；诊断从地图拾取整堆应使用数量重载并确认返回数量/原 Thing 引用。不要把夹具未拾取当作飞行丢货。

RimLayers 的实现与游戏证据见 Mods/RimLayers/Docs/JUMP-PACK.md、JUMP-VALIDATION.md。其楼板/屋顶阻挡复用现有 IShotScene 的真实结构事实，增加二次曲线人物体积扫掠；不采用楼梯/气体连接作为飞行洞口。
