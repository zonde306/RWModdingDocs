# 多层公共系统：电力、设施、空气、天气与光照

核对：2026-10-16，RimWorld 1.7.4980 rev60，主程序集 MVID 8e962ecc7ac742d28f3b1b16cd24d2cd，SHA-256 EFF7D97949F919FB2B151EBFB906E9BE749B63410B7DBDA78EEEBB7972B245C3。

范围：补齐工作台可运行和楼层隔离所需的原版公共系统入口，以及 ProjectZ / IsoRim 的相关源码。下文是源码事实、条件推导和实施建议；未修改参考 mod、构建原型或运行游戏。空间基础见 [queries](multilayer-queries.md)，屋顶/天空关系见 [structure](multilayer-structure.md)。

## 一个布局，多种连接图

不能将楼梯的 Movement 可达边同时当成所有系统的相邻边。建议共用楼层/格转换与结构版本，分别维护：

| 系统 | 合法连接的例子 | 默认不允许的推论 |
|---|---|---|
| 移动 | 楼梯、电梯、单向跳落，受 Pawn 能力/危险/门限制 | 电力也能沿同一边通过 |
| 空气/气体 | 开放楼梯井、洞口、启用的通风连接 | 实心楼板因为能承重就透气 |
| 热 | 楼板导热、洞口对流、专用热管，系数分别定义 | 多层 Room 永久合并 |
| 电力 | 明确配对的竖向电缆端子 | 同 Map 或同局部 x/z 就自动通电 |
| 射线/弹丸 | 无实体遮挡的三维线段 | Pawn 可走的电梯必定能透射 |
| 声音/惊扰 | 门/开口/楼梯与衰减规则 | 听觉范围等同走路距离或视线 |
| 设施增益 | 本层合法距离/朝向/LOS；显式 opt-in 的远程设施 | 工作台能跨层取料就应共享工具柜 |

这些图可共享端点索引和失效通知，权限与边权必须分开。间隔行只是存储布局，不是逻辑空气、地面、世界出口或无代价电缆。

## 电网：跨层端子加入图，而非伪造发电机

当前原版已核实：

- PowerNetMaker.ContiguousPowerBuildings 从 Building root 做集合遍历，只查看 GenAdj.CellsAdjacentCardinal，收集 TransmitsPowerNow 的建筑；NewPowerNetStartingFrom 用结果构建 PowerNet。
- PowerNet 构造将每个 transmitter.transNet 指向该网络，并注册其组件/连接子设备；PowerNet.Map 来自 powerNetManager.map。
- CompPower.ConnectToTransmitter 设置用电连接的父子关系并 RegisterConnector；它不是把两个独立 transmitter 网络合并的通用 API。
- PowerConnectionMaker.BestTransmitterForConnector 在存储格周围 ExpandedBy(6) 找电线；PotentialConnectorsForTransmitter 也用占地扩张 6 格。正常不同层隔数百格不会连接；坐标带边缘若距离足够近，也没有楼层过滤。
- PowerNetManager.UpdatePowerNetsAndConnections_First 先处理延期注销/注册、销毁受影响网络，再创建网络与接入用电器；它的受影响邻域同样是平面四邻接。
- Notfiy_TransmitterTransmitsPowerNowChanged（原版拼写如此）为仍 Spawned 的端子排入注销+注册，并刷新电线显示。

建议单 Map 分支提供“竖向电缆/端子”：每个真实端子仍是本层 transmitter；配对、启用且结构有效的端子在电网遍历中成为额外邻居。本层物理邻居也需拒绝越入其他带或间隔行。一次 BFS 必须覆盖远端端子后面的本层电线，而不是只在结果末尾添加一个远端设备。

连接建立、断开、开关变化、损坏、坍塌和读档时，收集两端原有网络及受影响设备，经正确的注销→重建→重连顺序处理。原版四邻接失效邻域不知道远端端子，必须补齐；只改 PowerNetMaker 枚举而不改失效范围，会留下两个旧网络或断开后仍供电。

本层普通用电器寻找电线时应带 SameFloor 条件；特殊跨层连接由端子图授权，不能全局把 6 格范围改成平面投影，使任何上层设备无条件连上下层电线。电线预览/选中连线也用逻辑转换显示。

PowerNetTick 根据发电、用电、存储能量及断电状态更新设备和电池。首选扩展连接图以复用原版预算；不要在每层生成虚假电源来“桥接”，也不要跨网络直接复制电量。若选择独立网络之间的能量桥，必须另外设计守恒、功率上限、损耗、执行阶段和断电规则。

多 Map 分支不能直接把不同 Map 的 CompPower 塞进同一个 PowerNet：其 manager / grid / Map 生命周期仍是单图。应采用自有跨图能源总线或受控能量桥；这与单 Map 的额外图邻居方案不同。

## 电力保存与重建

CompPower.PostExposeData 保存 connectParent 的 parentThing 引用，PostLoadInit 后重新 ConnectToTransmitter。它不保存自有竖向端子的拓扑语义。

本 mod 保存端子实体引用、稳定楼层/逻辑位置、配对 ID、开关与 dataVersion；不保存 PowerNet、transNet 或临时组件枚举。加载后先恢复布局/结构和端子，再让网络按现状重建，最后接用电设备。连接缓存不能先于交叉引用或有效楼面恢复。

旧组件/端子死亡、失去 Map 或版本不符时标记断连并重新建图；不能用保存的网络编号强行接入新网络。电池实际存量仍由原版电池组件保存，网络重建不额外创建或回放电量。

## 设施：默认同层，保留原版容量与朝向

CompAffectedByFacilities.CanPotentiallyLinkTo_Static 检查相邻、床头/面向、直线建筑、min/maxDistance 等；IsPotentiallyValidFacilityForMe_Static 在 requiresLOS 时检查二维 GenSight，未发现可通用代表楼层的字段。CompProperties_Facility 也没有“同楼层”或“跨层”属性。

CanPotentiallyLinkTo 与 CanLinkTo 进一步保留 maxSimultaneous、优先替换、设施类型、设施启用状态和重复连接检查。不能以一个自有距离函数替代整套验证。

原版 XML 入口：Buildings_Misc.xml 的 ToolCabinet 提供 WorkTableWorkSpeedFactor 和最大同时数量；Buildings_Furniture.xml 的 Dresser 明确 maxDistance=6。这些都是同一存储地图里的邻近设施，不应被层转换无条件扩大为远程增益。

建议在预览和实际连接的共同入口先拒绝不同逻辑层，再执行原版规则；目标移动/蓝图替换/结构坍塌时重新验证旧连接与相关 Stat 缓存。需要跨层生效的自定义设施用明确扩展声明连接方式、范围和遮挡。

在原型支持表中区分“跨层使用家具”（Pawn 走到其所在层使用）与“家具隔层提供增益”（设施网络规则）。前者不要求后者。

## 温度：房间独立，跨层热交换单独结算

当前 MapTemperature.MapTemperatureTick 在 TicksGame%120==7（或 fastEcology）时遍历本 Map 所有 Room 调 EqualizeTemperature。普通室内房间分别计算薄顶、无顶、墙与深层交换；ThinRoofEqualizationTempChangePerInterval 使用室外温差、ThinRoofCoverage 与 5E-05×120。

ProjectZ FloorHeat 在 RoomTempTracker.EqualizeTemperature 后补充相邻层共有楼板的热交换，并抵消 PZ_FloorRoof 原本向室外的薄顶损耗。它缓存上下共有格，在屋顶、楼面和房间变化时失效；这是可读的原型参考，不能直接把所有跨层 Room 合并。

建议区分楼板导热与开口对流。若需要严格成对守恒，在固定阶段读取房间温度快照，每条共享面/开口只算一次，再按双方有效体积施加等量相反热量。Room 引用和格索引读档重建，室外边界可作为无限热库。

不要一边新增楼板传热，一边保留同一楼板全部原版“薄顶向室外”交换，从而重复散热。也不要在每个房间回调中修改相邻房间再读取新温度，声称结果完全不受房间迭代顺序影响。是否追求严格热守恒是实现策略；样本没有证明这一性质。

## 气体：不可走的间隔行仍可能导气

GasGrid.GasCanMoveTo 只检查 InBounds 与 Edifice：完整填充建筑阻挡，门按 Open 状态判断；没有检查地形 Walkable。因此 PZ_OpenAir 的 Impassable 不能充当气体隔离墙。

TryDiffuseGases 只枚举存储四邻格，对 ToxGas、RotStink、DeadlifeDust 扩散；BlindSmoke 在此保留原浓度，不能假设所有 GasType 都有相同扩散语义。EqualizeGasThroughBuilding 也只看建筑前后或周围的平面格。

当间隔行没有挡气建筑时，气体可进入这些“非逻辑空间”并尝试继续扩散；最终是否到达另一带还受浓度、耗散和布局影响。此为源码条件推导，未游戏内复现。

建议：

1. 原生平面扩散拒绝任何间隔行和不属于当前逻辑层的边；不要用“是否有楼板”代替“是否有空气体积”——洞口上方也有空气。
2. 通过 AirLinks 显式连接上下层开口/通风口，实心结构面不透气；门和设备开闭分别更新该图。
3. 按 GasType 支持策略计算转移量，源扣多少、目标加多少，限制 0..255，正确处理舍入余量和多个连接争抢同一格，不能各连接独立使用旧浓度无限抽取。
4. 原版 SetDirect 保存通道/DLC 屏蔽；调用后按原生流程刷新气体绘制。气体的 outdoor 耗散/真空规则需与真实天空/空间状态协调。
5. 只保存原版格子浓度和本 mod 必要连接/余量状态；GasGrid.ExposeData 已保存 gasDensity 与循环索引，不另 Deep 复制整个气体网格。

当前 GasGrid.Tick 的耗散/扩散采样量分别约 Map.Area/64 与 Map.Area/32；大 Map 楼层预分配会增加迭代量。跳过间隔行优化时要保持真实格子的更新频率，不能换了遍历集合却沿用错误配额。

## 火焰、热源与惊扰

Fire.TrySpread 在本存储格附近选目标，近处直接 TryStartFireIn，远处检查二维 LOS 后生成 Spark。不存在原生的“从一层火焰沿楼梯蔓延到二层”的完整规则。

应先约束普通横向点火/火花不越逻辑层，再按开口、材料可燃性和明确概率/速率增加垂直传播；不能简单复制 Fire 实体。楼板/装修燃烧、楼板失效与更高层支撑联动通过结构事务处理，避免同一次事件重复破坏。ProjectZ XML 将结构楼板（包括木材版本）设为不燃，这是样本规则，不是用户已指定的玩法。

Fire.VulnerableToRain 读取本格 RoofDef，并保留厚顶和 holdsRoof 建筑的特殊判断。雨能否触及该格还需整列天空遮挡；不应只为避雨在所有洞口写入假屋顶，改变原版室内和射线语义。

GenClamor.DoClamor（ClamorEffect 重载）按开放门做 Region BFS，预算为 15，并以原始 InHorDistOf 和听力过滤 Pawn。扩展 Region 邻居后，存储距离仍可能排除真实近处的上下层 Pawn；不能把它当成已完成竖向声音。需要声音连接/衰减时单独适配，保留听力、门和源对象。

## 天气：真天空暴露不等于本格 Roofed

SteadyEnvironmentEffects.SteadyEnvironmentEffectsTick 以完整 Map.Area 和 cellsInRandomOrder 分批处理 DoCellSteadyEffects。它不是只通过 CellFinder.RandomCell 选格，所以将 RandomCell 限制到地面不会自动使所有环境效果只在地面运行。

DoCellSteadyEffects 用本格屋顶、Room、Terrain 决定降雪、雨洗污物、腐坏、室内融雪/高温点火等，最后仍调用地图条件和 GasUtility。SnowGrid.CanHaveSnow 只检查建筑能否积雪、terrain.holdSnowOrSand，不是完整的上方多层遮挡检查。

需要 SkyExposure 查询：同列所有更高实体面都参与，而不是只判断该层 RoofGrid。真实户外露台可淋雨/积雪；高层楼板下即使紧邻层是空洞，也不能当作无遮挡天空。天气采样的逻辑位置还影响噪声花纹，不能让存储带偏移变成楼层不同的“地理位置”。

适配局部天空遮挡时保留原版本格屋顶和 Room 的职责，不全局改 Roofed 让所有系统看见同一种替代结果。DoCellSteadyEffects 的游戏条件、气体与室内热源等副作用也应保留，不能整段跳过只重写下雪。

## 光照：游戏网格与绘制分别验证

当前 1.7 不应按旧版本类名猜 GlowFlooder。实际链条：

- GlowGrid.GlowGridUpdate_First 调 Verse.Glow.ComputeGlowGridsJob，按灯并行计算；随后 CombineColorsJob 合并格子颜色，两次 Schedule 后都 Complete。
- ComputeGlowGridsJob.Flood 只知道 mapSize、indices、lightBlockers 和存储格偏移，不知道楼层布局。
- GlowGrid.GroundGlowAt 对未 Roofed 的本格加入 SkyGlow，再与累计灯光比较。这会影响游戏判定，而不只是画面。
- LightBlockerAdded / Removed 更新 Native 位图并 DirtyCell / DirtyLightsAround；RegisterCustomGlower 可注册显式位置的灯，半径限到 40。
- IsoRim Glow.cs 绑定自己的纹理、天空参数和 Floors.GlowFloors，属于绘制适配，不证明原版游戏光照数值已经按楼层隔离。

默认先保证人工光照不横穿间隔行，可在进入原生计算前维护派生的间隔行 blocker / 每层裁剪规则，注意原版 blocker 增删不能擦掉本 mod 的边界掩码。不能在原生并行作业使用 NativeArray / 位图时任意修改、扩容或 Dispose；版本化输入在确定阶段提交。

穿洞口的竖向透光若需要，再由独立逻辑光线/透射服务生成受控的层内贡献；避免灯的镜像副本再递归生成镜像、光能被重复叠加。描绘效果与游戏数值共用天空暴露和结构几何，但验证点分开：画面、GroundGlow、植物生长、工作速度/黑暗心情。

## 生命周期、性能与支持边界

布局/拓扑、结构、门状态、通风开关、端子、设施和光源变化应失效各自的派生图；不要一次改顶就遍历所有 Pawn、所有灯和全图气体。读档按“布局/结构 → 原版实体 → 自有连接 → 原版/自有派生网格 → 任务”顺序协调，Region、PowerNet 和临时原生缓冲不作为稳定存档身份。

单 Map 方案的优势是原版实体和大部分管理器统一；代价不是只有绘制。电网、气体、环境采样和光照都依赖扩大后的真实网格。性能可从方法体定位，但内存峰值、tick、路径和绘制的实际上限必须用原型测，不能由 5.1 倍格数直接推出耗时倍数。

多 Map 分支的原生层内系统更自然，但竖向电力/空气/热/声音仍需要自有连接服务。DLC 真空、特殊气体、能力、重力飞船及第三方管网应使用明确支持列表；没有对任意外部 mod 的内部模型做无限扩展调查。

## 证据与待执行验证

主要 metadata 入口（当前 MVID）：PowerNetMaker.ContiguousPowerBuildings 06007B4D / NewPowerNetStartingFrom 06007B4E；PowerConnectionMaker.BestTransmitterForConnector 06007B31 / PotentialConnectorsForTransmitter 06007B30；CompPower.ConnectToTransmitter 06007AB0 / PostExposeData 06007AA7；PowerNetManager.UpdatePowerNetsAndConnections_First 06007B5C / Notfiy_TransmitterTransmitsPowerNowChanged 06007B55；PowerNet 构造 06007B34 / PowerNetTick 06007B3D；CompAffectedByFacilities.CanPotentiallyLinkTo_Static 0600995B / IsPotentiallyValidFacilityForMe_Static 0600995F / CanLinkTo 06009958；MapTemperature.MapTemperatureTick 06001855、RoomTempTracker.EqualizeTemperature 060017D8；GasGrid.GasCanMoveTo 060012D7 / TryDiffuseGases 060012D5 / ExposeData 060012DC；Fire.TrySpread 0600B166 / VulnerableToRain 0600B164；GenClamor.DoClamor 060037C9；SteadyEnvironmentEffects.DoCellSteadyEffects 0600CEBD / SteadyEnvironmentEffectsTick 0600CEBC；GlowGrid.GlowGridUpdate_First 06001255 / GroundGlowAt 0600124B；ComputeGlowGridsJob.Execute 06003F80 / Flood 06003F81。其他成员可按正文全名重新 resolve。相关原版 XML 与样本源码入口见各节。

必要验证：上下层供电、两条竖向电缆形成环、断开/拆毁/读档后不串网或复制电量；同层设施有效、隔层设施拒绝且数量限制正确；实心楼板隔气、开放楼梯井传气、门关闭后中止；上下房间冷热交换与室外散热不重复；火花与雨水不跨坐标带误作用；天空遮挡和露台积雪；游戏光照/画面一致；两层→五层压力测试。它们是实现阶段验收，2026-10-16 调研时均未执行；2026-10-18 的原型结果见下节。

至此公共系统的必要入口、边界与可行设计已补齐。已核实的成员足以支持实施前方案，不等于任何运行兼容或性能保证；研究收口与原始需求逐项对照见 [readiness](multilayer-readiness.md)。


## 1.7.4981 原型增量（2026-10-18）

基准：Assembly-CSharp MVID `f1e6382600a84191bbc6d8a184fd6b57`。已对本节接入重新核实，不把 1.7.4980 的其余方法快照视为全部回归。

- Map.FinalizeInit（0600145B）在 MapComponentUtility.FinalizeInit **之前**执行 powerNetManager.UpdatePowerNetsAndConnections_First。竖向端子索引必须在原版初始建网前可用；仅在地图组件 FinalizeInit 建索引过晚。索引重建不等于注销业务设备。
- CompPowerTrader.ResetPowerVars（06007B05）会清空功率/通电状态并调用 Flickable.ResetToOn；结构失效事务不能因此擅自打开玩家关闭的发电机。原版待处理网络动作按“全体先注销/注册准备，再全体建网，再接用电器”分阶段执行，远端也应加入同一阶段而非尾部手动拼接 PowerNet。
- GasGrid 两种 indexed SetDirect（060012CF / 060012D1）均保留 Biotech / Anomaly 通道掩码；packed 重载还设置 anyGasEverAdded。自有交换使用原版 packed 写入并刷新 MapMeshFlagDefOf.Gas。原版 GasCanMoveTo 只检查边界/完整 Edifice 与门；隔离区过滤必须另加。
- 原版气体扩散不处理 BlindSmoke；楼梯通行权限与空气权限应分别检查，关闭落脚点门仍可阻气，Pawn 禁止标记不等于气密门。

RimLayers 现有垂直端子、设施限制与竖向气体交换的实现、Core/全 DLC 组合和独立保存/读档证据统一见 [WORK-LOG](../Mods/RimLayers/Docs/WORK-LOG.md)。热/天空与火焰/惊扰增量见下节；特殊 Pawn 和规模仍需继续完成。原型的浓度交换守恒指自有竖向步骤，原版源汇/耗散继续保留。

### 热与天空的当前接入（同日、同 MVID）

- RegionTypeUtility.GetExpectedRegionType（06001788）将不可走且无完整建筑的格返回 ImpassableFreeAirExchange；存储隔离格要显式 None，不能只依赖 Terrain 的 Impassable。真正上层空域仍须保留空气体积。
- Room.PushHeat（060017B3）以 energy / CellCount 修改温度；室外房间拒绝注热。Room.UsesOutdoorTemperature（17000470）依赖 TouchesMapEdge 与至少 25% OpenRoofCount。逐层温度独立，不意味着所有室外 Room 身份必须不同。
- RoomTempTracker 的薄顶/无顶损耗（060017DB / 060017DC）分别为室外调整温差 × 覆盖率 × 0.006 / 0.084；竖向成对换热后应排除低层的重复室外天花板项。RegenerateEqualizeCells（060017D7）还会跨两格采样，需要拒绝存储隔离区。
- Room.Notify_RoomShapeChanged（060017B8）触发温度数据重建；Room.Notify_RoofChanged（060017B7）只使本房间缓存失效。远层遮挡改变后的派生天空缓存由 mod 自己失效。
- DoCellSteadyEffects（0600CEBD）分别做雨雪/腐坏、室内热、地形事件、游戏条件与气体；仅替换其 Roofed 查询，不整段重写。AddFallenSnowAt / AddFallenSandAt（0600CEC0 / 0600CEC1）的噪声使用 ModuleBase.GetValue(IntVec3)，可仅把噪声输入改为局部坐标，积雪目标仍用原格。
- GasGrid.TryDissipateGases（060012D4）以 Roofed 决定半速/全速耗散；Fire.VulnerableToRain（0600B164）另保留厚顶/holdsRoof 的特例，天空适配只加更高遮挡门槛。
- GlowGrid.GroundGlowAt（0600124B）可通过 ignoreSky 排除实际被上方遮挡的日光，同时保留原版累计灯光。SectionLayer_LightingOverlay.GenerateLightingOverlay（06001550）直接查 RoofAt(int) 和 Roofed(int)，必须在绘制入口单独适配；只改游戏 GroundGlow 不改变画面。

原型系数/热库策略、源代码与实际验收见 RimLayers PLAN / WORK-LOG；自有热步骤守恒不包含原版室外、墙体和热源的净源汇。人工灯光当前限层内，穿洞口竖向灯光为后续适配。

补充保存与热源：TemperatureVacuumSaveLoad.TempFloatToShort / TempShortToFloat（06001870 / 06001871）使用 clamp(-273.15..1000) → 乘 16 → int 向零截断 → ushort；加载恢复到 1/16°C 网格。DoExposeWork / ApplyLoadedDataToRegions（0600186E / 0600186F）写原版逐格缓存，再在重建 Region 后恢复房间温度。验证非整数温度不能要求读档后完全保持原始浮点数，也不应用测试容差掩盖超过原生量化的漂移。

DamageWorker.ExplosionStart（060009FC）默认把 explosionHeatEnergyPerCell × cellsToAffect.Count 注入中心；唯一当前派生 Vaporize.ExplosionStart（06001D82）调用基类一次。原型在同一个开始阶段按实际覆盖格分配该预算，GenTemperature.PushHeat（06001845）继续负责实际房间与相邻空气的注热/室外拒绝规则；不在传播 tick 重复注热。

### 火焰、惊扰与爆炸警戒（同日、同 MVID）

- Fire.TickInterval（0600B160）仅在 fireSize>1 时累计扩散时间，达到 SpreadInterval 才调用 TrySpread（0600B166）；间隔为 max(75,150-(fireSize-1)×40)。竖向传播应附着在该尝试而不是把每次 interval 回调当成一次扩散。FireUtility.ChanceToStartFireIn / TryStartFireIn（0600B16A / 0600B16B）保留可燃物、建筑内部与阻燃污物规则，但自身不认识逻辑隔离区。
- Spark.Impact（0600B34D）先调用 base.Impact，再在 Position 通过 FireUtility 点火；不使用 hitThing 来决定点火格。若弹丸接触点被放到墙外空气侧，须为此副作用保留实际被击占地格，不能简单用多格建筑中心锚点替代。原版点火器仍拥有 Fire 实例、燃料规则与 instigator。
- GenClamor.DoClamor 的委托重载（060037C9）使用开门 Region BFS、15 区预算、存储距离及 Hearing。追加跨层声路应保留同层原回调，按实际听力预算逐 Pawn 去重；不能把声音可传播等同于有可步行路线，也不能声称由此改好了音频混音。
- GenExplosion.NotifyNearbyPawnsOfDangerousExplosive（06003819）区分阻挡建筑的外围房间与普通来源，保留智力、阵营、伤害类型和 instigator；跨层距离/遮挡另接。Pawn_MindState.Notify（0600452F）会立刻 CheckForJobOverride（060044F9）。JobGiver_FleePotentialExplosion.TryGiveJob（06005299）虽然检查 PlayerForcedJobNowOrSoon（06008329），整棵树仍可能选择 Wait_Wander 抢占强制 Wait。真实夹具已复现；可在受管地图的通知入口只记住危险、推迟强制任务期间的整树重选，不绕过征召或另造任务恢复队列。
- 原版躲避节点使用 9 格范围和 RCellFinder.TryFindDirectFleeDestination（0600C92F）的存储坐标随机目标。跨层预警之外，必须把实际逃离目标也转为合法楼层/楼面并通过真实寻路，否则只收到警告仍不会移动。
- CompExplosive.ExplosiveRadius（06009CCD）的堆栈扩张为 sqrt((stackCount-1)×explosiveExpandPerStackcount)，不是线性。StartWick（06009CCB）会重置 wickTicksLeft，不能为读档恢复警告再次调用。PostExposeData（06009CC1）保存已启动状态、剩余 tick、instigator 和 customExplosiveRadius；Projectile_Explosive 的 landed / ticksToDetonation 也可作为派生警告来源。
- Pawn_MindState.MindStateTickInterval（06004522）的天气记忆另查本地 Roofed；原型在该入口复用 SkyExposure；后续 Core 与全 DLC 的真实雨天记忆专项及独立读档已通过，见下节。

原型的逐媒介权限、火焰速率、声路权重、强制命令保护、真实 Sprint/引爆与读档证据见 [RimLayers WORK-LOG](../Mods/RimLayers/Docs/WORK-LOG.md)。完整 M2–M5 的剩余要求仍从 PLAN / WORK-LOG 跟踪。


### 原生雨天记忆专项（2026-10-18，同 MVID）

MindStateTickInterval（06004522）在 120 tick 检查天气记忆；CanGainGainThoughtNow（06004523）会排除距游泳不足 7500 tick 的 SoakingWet。单阶段天气记忆在被遮蔽时不新增；进入遮蔽区不会主动删除之前已获得的原生记忆。CurWeatherLerped（17001550）在过渡前半段仍返回旧天气，测试应先满足实际过渡条件，不能只看到 curWeather 已改就断言记忆入口已处理新雨天。

当前 Core Rain 的 weatherThought=SoakingWet，来源 WeatherDefs/Weathers.xml；Thoughts_Memory_Misc.xml 中该记忆持续 0.1 天、仅一个阶段，保留原生思想/教义过滤。RimLayers 的 sky 场景在遮蔽区/露台调用真实思维状态阶段，并保存原生记忆后新进程重载：Core 19 / 9、全 DLC 20 / 9，零失败。完整输入及目录见 [WORK-LOG](../Mods/RimLayers/Docs/WORK-LOG.md) M5。
