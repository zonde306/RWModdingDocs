# RimWorld 1.7 mod 开发：agent 入口

本工作区仅面向 **RimWorld 1.7**。先读 `docs/README.md` 的任务路由，再按需读对应专题；不要每次扫描整个游戏和创意工坊。

## 已确认的基准

- 游戏目录：`G:/SteamLibrary/steamapps/common/RimWorld`。
- 创意工坊：`G:/SteamLibrary/steamapps/workshop/content/294100`。
- 文档基准：`1.7.4979 rev109`，核对日期 `2026-10-15`。
- 当前运行时是 **Waylith / .NET 10**；新工程使用 `net10.0`。
- 主程序集是游戏根目录的 `Assembly-CSharp.dll`。不要引用残留 Unity 安装目录中的同名文件。
- Unity 名称仍可能出现在 API 中；兼容程序集包括 `UnityStubs.dll`、`Waylith.UnityCompat.dll`。按实际类型归属添加引用，见 `docs/environment.md`。
- 本机 Harmony 的 1.7 目录使用 Harmony 3；不要混用旧版本 DLL。详见 `docs/harmony.md`。
- 最新日志位置在`C:\Users\zonre\AppData\LocalLow\Ludeon Studios\RimWorld by Ludeon Studios\Logs\Player.log`
- 已加载的MOD列表在`C:\Users\zonre\AppData\LocalLow\Ludeon Studios\RimWorld by Ludeon Studios\Config\ModsConfig.xml`

## 默认工作方式

1. 明确目标 mod 根目录，读取该目录适用的 `AGENTS.md`、`About/About.xml`、`LoadFolders.xml` 和项目文件。
2. 从 `docs/README.md` 选择专题。常见 API 优先查 `docs/csharp-api.md` 和 `docs/api-signatures.md`。
3. XML 改动先在 `Data/Core/Defs` 或对应 DLC 找一个最接近的例子，并只读取其父模板链与相关字段。
4. 本文档没有覆盖的行为才定向反编译当前 1.7 成员；先查类型/成员签名，再读方法体。不要导出整个程序集。
5. 优先采用 Def、原版配置和组合式 Comp；需要新行为时再用子类或 Harmony。用最小作用范围实现需求。
6. 做相应的 XML/构建检查；涉及游戏行为、存档或补丁的修改必须另记游戏内验证状态。编译通过不代表已在游戏里验证。
7. 把新核实且可复用的 1.7 结论补到对应专题，并在 `docs/sources.md` 记下基准与证据入口。

## 修改边界

- 原版 `Data/`、游戏 DLL 和运行时文件是参考资料。正常 mod 开发在目标 mod 内增加 Def/Patch/代码，不直接改原版。
- 默认新 mod 放 `Mods/<ModName>/`。修改已有 mod 时保持其目录、命名、包 ID 和构建习惯；不要同时改本地与创意工坊副本。
- 创意工坊数字目录不是 `packageId`。以 `About.xml` 为准；Steam 更新可能覆盖其中的本地改动。
- 此根 `AGENTS.md` 自动覆盖游戏目录的子目录；**不会自动覆盖同级的创意工坊目录**。在创意工坊任务中应显式读取本入口，且遵守目标目录自己的规则。
- 不因一次任务扫描整个工坊、不搬运别的 mod 的私有数据、不修改玩家存档、配置和启用列表来“方便测试”。如需运行，优先用独立测试数据目录。
- 所有新文本使用 UTF-8；示例中的 `Example17` / `example.agentdemo` 必须替换为目标 mod 的命名。

## 常见失误，开工前排除

- `supportedVersions` 只声明 `1.7`；它不会转换旧 API，也不会自动选择正确编译目标。
- `Name` / `ParentName` 是 XML 模板；`defName` 是运行时定义标识；`Class` 是 XML 节点的 C# 数据类型；`thingClass` 是实体类型。四者不可互换。
- XML 补丁先于继承展开执行；XPath 看不到只存在于父模板里的子节点。`Replace` 不会自动创建缺失路径。
- `Mod` 构造时 Def 数据库尚未准备好。资源加载、Def 缓存和地图状态分别遵守自己的生命周期。
- 1.7 有 `TickInterval(int delta)` / `CompTickInterval(int delta)` 等入口。不要把每次回调等同于一个 tick，也不要盲目把旧 tick override 改名。
- 不保存 Def 的深拷贝；Def 用 `Scribe_Defs`，已有实体引用用 `Scribe_References`，自有对象才用 `Scribe_Deep`。
- 禁止往 mod 输出目录复制游戏 DLL、系统运行时 DLL 或另一份 `0Harmony.dll`。
- 修改 DLL、Harmony 或启动初始化后，重启游戏进行有效验证；Def 热重载不能替代它。

## 按需检索

```powershell
# 在游戏根目录运行；限定到目标目录和文件类型。
rg -n '目标defName|ParentName="目标模板"' Data/Core/Defs -g '*.xml'
rg -n '目标符号' Mods/目标Mod/Source -g '*.cs'
rg -n '关键词' docs -g '*.md'
```

反编译器使用步骤、准确路径、构建命令和测试矩阵分别见 `docs/environment.md`、`docs/debugging.md`。以本地 1.7 程序集和原版数据为最终依据；不要为了“少查一次”编造 API 或沿用未核实的旧版结论。
