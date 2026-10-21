# RimWorld 1.7：agent 开发知识库

用途：让 agent 在当前安装上开发 mod 时，直接获得架构、格式和 API 的工作上下文。入口是根目录 [AGENTS.md](../AGENTS.md)，本页负责按任务分流。**不需要通读全部文档。**

## 按任务读取

| 当前任务 | 优先读取 | 必要时追加 |
|---|---|---|
| 新建 mod / 编译失败 / 找引用 | [environment.md](environment.md)、[mod-layout.md](mod-layout.md) | [recipes.md](recipes.md) |
| 改原版数值或兼容另一 mod | [xml-patches.md](xml-patches.md) | [xml-defs.md](xml-defs.md) |
| 新增物品、建筑、武器、配方 | [xml-defs.md](xml-defs.md) | [assets-localization.md](assets-localization.md) |
| 新行为、Comp、全局状态、设置 | [architecture.md](architecture.md)、[csharp-api.md](csharp-api.md) | [persistence.md](persistence.md)、[recipes.md](recipes.md) |
| AI 工作、Job、Hediff、事件 | [csharp-api.md](csharp-api.md) | [api-signatures.md](api-signatures.md) |
| 多层设计收口 / 原始需求对照 / 实施顺序 | [multilayer-readiness.md](multilayer-readiness.md) | [multilayer.md](multilayer.md)、[sources.md](sources.md) |
| 多层地图 / 跨图寻路、搬运、工作、战斗 | [multilayer.md](multilayer.md) | [architecture.md](architecture.md)、[persistence.md](persistence.md) |
| 连续跨层 / 需求选物与家具 / 中间层目标保持 | [multilayer-routing.md](multilayer-routing.md) | [multilayer.md](multilayer.md)、[csharp-api.md](csharp-api.md) |
| 多层实现前置 / 携物转移 / 楼层 Map 与空洞 | [multilayer-foundations.md](multilayer-foundations.md) | [multilayer-routing.md](multilayer-routing.md)、[environment.md](environment.md) |
| 本地多层 mod 参考 / Baal.SecondFloor | [multilayer-reference-baal.md](multilayer-reference-baal.md) | [multilayer.md](multilayer.md)、[sources.md](sources.md) |
| 单 Map 多楼层 / IsoRim ProjectZ / 叠层渲染 | [multilayer-reference-isorim.md](multilayer-reference-isorim.md) | [multilayer.md](multilayer.md)、[multilayer-routing.md](multilayer-routing.md) |
| 单 Map 选物/选料 / 格子工作漏选 / 楼梯动态断路 | [multilayer-queries.md](multilayer-queries.md) | [multilayer-reference-isorim.md](multilayer-reference-isorim.md)、[multilayer-routing.md](multilayer-routing.md) |
| 跨层选目标/射位 / 弹丸存档 / 护盾与爆炸 | [multilayer-combat.md](multilayer-combat.md) | [multilayer-reference-isorim.md](multilayer-reference-isorim.md)、[multilayer-queries.md](multilayer-queries.md) |
| 严格屋顶施工 / 完工提交 / 连锁失效 / 天空与出口 | [multilayer-structure.md](multilayer-structure.md) | [multilayer-foundations.md](multilayer-foundations.md)、[multilayer-combat.md](multilayer-combat.md) |
| 多层电力/设施 / 空气/天气/温度 / 游戏光照 | [multilayer-services.md](multilayer-services.md) | [multilayer-structure.md](multilayer-structure.md)、[multilayer-reference-isorim.md](multilayer-reference-isorim.md) |
| 任务续行仲裁 / 远程声明 / 牵绳/机体/Lord | [multilayer-coordination.md](multilayer-coordination.md) | [multilayer-routing.md](multilayer-routing.md)、[multilayer-queries.md](multilayer-queries.md) |
| Odyssey 逆重飞船 / 起降 / 跨地图载荷 | [odyssey-gravship.md](odyssey-gravship.md) | [persistence.md](persistence.md)、[environment.md](environment.md) |
| Harmony 补丁 / IL | [harmony.md](harmony.md) | [architecture.md](architecture.md) |
| 翻译、图标、纹理、渲染 | [assets-localization.md](assets-localization.md) | [environment.md](environment.md) |
| 存档损坏、读档后状态丢失 | [persistence.md](persistence.md)、[debugging.md](debugging.md) | [api-signatures.md](api-signatures.md) |
| 自动化测试 / 场景断言 / 重启读档 / 测试报告 | [testing.md](testing.md) | [debugging.md](debugging.md)、[environment.md](environment.md) |
| 红字、加载失败、卡顿 | [debugging.md](debugging.md) | 对应专题 |
| 性能优化 / 旧优化 MOD 迁移 | [performance.md](performance.md)、[debugging.md](debugging.md) | [harmony.md](harmony.md)、目标 MOD 的研究资料 |
| 需要确认文档依据 / 游戏更新 | [sources.md](sources.md) | 只复查被改动的成员 |

## 快速事实

- 当前安装：`1.7.4982 rev1280`，核对 `2026-10-19`；历史文档基准为 `1.7.4979 rev109`。指纹和本轮有限复核范围见 [sources.md](sources.md)，旧专题不视为已全面升级。用户已指定只覆盖 1.7。
- 引擎：Waylith；运行时：`net10.0`。游戏逻辑仍主要位于 `Assembly-CSharp.dll` 的 `Verse` / `RimWorld` 命名空间。
- 原版 Def：`Data/Core/Defs`；DLC：`Data/{Royalty,Ideology,Biotech,Anomaly,Odyssey}`。
- 主 DLL：`<GameRoot>/Assembly-CSharp.dll`。不要在旧 `*_Data/Managed` 目录寻找构建基准。
- Mod = 元数据 + 选中的内容目录 + 可选程序集。内容由 Def 描述，运行时实例由 Thing/Pawn/World 等持有。
- 默认选择顺序：已有 XML 字段 → XML Patch → Comp/worker/子类 → Harmony。
- `60` tick 是一个游戏秒；Rare 为 `250` tick，Long 为 `2000` tick。暂停、加速和现实时间必须区分。
- 修改 DLL 后重启；只重载 Def 不会重新加载程序集和重新安装 Harmony。

## 阅读与证据约定

- 文档中的相对游戏路径均相对于 `G:/SteamLibrary/steamapps/common/RimWorld`；`<ModRoot>` 是目标 mod 的根目录。
- 标为“已核实”的条目来自本地原版文件或当前程序集；“建议”是工程实践；“骨架”必须按需求补全。
- [api-signatures.md](api-signatures.md) 是选定类型的元数据快照，不是完整 SDK。它保留参数顺序/类型，但不列默认参数值和所有泛型约束；可直接抄用的完整代码放在 [recipes.md](recipes.md)。
- 只在文档缺少目标成员、构建报错、行为与描述不符，或游戏 DLL 指纹变化时进行定向探索。看到 `hasMore`、输出截断或缺少重载时，不能推断“没有这个 API”。
- 一次任务最终说明应包括：改动文件、行为变化、构建/XML 检查结果、游戏内是否验证、存档或兼容性影响。

## 维护方式

新增知识写入对应专题；只在这里添加路由，不把所有内容塞进 `AGENTS.md`。不要缓存全量反编译源码，也不要建立需要每次读取数万行的单文件“大全”。更新流程见 [sources.md](sources.md)。
