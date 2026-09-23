# BattletechPerformanceFix — IrianTech build (TKT-102)

Built 2026-09-22 to get **one** fix: `MDDB_TagsetQueryInChunks`, which stops lance
generation throwing `SQLite error: too many SQL variables`.

## Why a custom build

The fix was **never released**. Neither the installed build nor the official 2.13
release (`bfix for 1.9`) contains any MDDB feature at all — the README documents
`MDDB_TagsetQueryInChunks` but master's source does not have it. It exists only on the
unreleased branch `feature/TagsetQueryInChunks` (last touched 2019/2020).

## What the fix does

`TagSetQueryExtensions.GetMatchingDataByTagSet` binds the `TagSetID` parameter as a
`string[]`, and Dapper expands an array parameter into **one bound variable per
element**. So the bound-variable count equals the number of matching TagSet rows,
which scales with roster size. SQLite caps a prepared statement at **999** variables.
Vanilla's small roster stays under it; IrianTech's 1689 mechdefs plus vehicles do not.
The transpiler replaces the `Query` call with `InterceptQuery`, which splits `TagSetID`
into groups of 100 and concatenates the results.

Nothing is malformed — this is a stock-engine limit hit by roster scale, which is why
ModValidator sees nothing and why the failing contracts' own tag sets are empty.

## Changes made to the branch to build against BattleTech 1.9.1

The branch targets an older game/toolchain. All edits below are **removals** — no
behaviour was altered in the feature we want.

1. `TargetFrameworkVersion` v3.5 -> **v4.7.2** (only 4.7.2/4.8 targeting packs exist on
   this machine, and v4.7.2 is what the working PanicSystem fork uses).
2. Dropped the `app.config` `<None>` entry (file absent in the branch).
3. Dropped both `System.Data.SQLite` references and the NuGet import-guard `<Error>`.
   Only `MDDB_InMemoryCache.cs` used them, and we do not want that feature.
4. Fixed the `Assembly-CSharp-firstpass` HintPath, which was a relative path
   (`..\..\BattleTech_Data\...`) that only resolves inside a Mods/ checkout. Now uses
   `$(BTechData)`. This was why `SVGAsset` appeared "missing".
5. Added `UnityEngine.CoreModule` and `UnityEngine.AssetBundleModule` references — this
   Unity version is modular, the branch expected a monolithic `UnityEngine.dll`.
6. Removed the PostBuildEvent copy steps (they target a Mods/ checkout layout).
7. **Removed six features that do not compile against 1.9.1** (API drift), plus their
   `Compile` entries and their rows in `Main.cs`'s feature table:
   - `MDDB_InMemoryCache` (System.Data.SQLite)
   - `DMFix`, `CollectSingletons` (`DataManagerLoadRequest`,
     `DataManagerRequestCompleteMessage`, `CheckDependenciesAfterLoad` all changed)
   - `BTLightControllerThrottle` (`GetLightArray` signature changed)
   - `PatchMechlabLimitItems` (`InventoryFilter` ctor changed) — this also defined
     `MechlabFix`, so that row went too
   - `LazyLoadAssets` (depended on `CollectSingletons`)
   Dropping `MechlabFix` is a bonus: it is the BPF feature most likely to fight
   CustomComponents / CustomUnits, and it is now absent from the assembly entirely.
   `SimpleMetrics` was KEPT — it only mentions `CheckDependenciesAfterLoad` as a string
   in a name list, so it compiles, and `Extensions.cs` depends on it.

## Build

    dotnet msbuild source/BattletechPerformanceFix.csproj -p:Configuration=Release \
      -p:BTechData="C:\Program Files (x86)\Steam\steamapps\common\BATTLETECH\BattleTech_Data\Managed\"

Output lands at the repo root as `BattletechPerformanceFix.dll`.

## Deploy

Copy `BattletechPerformanceFix.dll` **and `libs/RSG.Promise.dll`** into
`Mods/BattletechPerformanceFix/`. RSG.Promise is a real runtime reference of the built
assembly and is NOT in the previously installed mod folder — omitting it will fail to
load. Settings.json must list every feature false except `MDDB_TagsetQueryInChunks`.

Backup of what was there before: `C:\BattleTech\_bpf_backup_2026-09-22\`.

## ⚠ `PatchAll` applies attribute patches REGARDLESS of the feature switches

`Main.cs` calls `harmony.PatchAll(Assembly.GetExecutingAssembly())`, so **any class carrying
a `[HarmonyPatch]` attribute is applied even when every feature is switched off**. The
Settings.json switches only gate `Feature.Activate()`; they do not gate attribute patches.

This broke the MechLab on the first working launch. `PatchMechLabPanelLoadThrottling.cs`
carried `[HarmonyPatch(typeof(MechLabPanel), "RequestResources")]`, so it patched the panel
with all features off, and its static initializer — which resolves the `WaitForLoads`
state-machine type by string — threw on 1.9.1:

    TypeLoadException: Could not load type 'BattleTech.UI.MechLabPanel'
      BattletechPerformanceFix.Patch_MechLabPanel_RequestResources..cctor()
      -> MechLabPanel.RequestResources -> MechLabPanel.SetData -> OnGotoMechLab

Opening a mech in the MechLab threw every time. **The file was removed**; it was the only
`[HarmonyPatch]`-attributed class in the assembly (verified by grep), and the rebuilt DLL
contains no `MechLabPanel` reference at all (verified by string scan).

**If you ever re-add a file to this build, grep it for `[HarmonyPatch]` first.** A feature
being "off" is not protection.

## Where this lives

Fork: **https://github.com/jaredvjonas/BattletechPerformanceFix**, branch **`iriantech-fork`**,
branched from upstream `feature/TagsetQueryInChunks` so the IrianTech changes show as a
reviewable diff against the real upstream history rather than a source drop.

Working copy: `C:\BattleTech\Modders\BattletechPerformanceFix` (alongside the other forks).
Build artifacts and the game DLLs msbuild copies into the output are covered by `.gitignore`
and must never be committed.

Upstream is not tracked as a remote; add it with
`git remote add upstream https://github.com/m22spencer/BattletechPerformanceFix.git`
if the branch ever needs re-syncing.
