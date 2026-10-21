# Mod 文件结构、元数据与加载顺序

## 最小的 1.7 单版本布局

```text
MyMod/
  About/
    About.xml
    Preview.png                  # 可选
  Defs/                          # 可选：新定义
  Patches/                       # 可选：修改合并后的 Def XML
  Assemblies/                    # 可选：自己的 DLL/PDB
  Textures/                      # 可选
  Sounds/                        # 可选
  Languages/
    English/Keyed/
    ChineseSimplified/Keyed/
    ChineseSimplified/DefInjected/ThingDef/
  Source/                        # 开发源码，游戏不自动编译
```

对只支持 1.7 的新 mod，根目录内容即可，`LoadFolders.xml` 可省略。不要为了“版本兼容”生成其他版本目录。已有 mod 保留其既定布局。

```xml
<?xml version="1.0" encoding="utf-8"?>
<ModMetaData>
  <name>Example 1.7</name>
  <author>Example Author</author>
  <packageId>example.agentdemo</packageId>
  <supportedVersions><li>1.7</li></supportedVersions>
  <description>A small RimWorld 1.7 example.</description>
</ModMetaData>
```

## About.xml 字段

| 字段 | 类型 / 作用 |
|---|---|
| `name`, `author`, `description`, `url` | 展示元数据；描述中的 XML 特殊字符必须转义 |
| `packageId` | 稳定唯一包标识；依赖和排序使用它，建议小写并保持不变 |
| `supportedVersions` | `<li>1.7</li>`；声明兼容，不验证 API 或选择构建目标 |
| `modDependencies` | `<li>` 含 packageId、displayName、下载地址；表示必须启用的依赖 |
| `loadAfter`, `loadBefore` | packageId 列表；排序约束不等于依赖声明 |
| `incompatibleWith` | 无法同时使用的包列表；不是修复兼容性的替代方案 |

包 ID 沿用游戏约束：字母/数字/点、至少一个点、不含空格、不首尾点、不连续点、不使用 Ludeon 保留命名，长度不超过 60。不要使用工坊数字 ID 代替 packageId；不要为现有 mod 随意换 ID。

使用 Harmony 的元数据可追加：

```xml
<modDependencies>
  <li>
    <packageId>brrainz.harmony</packageId>
    <displayName>Harmony</displayName>
    <steamWorkshopUrl>steam://url/CommunityFilePage/2009463077</steamWorkshopUrl>
  </li>
</modDependencies>
<loadAfter><li>brrainz.harmony</li></loadAfter>
```

排序需要以实际依赖关系为准，不把所有 mod 都写进 loadAfter。`loadAfter` 不会自动安装依赖；没有激活的 mod 也不会因为写了排序标签就被激活。

## LoadFolders.xml：显式选择内容根

要按功能分目录时，可以只定义 1.7：

```xml
<loadFolders>
  <v1.7>
    <li>Common</li>
    <li>1.7</li>
    <li IfModActive="ludeon.rimworld.biotech">Compatibility/Biotech</li>
  </v1.7>
</loadFolders>
```

每个被选中的内容根都可以含 `Defs/`、`Patches/`、`Assemblies/` 等。`Common` 和 `1.7` 必须实际存在；`About/`、`LoadFolders.xml` 本身仍放 mod 根目录。若还需要根目录内容，显式加入 `<li>/</li>`。

已核实 `ModLoadFolders.LoadDataFromXmlCustom` 支持：

| 属性 | 判断内容 | 多个包 ID |
|---|---|---|
| `IfModActive` | 任一指定包启用 | 逗号分隔，OR |
| `IfModActiveAll` | 全部指定包启用 | 逗号分隔，AND |
| `IfModNotActive` | 指定包全部未启用 | 任一启用则不加载 |

多个属性同时存在时共同约束；这些是 **packageId**。文件夹条件只控制内容是否加载，不能替代依赖和排序。

显式列表会反向存入 `foldersToLoadDescendingOrder`，后列目录具有更高的同路径文件优先级。`GetAllFilesForMod` 按内容相对路径去重；不要把“同一路径文件覆盖”误解为“重复 defName 会合并”。不同路径里的同名 Def 仍然是重复定义风险。

没有显式列表时，当前加载器会尝试匹配版本目录，再选可用版本目录，随后加入 `Common` 和根目录。这个回退行为会使旧目录被加载；排错时查看实际 `foldersToLoadDescendingOrder`，不能只看目录名或 About 的支持版本。新建单版本 mod 直接采用最小布局。

## 内容格式边界

| 路径 | 根 / 格式 | 说明 |
|---|---|---|
| `About/About.xml` | `<ModMetaData>` | 发现与元数据 |
| `LoadFolders.xml` | `<loadFolders>` | 内容选择 |
| `Defs/**/*.xml` | `<Defs>` | 直接子节点是 Def 或模板 |
| `Patches/**/*.xml` | `<Patch>` | `<Operation Class="...">` |
| `Languages/<Language>/Keyed/*.xml` | `<LanguageData>` | 文本 key → 字符串 |
| `Languages/<Language>/DefInjected/<DefType>/*.xml` | `<LanguageData>` | `defName.field` → 翻译 |
| `Languages/<Language>/Strings/**` | 通常 txt | 按原版对应系统约定读取 |
| `Assemblies/*.dll` | 托管类库 | 已编译代码；Source 不会自动生效 |
| `Textures/**`、`Sounds/**` | 资源 | 由内容加载器解析资源路径 |

`.rws` 存档、玩家 `ModsConfig.xml`、`Prefs.xml` 和 `Mod_*.xml` 设置属于用户数据；不是 mod 随包发布的内容。1.7 支持打包资源，原版部分内容在根目录 `Assets.zip` 中，文件系统未找到不等于资源不存在，见 [assets-localization.md](assets-localization.md)。
