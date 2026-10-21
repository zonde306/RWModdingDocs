# 资源、渲染与翻译

## 路径与 1.7 打包资源

mod 的常规纹理放 `Textures/`，声音放 `Sounds/`；XML 和 ContentFinder 使用内容相对路径，通常不写文件扩展名或前面的 `Textures/` / `Sounds/`。

```text
Textures/Things/Building/EX17/Counter.png
  → texPath: Things/Building/EX17/Counter
Sounds/EX17/CounterClick.ogg
  → clipPath: EX17/CounterClick
```

路径用 `/`，保持大小写一致；不要把盘符或开发机绝对路径写入 Def。开发时新增普通 PNG 最直接；导入格式、音频支持和特殊资源以当前加载器为准。

原版许多资源在游戏根目录 **Assets.zip** 内，没有 `Data/Core/Textures` 并不代表原版图像缺失。本地索引已确认：

```text
Resources/Textures/Things/Building/Production/CraftingSpot.psd.dds
Resources/Textures/Things/Item/Resource/Steel/Steel_a.psd.dds
Resources/Textures/Things/Item/Resource/Steel/Steel_b.psd.dds
Resources/Textures/Things/Item/Resource/Steel/Steel_c.psd.dds
```

XML 仍写 `Things/Building/Production/CraftingSpot` 等逻辑路径；不要写 zip 内部文件名或 `.psd.dds`。查原版资源先查其 Def 的 texPath，再按需列出 zip 中相关条目，不要解压整个资源包。原版 DDS/打包细节不必复制到普通 mod 工程。

## graphicData 与资源选择

| 场景 | 常见字段 / 类型 | 注意 |
|---|---|---|
| 单图物品/建筑 | `graphicData.texPath` + `Graphic_Single` | 路径对应实际单图 |
| 堆叠数量图 | `Graphic_StackCount` | 目录内多张图；原版 Steel 就是此结构 |
| 多方向 | `Graphic_Multi` | 常见 `_north/_east/_south/_west` 约定，按原版同类例子核对 |
| 随机变体 | `Graphic_Random` 等 | 目标通常为变体目录，不是任意文件 |
| 绘制尺寸 | `drawSize` | 与占地 `ThingDef.size` 不同 |
| 材质/颜色 | `shaderType`、`color`、`colorTwo` 等 | 是否生效取决于 Graphic 与 shader |
| 地图渲染方式 | `drawerType`、`altitudeLayer` | 静态 mesh、实时和混合路径不同，影响更新/脏标记 |

`Graphic_Single` 不能直接拿原版 Steel 的目录当成单图文件。图像发紫/缺图时先检查 Graphic 类型与路径结构，再考虑 shader。透明边缘、中心位置、朝向和 drawSize 要在地图上实际检查。

## C# 资源与 Pawn 渲染

`ContentFinder<UnityEngine.Texture2D>.Get("UI/...")`、`GraphicDatabase`、`MaterialPool` 是常见游戏侧入口。加载共享资源用 `[StaticConstructorOnStartup]` 等主线程安全时机，避免 Mod 构造或后台线程加载图形。不要在每帧/每 tick 反复取纹理、生成 Material 或重建 Graphic。

`UnityEngine` 命名空间还存在于兼容层，但底层已是 Waylith。原版有 `Data/Core/Shaders`，本地包含 GLSL 文件；`Assets.zip` 也包含缓存着色器数据。旧 Unity AssetBundle、Unity shader 编译流程、私有 Material/Camera 成员不能直接套用到 1.7。

Pawn 外观修改先找当前 `PawnRenderTree`、`PawnRenderNode`、`PawnRenderNodeWorker` 和对应配置。衣物图、身体节点、头发、附加绘制及其缓存各有更新路径；不要用旧版的整段 PawnRenderer Harmony 当作通用方案。需要新着色器、资源包打包或节点 worker 覆写时，定向核实该领域的 1.7 实现；本知识库不声称提供完整引擎 SDK。

## 声音定义

```xml
<Defs>
  <SoundDef>
    <defName>EX17_Click</defName>
    <context>MapOnly</context>
    <subSounds>
      <li>
        <grains>
          <li Class="AudioGrain_Clip"><clipPath>EX17/CounterClick</clipPath></li>
        </grains>
        <volumeRange>10~10</volumeRange>
      </li>
    </subSounds>
  </SoundDef>
</Defs>
```

这是格式示例，使用前必须提供 `Sounds/EX17/CounterClick.ogg`。持续声音还需要 sustain、生命周期和停止逻辑；一次性地图音与 UI 音的播放入口不同。参考 `Data/Core/Defs/SoundDefs` 同类 SoundDef 和调用点，不在 tick 中反复启动独立 sustainer。

## Keyed 翻译

文件：`Languages/English/Keyed/Example.xml`。

```xml
<LanguageData>
  <EX17_EnableCounting>Enable counting</EX17_EnableCounting>
  <EX17_Count>Count: {0}</EX17_Count>
</LanguageData>
```

文件：`Languages/ChineseSimplified/Keyed/Example.xml`。

```xml
<LanguageData>
  <EX17_EnableCounting>启用计数</EX17_EnableCounting>
  <EX17_Count>计数：{0}</EX17_Count>
</LanguageData>
```

C# 用 `"EX17_Count".Translate(count)`，需要 `using Verse;`。英文作为基础文本同时提供，key 使用 mod 前缀，保留 `{0}` 等占位符。只有在调用系统本身提供对应参数时才使用命名占位符/grammar，不把它们与普通 string.Format 混为一谈。

## DefInjected 翻译

文件：`Languages/ChineseSimplified/DefInjected/ThingDef/Example.xml`。

```xml
<LanguageData>
  <EX17_Counter.label>示例计数器</EX17_Counter.label>
  <EX17_Counter.description>用于检查组件与存档的示例建筑。</EX17_Counter.description>
</LanguageData>
```

目录中的 `ThingDef` 是 Def 类型，XML key 使用 `defName.field`，不是显示 label。其他类型放自己的 DefType 目录；列表嵌套路径可能使用索引或可识别字段，按同类原版翻译核对，不能臆造路径。

单纯 Def 翻译通常只写可翻译字段，不翻译 defName、packageId、类型名、资源路径、enum、引用值或 XML 模板名。文件名可自行组织，加载依据是目录和 key；同一语言中不要重复定义同一 key。

已有游戏语言不需要随 mod 复制整份 LanguageInfo.xml 或完整原版翻译。`Strings` 下的名字/规则文本有独立系统约定，每行格式按对应原版文件处理。全部文本保存为 UTF-8，并在英文、简中两种语言实际检查占位符与换行。


## 1.7.4981：原生可改键与下层显示投影

2026-10-18，Core `Defs/Misc/KeyBindings/KeyBindings.xml` 的抽象 `GameMapKeyBinding` 已配置 `GameMapCamera` 类别；自有 KeyBindingDef 可用 `defaultKeyCodeA` 指定初始键，通过 `KeyDownEvent` 检测、`MainKeyLabel` 显示用户重绑后的键名，DefInjected 放 `KeyBindingDef`。不要用固定绘制文本假装可改键。当前 Core 的 PageUp/Down 被缩放占用，方括号被开发菜单占用；RimLayers 采用 F7/F8，并保留原版重绑定入口。

若相机在后续 GUI 流程会处理同一事件，可在 `UIRoot_Play.UIRootOnGUI` 的高优先级前置入口处理并 `Event.Use()`；输入框有 keyboardControl 或强制暂停窗口开启时不抢切层。原生输入与当前界面的实际表现都需验证。

同 Map 的背景显示可复用真实 SectionLayer 的已完成 SubMesh，以 Matrix4x4 移位绘制，不移动 Thing 或切换模拟所用的 Map/查看层。分区起点需对齐原生17格分块；原生迷雾应一起投影。动态 Pawn/Corpse 走 `EnsureInitialized → ParallelPreDraw → Draw`。其原生输入必须保留真实位置：躺卧/床位等路径可能自行重算位置；若在入参和最终网格两处都投影，会重复偏移。只在受限背景 pass 的最终网格提交处投影，不改 Position。可见背景与可选目标分开：普通点选限查看层，攻击模式返回真实 Thing；瞄准高亮只改绘制副本，不替换业务目标引用。当前实际实现及已查看画面见 [RimLayers 0.1.1](../Mods/RimLayers/Docs/USABILITY-VALIDATION.md)，不将静态背景方案扩张为全部第三方实时特效或三维渲染框架。


### 0.1.2：楼面遮盖、动态实体与跨层瞄准

`TerrainDef.PostLoad` 的材质队列为 `2000 + renderPrecedence`。背景只降低 y 不足以保证上层楼面盖住它；可复制材质，给各背景深度独立队列并压平原生高度，避免高原生 Altitude 的 Pawn 仍穿过前景。不要改 `TerrainGrid.GetMaterial` 或共享材质本身。

当前 UnityStubs 的 `Graphics.DrawMeshCore(Mesh, Matrix4x4, Material, int, int, MaterialPropertyBlock, int)` 是 MaterialPropertyBlock 延迟绘制的共同提交端；`DrawPropertyBlock` 的十参数 DrawMesh 重载独立提交，**不经过该 Core**。若同时补丁外部包装和共同提交会重复投影。原生 DrawEquipmentAiming 直接调用 Graphics.DrawMesh，不经过 GenDraw；只补 GenDraw 会漏掉武器与部分自定义 PostDraw。即时 DrawMeshNow 用于其他绘制上下文，不应被背景作用域投影。

`Corpse.DynamicDrawPhaseAt` 复用 InnerPawn 的原生绘制，包括 lying 姿态。只遍历 mapPawns 无法显示尸体；可在限定层/视野/迷雾/上层遮盖后遍历 DynamicDrawManager.DrawThings。弹丸、Skyfaller 和 PawnFlyer 等已有专用显示的对象应保持原有所有权。

原生 `PawnRenderUtility.DrawEquipmentAndApparelExtras` 用实际目标 DrawPos 减 Pawn.DrawPos 得到角度及装备偏移，`Pawn_RotationTracker.Face` 同样如此。跨存储带时须在这些领域入口用逻辑向量，保留 Verb.AimAngleOverride。只读 Nanami Startrail 的 Rendering.cs 还发现其 worker 直接以 70/290 度调用 DrawEquipmentAiming；共用入口可通过装备 ParentHolder 的 Pawn_EquipmentTracker 找回真实射手，仅对有效跨层 Stance_Busy 修正角度，不改同层/闲置姿态或原生 NeverAimWeapon/Portrait 范围。

原生 CameraJumper.JumpLocalInternal 在 TryJumpInternal 确定当前 Map 后选择 Cut 或 Pan。跨层必须在该入口前切查看层并停旧 CameraPanner/速度；同层继续原生 Pan。若在 TryJumpInternal 后才切层，原生相机裁剪可能已经使用旧层。实际双向画面、生成纹理与重启证据见 [0.1.2 显示验收](../Mods/RimLayers/Docs/DISPLAY-VALIDATION.md)。

## 1.7.4982：GPU 地图捕获与 UI 前合成

2026-10-21，当前 `Assembly-CSharp.dll` MVID 为 `484efdc5765b463ca9cc54a6131e38f0`。完整地图截取应在原生 `RimWorldWaylith.Rendering.MapRenderFeature.CaptureFrame` 已生成 `MeshDrawFrame` 后，借用其 `Draws + PersistentDraws`、背景颜色及太阳方向；不能只复制待提交的 `Graphics.DrawMesh` 命令。后者漏掉持久 Section 绘制，实测会缺少光照遮罩，导致退出截图时明暗跳变。

禁用自动渲染的独立 Camera 可 `CopyFrom(Find.Camera)` 保持视图。当前 `CameraTextureRenderer.Current` 的原生实现有公开 `RenderItems(RenderTexture, List<FinalDrawItem>, Camera, Rect, Color?, System.Numerics.Vector3?)`；借用完整提交项后用 `DrawSortKeyComparer.Instance` 排序，传入原帧背景和 SunDirection，在 GPU 中绘入目标纹理。借用列表不能释放或修改原绘制项。此为版本相关的原生后端接口，升级时须重新核实；失败须回退原操作而不是阻塞输入。

当前 UnityStubs 的显式 `Camera.Render()` 只复制 pending draw commands；其 camera callbacks 在原生离屏渲染前收集即时命令。可在受限 callback 中用私有 Transparent 材质的 `SetPass` 与 `Graphics.DrawMeshNow` 合成两张纹理，cullingMask=0 排除地图命令，并始终解除 callback / 恢复 targetTexture 与 RenderTexture.active。此用途保留 GPU 路径，无 CPU 读回。

`Graphics.DrawTexture` 的 RT GPU 快速路径只接受白色 tint；不能直接用带 alpha 的颜色当 GPU 交叉淡入。Transparent 材质的 `_MainTex_ST` 和 `_Color` 可用于中心缩放、UV 平移与透明混合，纹理 wrapMode=Clamp 防止越界重复。

全屏地图合成在 `UIRoot_Play.UIRootOnGUI` 前绘制，固定 UI 留给原生后续流程。动画期间禁用地图附着标签而非整个 UI；处理地图输入前结束视觉偏移。相关实机逐帧检查、独立配置和性能记录见 [RimLayers 换层验收](../Mods/RimLayers/Docs/TRANSITION-VALIDATION.md)。
