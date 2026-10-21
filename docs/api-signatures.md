# 1.7 常用 API 签名快照

基准：`1.7.4979 rev109`，根目录 `Assembly-CSharp.dll`，MVID `425a593954d7432a8f44e3076b25fce9`。这些签名直接提取自本地程序集元数据，并按开发任务筛选。职责说明见 [csharp-api.md](csharp-api.md)。

**读取规则**：这是签名速查，不是可编译 C# 文件；未列可选参数的默认值、泛型约束和全部重载。`T` / `K` / `V` 是声明类型/方法的泛型参数。`Int32= int`、`Single=float`、`Boolean=bool`、`String=string`、`Void=void`。声明类型见各标题；短类型名按对应命名空间及导入解析。`public/protected` 与 static/virtual/abstract 标记保留，未列出的 API 不表示不存在。

按标题或符号检索即可，不需要把本页整体塞入上下文：

```powershell
rg -n -A 22 '^## Verse.ThingComp|^## Verse.AI.JobDriver' docs/api-signatures.md
```

## Verse.Mod

```text
public Void .ctor(ModContentPack content)
public ModContentPack Content
public virtual Void DoSettingsWindowContents(Rect inRect)
public T GetSettings<T>()
public virtual String SettingsCategory()
public virtual Void WriteSettings()
```

## Verse.ModSettings

```text
public virtual Void ExposeData()
public Void Write()
```

## Verse.Def

```text
public IEnumerable<String> ConfigErrors()
public T GetModExtension<T>()
public Void ResolveReferences()
public String defName
public String description
public String label
public ModContentPack modContentPack
```

## Verse.DefDatabase<T>

```text
public static IEnumerable<T> AllDefs
public static List<T> AllDefsListForReading
public static T GetNamed(String defName, Boolean errorOnFail)
public static T GetNamedSilentFail(String defName)
```

## Verse.DefModExtension

```text
public virtual IEnumerable<String> ConfigErrors()
```

## Verse.CompProperties

```text
public Void .ctor()
public Void .ctor(Type compClass)
public virtual IEnumerable<String> ConfigErrors(ThingDef parentDef)
public Type compClass
```

## Verse.ThingComp

```text
public virtual Boolean AllowStackWith(Thing other)
public virtual IEnumerable<Gizmo> CompGetGizmosExtra()
public virtual String CompInspectStringExtra()
public virtual Void CompTick()
public virtual Void CompTickInterval(Int32 delta)
public virtual Void CompTickLong()
public virtual Void CompTickRare()
public virtual Void Initialize(CompProperties props)
public virtual Void PostDeSpawn(Map map, DestroyMode mode)
public virtual Void PostDestroy(DestroyMode mode, Map previousMap)
public virtual Void PostExposeData()
public virtual Void PostPostMake()
public virtual Void PostSpawnSetup(Boolean respawningAfterLoad)
public virtual Void PostSplitOff(Thing piece)
public virtual Void ReceiveCompSignal(String signal)
public ThingWithComps parent
public CompProperties props
```

## Verse.Thing

```text
public Void DeSpawn(DestroyMode mode)
public virtual Void Destroy(DestroyMode mode)
public Boolean Destroyed
public virtual Void ExposeData()
public virtual String GetInspectString()
public Map Map
public Map MapHeld
public IntVec3 Position
public IntVec3 PositionHeld
public virtual Void SetFaction(Faction newFaction, Pawn recruiter)
public Void SpawnSetup(Map map, Boolean respawningAfterLoad)
public Boolean Spawned
public virtual Thing SplitOff(Int32 count)
public ThingDef def
public Int32 stackCount
```

## Verse.ThingWithComps

```text
public List<ThingComp> AllComps
public T GetComp<T>()
public String GetInspectString()
public Void InitializeComps()
public Void TickLong()
public Void TickRare()
protected Void Tick()
protected Void TickInterval(Int32 delta)
```

## Verse.ThingMaker

```text
public static Thing MakeThing(ThingDef def, ThingDef stuff)
```

## Verse.GenSpawn

```text
public static Thing Spawn(ThingDef def, IntVec3 loc, Map map, WipeMode wipeMode)
public static Thing Spawn(ThingDef def, IntVec3 loc, Map map, Rot4 rot, WipeMode wipeMode)
public static Thing Spawn(Thing newThing, IntVec3 loc, Map map, WipeMode wipeMode)
public static Thing Spawn(Thing newThing, IntVec3 loc, Map map, Rot4 rot, WipeMode wipeMode, Boolean respawningAfterLoad, Boolean forbidLeavings)
public static Boolean TrySpawn(ThingDef def, IntVec3 loc, Map map, out Thing thing, WipeMode wipeMode, Boolean canWipeEdifices)
public static Boolean TrySpawn(ThingDef def, IntVec3 loc, Map map, Rot4 rot, out Thing thing, WipeMode wipeMode, Boolean canWipeEdifices)
```

## Verse.GenPlace

```text
public static Boolean TryPlaceThing(Thing thing, IntVec3 center, Map map, ThingPlaceMode mode, Action<Thing, Int32> placedAction, Predicate<IntVec3> extraValidator, Rot4? rot, Int32 squareRadius)
public static Boolean TryPlaceThing(Thing thing, IntVec3 center, Map map, ThingPlaceMode mode, out Thing lastResultingThing, Action<Thing, Int32> placedAction, Predicate<IntVec3> extraValidator, Rot4? rot, Int32 squareRadius)
```

## Verse.Map

```text
public Void ExposeData()
public T GetComponent<T>()
public MapComponent GetComponent(Type type)
public EdificeGrid edificeGrid
public ListerThings listerThings
public MapPawns mapPawns
public ReservationManager reservationManager
public TerrainGrid terrainGrid
public ThingGrid thingGrid
```

## Verse.Game

```text
public Void ExposeData()
public T GetComponent<T>()
public GameComponent GetComponent(Type type)
```

## RimWorld.Planet.World

```text
public Void ExposeData()
public T GetComponent<T>()
public WorldComponent GetComponent(Type type)
```

## Verse.MapComponent

```text
public Void .ctor(Map map)
public virtual Void ExposeData()
public virtual Void FinalizeInit()
public virtual Void MapComponentDraw()
public virtual Void MapComponentOnGUI()
public virtual Void MapComponentTick()
public virtual Void MapComponentUpdate()
public virtual Void MapGenerated()
public virtual Void MapRemoved()
public Map map
```

## Verse.GameComponent

```text
public virtual Void ExposeData()
public virtual Void FinalizeInit()
public virtual Void GameComponentOnGUI()
public virtual Void GameComponentTick()
public virtual Void GameComponentUpdate()
public virtual Void LoadedGame()
public virtual Void StartedNewGame()
protected Void .ctor()
```

## RimWorld.Planet.WorldComponent

```text
public Void .ctor(World world)
public virtual Void ExposeData()
public virtual Void FinalizeInit(Boolean fromLoad)
public virtual Void WorldComponentOnGUI()
public virtual Void WorldComponentTick()
public virtual Void WorldComponentUpdate()
public World world
```

## Verse.GenTicks

```text
public static Boolean IsTickIntervalDelta(Int32 offset, Int32 period, Int32 delta)
public static Boolean IsTickIntervalDelta(Int32 period, Int32 delta)
public static Int32 MaxTickInterval
public static Int32 SecondsToTicks(Single numSeconds)
public static Int32 TickLongInterval
public static Int32 TickRareInterval
public static Int32 TicksAbs
public static Int32 TicksGame
public static Int32 TicksPerRealSecond
public static Single TicksToSeconds(Int32 numTicks)
```

## Verse.Gen

```text
public static Boolean IsHashIntervalTick(Map m, Int32 interval)
public static Boolean IsHashIntervalTick(Map m, Int32 interval, Int32 offset)
public static Boolean IsHashIntervalTick(Thing t, Int32 interval)
public static Boolean IsHashIntervalTick(Thing t, Int32 interval, Int32 delta)
public static Boolean IsHashIntervalTick(WorldObject o, Int32 interval)
public static Boolean IsHashIntervalTick(WorldObject o, Int32 interval, Int32 delta)
public static Boolean IsHashIntervalTick(Faction f, Int32 interval)
```

## Verse.JobMaker

```text
public static Job MakeJob()
public static Job MakeJob(JobDef def)
public static Job MakeJob(JobDef def, LocalTargetInfo targetA)
public static Job MakeJob(JobDef def, LocalTargetInfo targetA, LocalTargetInfo targetB)
public static Job MakeJob(JobDef def, LocalTargetInfo targetA, LocalTargetInfo targetB, LocalTargetInfo targetC)
public static Job MakeJob(JobDef def, LocalTargetInfo targetA, Int32 expiryInterval, Boolean checkOverrideOnExpiry)
public static Job MakeJob(JobDef def, Int32 expiryInterval, Boolean checkOverrideOnExpiry)
```

## Verse.AI.Job

```text
public Int32 count
public JobDef def
public Boolean playerForced
public LocalTargetInfo targetA
public LocalTargetInfo targetB
public LocalTargetInfo targetC
```

## Verse.AI.JobDriver

```text
public Void AddFailCondition(Func<Boolean> newFailCondition)
public Void AddFinishAction(Action<JobCondition> newAct)
public Void EndJobWith(JobCondition condition)
public virtual Void ExposeData()
public abstract Boolean TryMakePreToilReservations(Boolean errorOnFailed)
public Job job
public Pawn pawn
protected abstract IEnumerable<Toil> MakeNewToils()
```

## Verse.AI.Toil

```text
public Void AddFailCondition(Func<Boolean> newFailCondition)
public Void AddFinishAction(Action newAct)
public ToilCompleteMode defaultCompleteMode
public Int32 defaultDuration
public Action initAction
public Action tickAction
public Action<Int32> tickIntervalAction
```

## Verse.AI.Toils_Goto

```text
public static Toil GotoCell(TargetIndex ind, PathEndMode peMode)
public static Toil GotoCell(IntVec3 cell, PathEndMode peMode)
public static Toil GotoThing(TargetIndex ind, PathEndMode peMode, Boolean canGotoSpawnedParent)
public static Toil GotoThing(TargetIndex ind, IntVec3 exactCell)
```

## Verse.AI.Toils_General

```text
public static Toil Do(Action action)
public static Toil Wait(Int32 ticks, TargetIndex face)
```

## RimWorld.WorkGiver_Scanner

```text
public virtual Boolean HasJobOnCell(Pawn pawn, IntVec3 c, Boolean forced)
public virtual Boolean HasJobOnThing(Pawn pawn, Thing t, Boolean forced)
public virtual Job JobOnCell(Pawn pawn, IntVec3 cell, Boolean forced)
public virtual Job JobOnThing(Pawn pawn, Thing t, Boolean forced)
public virtual PathEndMode PathEndMode
public virtual IEnumerable<IntVec3> PotentialWorkCellsGlobal(Pawn pawn)
public virtual ThingRequest PotentialWorkThingRequest
public virtual IEnumerable<Thing> PotentialWorkThingsGlobal(Pawn pawn)
```

## Verse.HediffComp

```text
public virtual Void CompExposeData()
public virtual Void CompPostMake()
public virtual Void CompPostPostAdd(DamageInfo? dinfo)
public virtual Void CompPostPostRemoved()
public virtual Void CompPostTick(ref Single severityAdjustment)
public virtual Void CompPostTickInterval(ref Single severityAdjustment, Int32 delta)
public HediffWithComps parent
public HediffCompProperties props
```

## Verse.HediffMaker

```text
public static Hediff MakeHediff(HediffDef def, Pawn pawn, BodyPartRecord partRecord)
```

## Verse.Pawn_HealthTracker

```text
public Hediff AddHediff(HediffDef def, BodyPartRecord part, DamageInfo? dinfo, DamageResult result)
public Void AddHediff(Hediff hediff, BodyPartRecord part, DamageInfo? dinfo, DamageResult result)
public Void RemoveHediff(Hediff hediff)
```

## RimWorld.ThoughtWorker

```text
public ThoughtState CurrentState(Pawn p)
protected virtual ThoughtState CurrentSocialStateInternal(Pawn p, Pawn otherPawn)
protected virtual ThoughtState CurrentStateInternal(Pawn p)
```

## RimWorld.StatWorker

```text
public virtual Void FinalizeValue(StatRequest req, ref Single val, Boolean applyPostProcess)
public Single GetValueAbstract(BuildableDef def, ThingDef stuffDef)
public Single GetValueAbstract(AbilityDef def, Pawn forPawn)
public virtual Single GetValueUnfinalized(StatRequest req, Boolean applyPostProcess)
```

## RimWorld.IncidentWorker

```text
public Boolean CanFireNow(IncidentParms parms)
public Boolean TryExecute(IncidentParms parms)
protected virtual Boolean CanFireNowSub(IncidentParms parms)
protected virtual Boolean TryExecuteWorker(IncidentParms parms)
```

## Verse.RecipeWorker

```text
public virtual Void ApplyOnPawn(Pawn pawn, BodyPartRecord part, Pawn billDoer, List<Thing> ingredients, Bill bill)
public virtual Boolean AvailableOnNow(Thing thing, BodyPartRecord part)
public virtual Void ConsumeIngredient(Thing ingredient, RecipeDef recipe, Map map)
public virtual Void Notify_IterationCompleted(Pawn billDoer, List<Thing> ingredients)
```

## RimWorld.DefOfHelper

```text
public static Void EnsureInitializedInCtor(Type defOf)
```

## Verse.Scribe_Values

```text
public static Void Look<T>(ref T value, String label, T defaultValue, Boolean forceSave)
```

## Verse.Scribe_Defs

```text
public static Void Look<T>(ref T value, String label)
```

## Verse.Scribe_References

```text
public static Void Look<T>(ref T refee, String label, Boolean saveDestroyedThings)
public static Void Look<T>(ref WeakReference<T> refee, String label, Boolean saveDestroyedThings)
```

## Verse.Scribe_Deep

```text
public static Void Look<T>(ref T target, String label, params Object[] ctorArgs)
public static Void Look<T>(ref T target, Boolean saveDestroyedThings, String label, params Object[] ctorArgs)
```

## Verse.Scribe_Collections

```text
public static Void Look<T>(ref List<T> list, String label, Boolean saveDestroyedThings, LookMode lookMode, params Object[] ctorArgs)
public static Void Look<T>(ref List<T> list, String label, LookMode lookMode, params Object[] ctorArgs)
public static Void Look<K, V>(ref Dictionary<K, V> dict, String label, LookMode keyLookMode, LookMode valueLookMode)
public static Void Look<K, V>(ref Dictionary<K, V> dict, String label, LookMode keyLookMode, LookMode valueLookMode, ref List<K> keysWorkingList, ref List<V> valuesWorkingList, Boolean logNullErrors, Boolean saveDestroyedKeys, Boolean saveDestroyedValues)
public static Void Look<T>(ref HashSet<T> valueHashSet, String label, LookMode lookMode)
public static Void Look<T>(ref HashSet<T> valueHashSet, Boolean saveDestroyedThings, String label, LookMode lookMode)
public static Void Look<T>(ref Stack<T> valueStack, String label, LookMode lookMode)
public static Void Look<T>(ref Queue<T> valueQueue, String label, LookMode lookMode)
```

## Verse.Window

```text
public abstract Void DoWindowContents(Rect inRect)
public virtual Vector2 InitialSize
public virtual Void PostClose()
public virtual Void PostOpen()
public virtual Void PreClose()
public virtual Void PreOpen()
```

## Verse.Listing_Standard

```text
public Void Begin(Rect rect)
public Void CheckboxLabeled(String label, ref Boolean checkOn, Single tabIn)
public Void CheckboxLabeled(String label, ref Boolean checkOn, String tooltip, Single height, Single labelPct)
public Void End()
public Rect Label(TaggedString label, Single maxHeight, String tooltip)
public Rect Label(String label, Single maxHeight, TipSignal? tipSignal)
```

## Verse.Command_Action

```text
public Void ProcessInput(Event ev)
public Action action
```

## Verse.GenCommandLine

```text
public static Boolean CommandLineArgPassed(String key)
public static Boolean TryGetCommandLineArg(String key, out String value)
```

## 维护

本页只收录本次定向核实的常用成员；由反编译器元数据手动筛选生成。游戏更新、类型转发或重载变化后，只刷新受影响的条目，并更新 [sources.md](sources.md)。不要直接缓存 memberId：它由 MVID 和 metadata token 构成，会随程序集变化。

