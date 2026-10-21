# Harmony 3：1.7 的运行时补丁

## 当前安装基准

本机工坊 `2009463077/1.7/Assemblies/0Harmony.dll` 的程序集身份为 `0Harmony, Version=3.0.0.0`，对应 mod 的 About 标注 `3.0.1.0`。两种版本号代表不同元数据，不必相等。1.7 项目引用这个目录并声明 `brrainz.harmony` 依赖；不要因为已有旧 mod 项目引用 1.6 目录就继续沿用。

优先选择公开扩展点和 XML。需要补丁时，先明确 **目标声明类型、完整重载、调用时机、影响对象范围**。调 Def 数值通常不需要 Harmony。

## 初始化与精确目标

```csharp
using HarmonyLib;
using Verse;

namespace Example17;

public sealed class ExampleHarmonyMod : Mod
{
    public ExampleHarmonyMod(ModContentPack content) : base(content)
    {
        new Harmony("example.agentdemo.patches").PatchAll(typeof(ExampleHarmonyMod).Assembly);
    }
}

public sealed class InspectNoteExtension : DefModExtension
{
    public string noteKey;
}

[HarmonyPatch(typeof(ThingWithComps), nameof(ThingWithComps.GetInspectString))]
internal static class Patch_InspectNote
{
    private static void Postfix(ThingWithComps __instance, ref string __result)
    {
        var data = __instance.def.GetModExtension<InspectNoteExtension>();
        if (string.IsNullOrEmpty(data?.noteKey)) return;
        string note = data.noteKey.Translate();
        __result = string.IsNullOrEmpty(__result) ? note : __result + "\n" + note;
    }
}
```

该完整示例只修改明确附有 extension 的 Def；需在目标 Def 的 modExtensions 中指定 `Example17.InspectNoteExtension` 与 noteKey，并提供对应 Keyed 文本。实际项目已有 Mod 类时，把 PatchAll 放进已有初始化入口，不额外复制一个 Mod 子类。只需追加检查文本的自有 Comp 通常直接用 CompInspectStringExtra 即可；这里用于演示补丁契约。

Harmony ID 保持稳定且全局唯一，通常由 packageId 派生。只安装一次补丁；不要同时在 Mod 构造和启动静态构造 PatchAll。同一程序集的范围显式指定，避免无参 PatchAll 的调用者程序集推断问题。[Harmony 3 API](https://harmony.pardeike.net/v3/api/HarmonyLib.Harmony.html)

有重载时在属性中指定参数类型数组，或通过 `TargetMethod()` 返回已确认的 `MethodBase`。property 需要区分 getter/setter；不要用同名方法“碰运气”。找目标的流程见 [environment.md](environment.md)。

## 补丁参数与执行语义

| 参数 / 补丁 | 契约 |
|---|---|
| `__instance` | 原方法的 this；静态目标不要假设有实例 |
| `__result` / `ref __result` | 读/改返回值，类型应匹配；void 目标没有普通返回值 |
| 原参数名 / `__0`、`__1` | 按名字或位置注入；ref 用于修改参数 |
| `__state` | 同一个补丁类的 Prefix 与 Postfix/Finalizer 共享每次调用状态 |
| `___fieldName` | 三下划线字段注入；依赖实现细节，升级容易失效 |
| Prefix `bool` 返回 | 控制是否运行原方法，不是原方法的 bool 结果 |
| Postfix | 原方法完成或被跳过后执行；前面的异常可能阻止执行 |
| Finalizer | 处理异常路径的必要清理；不要靠吞异常掩盖错误 |

Prefix 返回 false 会跳过原方法，并影响后续被 Harmony 判断能影响原方法的 Prefix；其他 Prefix、Postfix、Finalizer 的调度仍有规则。不要把“返回 false”当成只有本 mod 生效的局部优化。[Prefix](https://harmony.pardeike.net/v3/articles/patching-prefix.html)

Postfix 可修改结果；对 `IEnumerable<T>` 可使用 pass-through 返回迭代结果，保留原序列并按需求追加/过滤。Postfix 不是异常时一定执行的 finally。[Postfix](https://harmony.pardeike.net/v3/articles/patching-postfix.html)

注入复杂值类型、ref return、特殊参数名冲突时，按实际 Harmony 3 规则核实；不要从 Harmony 2 的宽松行为推断当前绑定成功。[Injected values](https://harmony.pardeike.net/v3/articles/patching-injections.html)

## 可选依赖与排序

- About 的 loadAfter/loadBefore 控制 mod 内容顺序；HarmonyPriority / HarmonyBefore / HarmonyAfter 控制补丁相对顺序，后者使用 **Harmony ID**，不必等于 packageId。
- 补丁本身也要限制到目标 Def/Comp/场景，防止给其他 mod 的对象全部改变行为。
- 对可选 mod 的类型，直接写 `typeof(OptionalType)` 可能在检测条件前就使类型加载失败。将兼容代码放条件内容目录/独立程序集，或在确认存在后用名字解析并手动注册。
- 选中的 DLL 在反射取类型阶段就要能解析依赖；只在 Prefix 里 `if (!ModActive)` 不能修复这个阶段的缺依赖。
- 不要在新 mod 中随包附另一份 `0Harmony.dll`；使用共同的 Harmony mod。

## Transpiler 的最低要求

仅在其他扩展点无法表达局部修改时使用。它重写 IL，在安装补丁时运行，也可能在其他补丁改变链条时重新运行，不是每次游戏方法调用都执行。

- 依据可识别的字段/方法/指令邻接结构定位，不按固定偏移、固定行号或某个整数的第 N 次出现修改。
- 先检查匹配数量，构造完整的成功结果再返回；匹配失败时明确报一次带目标信息的诊断，并按设计回退到未修改指令或停止安装，避免留下半截改写。
- 保留 labels、exception blocks 与栈平衡；新增分支、局部变量或返回值时核对所有路径。
- 保存可复现的匹配依据（目标 build、方法、语义锚点），并与相关 mod 一起测试。

官方参考：[Rewrite instructions](https://harmony.pardeike.net/v3/articles/patching-transpiler.html)。文档中的旧 RimWorld 样例路径不适用于本地 1.7；程序集位置始终使用本知识库的环境基准。

## 验证与排错

构建成功只能证明引用和 C# 签名；目标查找、IL 匹配和 PatchAll 都在游戏运行期发生。重启后确认补丁无安装错误，再测试命中对象、不命中对象和其他 mod 的共存行为。

冲突定位可用 `Harmony.GetPatchInfo(MethodBase)` 查看所有者和排序，`Harmony.GetPatchedMethods()` 查看自己安装的目标；按本机 API 签名核对。不要全局 `UnpatchAll()` 清掉其他 mod 补丁；需要撤销时限定自己的 Harmony ID。


## 1.7.4981：静态替换与喂婴回调

2026-10-17 在 animal-hardworking 迁移中实际安装 Harmony 3 补丁核实：

- 将实例方法调用替换成静态辅助方法时，即使参数和返回值兼容，也必须把原 `callvirt` 改为 `call`。`GenRecipe.PostProcessProduct` 中的 `Pawn.Ideo` getter 有两处；原转译器仅替换 operand，安装时抛 `MissingMethodException`。按精确 MethodInfo 匹配并同步改 opcode 后安装通过。
- `JobDriver_BottleFeedBaby.<FeedBabyFoodFromInventory>b__15_2()` 是声明在 driver 上的无参结束回调。`Prefix(Toil __instance)` 会被 Harmony 3 拒绝；正确实例类型是 `JobDriver_BottleFeedBaby`，`initialFoodPercentage` 可由继承字段注入。编译生成名称仅适用于此 build，后续升级应重新核实 `FeedBabyFoodFromInventory` 方法体及回调声明类型。

验证入口：`Mods/animal-hardworking/Source/Validation/`。探针通过不代表地图行为或存读档已验证。


## 1.7.4981：Mimikko 蹭蹭与娱乐计时

2026-10-17 定向核实：`RimWorld.JobDriver_Nuzzle.MakeNewToils()` 最后的 `Toils_General.Do` 回调访问 driver 的 `pawn`；不能把该编译器生成回调的 `__instance` 声明为 Toil。角色专属蹭蹭可在公开 `Pawn_InteractionsTracker.TryInteractWith(Pawn recipient, InteractionDef intDef)` 的 Prefix 中，经 `Pawn ___pawn` 限定发起者，再替换 `ref InteractionDef intDef`；只匹配原版 `InteractionDefOf.Nuzzle`，不拦截其他互动。

`JoyUtility.JoyTickCheckEnd(Pawn pawn, int delta, JoyTickFullJoyAction fullJoyAction, float extraJoyGainFactor, Building joySource)` 的 Postfix 可按参数名取得 delta；定期回调用 `pawn.IsHashIntervalTick(interval, delta)`，避免只检测批次末端 tick。Mimikko 的 13 个核心补丁经独立 Harmony 3 PatchAll 实际安装，且游戏启动完成初始化；地图内互动效果尚未验收。证据入口见 sources.md 的 Mimikko 条目。
