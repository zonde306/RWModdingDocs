# 本地环境与 C# 构建

## 1.7 基准与路径

| 用途 | 本机位置 / 事实 |
|---|---|
| 游戏根目录 | `G:/SteamLibrary/steamapps/common/RimWorld` |
| 当前版本 | `Version.txt`：`1.7.4981 rev136`（2026-10-17 核对） |
| 游戏逻辑 | 根目录 `Assembly-CSharp.dll` |
| 启动器 / 宿主 | `RimWorld.exe`、`RimWorld.dll` |
| 运行时配置 | `RimWorld.runtimeconfig.json`、`RimWorld.deps.json` |
| 本地 mod | `Mods/<Name>` |
| 工坊 mod | `G:/SteamLibrary/steamapps/workshop/content/294100/<PublishedFileId>` |
| 原版配置 | `Data/Core/Defs`、`Data/<DLC>/Defs` |
| 当前用户数据候选目录 | `C:/Users/zonre/AppData/LocalLow/Ludeon Studios/RimWorld by Ludeon Studios` |
| 日志 / 配置 / 存档 | 常规目录下 `Logs/Player.log`；配置与存档为 `GenFilePaths.SaveDataFolderPath` 下 `Config/`、`Saves/`。当前 Waylith 日志不随 `-savedatafolder` 重定向，见 `debugging.md` |
| 已安装 SDK | .NET SDK `10.0.401`、`10.0.402`，也存在更老 SDK；新建工程选择 10.x |
| 当前 Harmony | 工坊 `2009463077/1.7/Assemblies/0Harmony.dll`；程序集版本 `3.0.0.0` |

本地 runtimeconfig 声明 `tfm=net10.0`，内置框架版本为 `10.0.13-waylithclr.dev`。游戏自带运行时不等于开发 SDK；构建使用安装的 .NET 10 SDK，运行仍由游戏提供的定制运行时负责。

`UnityEngine.*` 仍可能是方法签名中的类型名。runtimeconfig 中存在 `UnityEngine*=UnityStubs` 等程序集重定向，这是兼容层，不代表游戏仍是 Unity。`UnityStubs.dll`、`Waylith.UnityCompat.dll`、`Waylith.dll` 的引用由用到的类型决定。不要修改游戏运行时开关来掩盖 mod 的引用或 API 错误。

当前安装的主程序集指纹已变化；历史 1.7.4979 的 API 快照和示例构建结果不能视为 1.7.4981 全面回归。当前指纹及本轮已复核成员见 [sources.md](sources.md)。

## 新建工程模板

目录假定为 `<ModRoot>/Source/Example17/Example17.csproj`。输出到 `<ModRoot>/Assemblies`，内容直接放 mod 根目录，不设置多版本目录。完整配套例子见 [recipes.md](recipes.md)。

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <AssemblyName>Example17</AssemblyName>
    <RootNamespace>Example17</RootNamespace>
    <Nullable>disable</Nullable>
    <ImplicitUsings>disable</ImplicitUsings>
    <GenerateAssemblyInfo>true</GenerateAssemblyInfo>
    <AppendTargetFrameworkToOutputPath>false</AppendTargetFrameworkToOutputPath>
    <AppendRuntimeIdentifierToOutputPath>false</AppendRuntimeIdentifierToOutputPath>
    <OutputPath>../../Assemblies/</OutputPath>
    <RimWorldDir Condition="'$(RimWorldDir)' == ''">G:/SteamLibrary/steamapps/common/RimWorld</RimWorldDir>
  </PropertyGroup>
  <ItemGroup>
    <Reference Include="Assembly-CSharp">
      <HintPath>$(RimWorldDir)/Assembly-CSharp.dll</HintPath>
      <Private>false</Private>
    </Reference>
    <Reference Include="UnityStubs">
      <HintPath>$(RimWorldDir)/UnityStubs.dll</HintPath>
      <Private>false</Private>
    </Reference>
    <Reference Include="Waylith.UnityCompat">
      <HintPath>$(RimWorldDir)/Waylith.UnityCompat.dll</HintPath>
      <Private>false</Private>
    </Reference>
  </ItemGroup>
</Project>
```

这是普通类库，不是 exe，不需要复制游戏的 runtimeconfig 或自带的 `System.*.dll`。若编译器提示某类型被转发到另一个程序集，以错误中给出的程序集名增加显式引用，并设置 `Private=false`。直接使用 Waylith 的接口时才增加相应引用。

用 Harmony 时追加以下引用；不用补丁的 mod 不需要该引用或依赖。

```xml
<Reference Include="0Harmony">
  <HintPath>G:/SteamLibrary/steamapps/workshop/content/294100/2009463077/1.7/Assemblies/0Harmony.dll</HintPath>
  <Private>false</Private>
</Reference>
```

`HintPath` 是当前开发机路径，可以替换成可覆盖的 MSBuild 属性；不要让玩家安装在相同盘符。游戏通过启用的 mod 提供依赖。新 mod 仅面向 1.7；维护已有 mod 时先保留其工程布局，再修正实际有问题的引用。

## 构建与输出检查

```powershell
dotnet --version
dotnet build 'Mods/目标Mod/Source/目标项目/目标项目.csproj' -c Release '-p:RimWorldDir=G:/SteamLibrary/steamapps/common/RimWorld'
Get-ChildItem 'Mods/目标Mod/Assemblies' -File
```

如果工程只有本地引用，离线环境可以先执行 `dotnet restore <项目> --ignore-failed-sources`。不能因依赖包缺失而假报构建成功；也不要对已有工程随意禁用全部包源。被 `global.json` 固定到旧 SDK 时，先查看其版本要求。

输出应主要是自己的 DLL 和可选 PDB。若发现 `Assembly-CSharp.dll`、`UnityStubs.dll`、`System.Private.CoreLib.dll`、`0Harmony.dll` 被复制进 mod，先修复 `Private` / 包资产配置，避免加载重复程序集。不要把 `bin/obj` 指向玩家的游戏数据目录。

## 定向反编译流程

有 `mcp__csdecompiler` 时：

1. `status` 确认上下文。用显式 `assemblyPath=<GameRoot>/Assembly-CSharp.dll` 加载，建议别名 `rimworld17`；上下文不是源文件缓存。
2. 碎片名称用 `search_symbols(query=...)`；完整名称用 `resolve_member_id(memberId="Verse.ThingComp")`。
3. 得到类型后 `list_members(typeId=返回的ID)`，确认声明类型、访问级别和重载。`query` 不是 `pattern`。
4. 只对目标成员用 `get_decompiled_source(memberId=...)`。确有需要再找 callers/callees/derived types。
5. 使用返回的实际 ID；MVID 改变后旧 ID 失效，不要把它们作为长期 API 名称写进业务代码。

没有该工具时，用现成 ILSpy/dnSpy 或项目已有的反编译工具，对同一个根目录 DLL 进行相同步骤。不要默认下载整份非官方反编译仓库。查不到方法时先确认命名空间和 declaring type，例如 `JobMaker`、`RecipeWorker` 在 **Verse**，不是 `Verse.AI` / `RimWorld`。
