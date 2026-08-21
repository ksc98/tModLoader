# Porting notes: Terraria 1.4.5.6 -> 1.4.5.7

Working notes for retargeting the `1.4.5` branch to vanilla 1.4.5.7. Updated as the port progresses.

## Version bump (done)

- `setup/Core/TerrariaDecompileExecutableProvider.cs`: `ClientVersion`/`ServerVersion` -> 1.4.5.7. The Windows `Terraria_v1.4.5.7_win.exe` / `TerrariaServer_v1.4.5.7_win.exe` were placed in the Terraria Steam dir by hand (the `.enc` secret asset is still 1.4.5.6 and a 1.4.5.7 install cannot derive its key); the server exe is also downloadable from `terraria.org/api/download/pc-dedicated-server/terraria-server-1457.zip`.
- `patches/*/Terraria/Terraria.csproj.patch`: `<Version>` context -> 1.4.5.7.
- `InstallVerifier.cs`: `TerrariaVersion` -> 1.4.5.7; Steam `Terraria.exe` MD5s updated for Windows (`25aab895d26cd4690e3130faf426c85f`) and Linux (`23377672abcd844bfb5374ac4224e01a`). **TODO**: macOS Steam hash, and GOG hashes for all three platforms (need those builds).
- **TODO for a team member**: re-encrypt `setup/SecretAssets/Terraria_v1.4.5.7_win.exe.enc` and add 1.4.5.7 proof-of-ownership keys to `keys.json`.

## Upstream status (checked 2026-08-21)

- Terraria 1.4.5.7 shipped 2026-08-19. No branch, PR, issue, or commit in tModLoader/tModLoader references 1.4.5.7; `1.4.5` HEAD is 241434b944 (2026-08-20). Chicken-Bones on #5208 (2026-07-23): "This will need at least minor updates with 1.4.5.7". No timeline in pinned #5070.
- Fork `H-A-M-G-E-R/tModLoader` branch `1.4.5.7` (commit 9f6a20c0d1, 2026-08-21) covers only `patches/Terraria` + the version consts; its patch content matches this branch's layer 1 exactly. No PR opened.
- Previous bump (1.4.5.5 -> 1.4.5.6) was three commits: PR #5028 (steviegt6: csproj/Terraria/NetCore patches + version consts), 08f5d4c9fa (Chicken-Bones: new `.enc` + `keys.json`, maintainer-only), 95f853d96b (JavidPack: InstallVerifier hashes). tModLoader-layer re-patching landed separately as "Batch N" commits. This branch follows the same split.
- Etiquette: PR against `1.4.5`; commit only patches produced by `diff`; tabs; "WIP:" prefix while seeking feedback; vanilla-source change requests go to the `collaborators` Discord channel.

## Setup tool

- `setup -m FUZZY` / `--patch-mode` was silently ignored: `SetupCommand` never copied `settings.PatchMode` into `ProgramSettings` (only `RegenSourceCommand` and `PatchXCommand` did). Fixed.
- Note that `regen-source` (and therefore `setup`) deliberately downgrades FUZZY to OFFSET (`RegenSourceTask.Run`). For a vanilla bump, run the per-layer commands: `patch terraria -m FUZZY`, `diff terraria`, `patch netcore -m FUZZY`, `diff netcore`, `patch tml -m FUZZY`, `diff tml`.

## Vanilla changes that broke patches

Layer `patches/Terraria` (2 hunks):
- `Audio/LegacyAudioSystem.cs`: constructor now delegates to `Init(string renderer = "")`, and `AudioEngine` is constructed with `(path, TimeSpan.FromMilliseconds(250), renderer)`. The FNA audio-driver log line moved into `Init`.
- `Netplay.cs`: `HasClients` renamed to `HasFullyConnectedClients`.

Layer `patches/TerrariaNetCore` (9 hunks, all plain drift): `FocusHelper.cs` (`wantsToPause` is now computed before the Windows `Form` block), `Net/Sockets/SocialSocket.cs` (`Invariant.Assert` lines added before the sends), `IO/WorldFile.cs` (version constant 319 -> 325), `Projectile.cs` (SetDefaults grew; last case gained `usesIDStaticNPCImmunity`/`idStaticNPCHitCooldown`), `Main.cs` (ctor moved), `Netplay.cs` (rename above).

Layer `patches/tModLoader`: 36 hunks failed outright and 562 applied fuzzy (128 below 80% quality) on first pass; see per-file review below.
- `Player.cs`: vanilla now has `using System.Linq;`.
- `ID/ItemID.cs`: new `AccessoryIncompatibilityType` set; `ItemsThatShouldNotBeInInventory` lost 6143.

## Decision: whip tag effects follow vanilla's new `TagDamageChanges` API

1.4.5.7 rewrote the tag-effect hit API: `UniqueTagEffect.ModifyTaggedHit/ModifyProcHit` now take `ref TagDamageChanges changes` (struct: `AddProjectileTagDamage`, `AddedBaseDamage`, `AddedFlatDamage`, `TotalDamageMultiplier`, `Crit?`, `HighestAddedBaseDamage`), `OnTaggedHit/OnProcHit` take `int calcDamage`, `OnProcHit` returns `bool` (true = consume the proc), and `Player.TagEffectState` became `Player.TagEffectStack` (up to 5 stacked effects, `Player.maxTagEffects`). Vanilla applies the accumulated `changes` once, in `Projectile.StatNPC`.

TML's 1.4.5.6 patches had replaced those signatures with `ref NPC.HitModifiers` / `NPC.HitInfo`. The two designs cannot coexist, so this branch **drops the TML signature overrides** on `UniqueTagEffect`, `WhipTagEffect`, `WhipTagEffect_Firecracker`, `WhipTagEffect_ViolentDisplayOfFlower` and `TagEffectState` (keeping TML's XML docs and the `ImmuneToWhipTags` rewrite of `WhipTagEffect.CanApplyTagToNPC`), and instead translates `TagDamageChanges` into `HitModifiers` at the single consumer in `Projectile.StatNPC` (`AddedBaseDamage`/`AddedFlatDamage` -> `FlatBonusDamage`, `TotalDamageMultiplier` -> `FinalDamage`, `Crit` -> `SetCrit()`/`DisableCrit()`). `OnHit` receives `strike.SourceDamage`. `ExampleWhipProjectileAdvanced` now calls `TagEffectStack.TryEnableProcOnNPC`.

Modder-facing consequence: `ModifyTaggedHit(..., ref NPC.HitModifiers)` etc. no longer exist on the 1.4.5 branch (never shipped). Maintainers may prefer to re-layer a HitModifiers-based API on top of the stack; that is a follow-up, not part of the bump.

## Decision: `NPC.StrikeNPC` / packet 28 on top of vanilla's new pending-damage design

1.4.5.7 rewrote `NPC.StrikeNPC`: it returns `int`, has no `noEffect`/`ignorePlayerInteractions`, expresses "no player interaction" as `owner == 255`, **sends packet 28 itself**, guards against nested strikes, and on multiplayer clients records a `PendingDamage` (`PushPendingDamage`/`AddStrikeDamage`) that the server acknowledges with the new packet 162; the NPC-sync handler (packet 23) subtracts still-pending damage via `GetPendingDamage` so client life doesn't snap back. Packet 28 now also carries the NPC `generation` byte, and the server calls `PlayerInteraction` in the handler.

TML keeps its split API (`StrikeNPC(HitInfo, fromNet, noPlayerInteraction, owner)` does not send; `NetMessage.SendStrikeNPC` does) because hundreds of TML/mod call sites depend on it. Grafted from vanilla: `noPlayerInteraction` maps to `owner = 255`; the pending-damage push/clear wraps `StrikeNPC_Inner`; the nested-strike guard and generation asserts are kept; `GetIncomingStrikeModifiers` absorbs the defense/crit/`takenDamageMultiplier` math as before. The legacy `int StrikeNPC(...)` wrapper now sends the packet itself (via `SendStrikeNPC`), because unpatched vanilla callers no longer send after striking. Packet 28 TML encoding gains the generation byte after the index; the receiver mirrors vanilla (162 ack, generation check, `PlayerInteraction`, `owner: server ? whoAmI : 255`).

Contract reminder for mods (unchanged, now load-bearing on clients): a `StrikeNPC(hit)` on a multiplayer client must be followed by `NetMessage.SendStrikeNPC`, otherwise the pending-damage entry is never acknowledged.

### Smaller decisions (tModLoader layer)

- `Main.RefreshSettingsButtonStatus`: vanilla rewrote the push-to-side condition (`posY - 28 <= num3`, fires even with zero extra accessory slots at small UI heights). TML's old `if (false && ...)` override ("ModAccessorySlot rework handles this") is dropped; vanilla logic kept. Revisit if the settings button overlaps modded slots.
- `Main.GetLinesInfo`: vanilla removed `researchLine` (it colours Journey research lines itself). New vanilla tooltip lines got TML `TooltipLine.Name`s: `LoadoutSharedFrom`, `LoadoutShared`, `LoadoutShareHint`, `Buffs`, `Healing`, `ManaHealing`, `Mount`.
- `PlayerLoader.ModifyZoom` now hooks `Main.GetPlayerControlledCameraPan` right after `Player.GetPanControlValues(out panFactor, ...)` (vanilla moved the zoom math into Player).
- `Main.DoUpdate` was split into `DoUpdateInWorld_Inner` + `UpdateWorld_*` sub-methods under `SwapRandom` scopes; all ten `SystemLoader.Pre/PostUpdate*` hooks now live in `DoUpdateInWorld_Inner` between the corresponding sub-calls, same order as before.
- `Chest.AddItemToShop`: vanilla now merges a sold stack into existing `buyOnce` slots before using an empty slot. TML's `int` return (for `PlayerLoader.PostSellItem`) is the empty slot if one was filled, else the last slot that absorbed stack.
- `TileDrawing.IsTileDangerous`: vanilla gained its own `(Player, int x, int y, ...)` overload; TML keeps its `(int x, int y, Player, ...)` order (call sites + `TileDrawing.TML.cs` unchanged) with vanilla's new body.
- `TileDrawing.DrawTile_LiquidBehindTile`: vanilla removed the `type == 379 -> new Tile()` hack TML was working around; TML's sand/liquid workaround hunks dropped, vanilla logic used.
- `WorldGen` error-world swaps: vanilla added `ErrorWorldSwapTiles`; TML keeps its full-copy `SwapTileData` behaviour.
- `WorldItem.CheckLavaDeath`, `Tile(Tile copy)`, `LegacyLighting` dev-lighting block, `Main.GoldCrittersCollection` loop, `NPC` `noGrabDelay`/`SyncItem` catch workaround: vanilla now does what TML patched in (or removed the code) — those TML hunks are dropped as redundant.
- `NPC.UpdateNPC_BuffApplyDOTs`: `NPCLoader.UpdateLifeRegen(ref damage)` now sees vanilla's `-1` ("auto") default instead of the old computed value; the `lifeRegen <= -240` cap is gone.
- `NPC.BeHurtByOtherNPC`: vanilla's new `type 695/696 -> 5 damage` override is folded into the HitModifiers path without damage variation.
- Whip tag proc damage: `WhipTagEffect_Firecracker.ProcDamageMultiplier` is 2.75 in vanilla now (was 1.75).

## Decision: `Item.NewItem` follows vanilla's new signature

1.4.5.7 replaced `bool noGrabDelay` with `NewItemOwnership ownership` (`None` / `ReserveForLocalPlayer` / `GrabDelayForLocalPlayer` / `GrabDelayForAllPlayers`), added `Vector2? velocity` and `NewItemModifier modifier`, introduced `Item.RequestNewItem` for clients and `Item.DefaultAssignNewItemsToPlayer(plr)` scopes, made the `(X, Y, Width, Height)` overload `[Obsolete]` in favour of `(Vector2 center, ...)`, removed `WorldItem.noGrabDelay`/`ownTime`/`keepTime` (now `grabDelayTime`/`grabDelayPlayer`/`enemyGrabDelayTime`), and packet 21's `number2` is now the ownership enum. TML's convenience overloads in `Item.TML.cs` take the same `ownership, velocity, modifier` parameters in the old `noGrabDelay` position; the TML `NewItem(source, Vector2, int type, ...)` overload is removed (vanilla has it now, the pair would be ambiguous). TML's clone-from-instance mechanism is rebuilt as `NewItem_Inner(source, center, itemToClone, ...)` on vanilla's body, keeping the `NPCLoader.blockLoot`/`newItemDisabled` guard and `ItemLoader.OnSpawn`. Old `noGrabDelay: true` call sites map to `NewItemOwnership.None` (the default).

Also: `PlayerDeathReason.CustomReason` stays TML's `ref NetworkText`; vanilla's new `string CustomReason` property is suppressed and `PlayerDamageTracker` calls `.ToString()`. `ItemSlot`'s new per-context bool arrays (`canFavoriteAt`, `canLoadoutShareAt`, `canItemSwapAt`, `retainFavoriteOnMoveAt`) are indexed with `Math.Abs(context)` because modded contexts are negative. `ItemsSacrificedUnlocksTracker` uses vanilla's `ConcurrentDictionary`.

## Decision: hit-source attribution (`PlayerNPCHitSource`)

1.4.5.7 attributes player damage to NPCs via `NPCDamageTracker.SetSourceForNextHit(npc, PlayerNPCHitSource)` called right before each `StrikeNPC`, with `Player.ApplyDamageToNPC` gaining a `(string sourceIdentifier, int sourceItemId, int sourceProjectileId)` overload and a `PlayerNPCHitSource hitSource` overload. TML keeps its `ApplyDamageToNPC(npc, damage, knockback, direction, crit, DamageClass, damageVariation)` overload, adds a trailing `PlayerNPCHitSource hitSource = default`, threads it into `StrikeNPCDirect` (which calls `SetSourceForNextHit` before striking), and makes vanilla's two overloads delegate to it. The string overload loses its `= null` default so 5-argument calls aren't ambiguous. Vanilla's `SetSourceForNextHit` lines in melee (`FromItem`) and projectile (`FromProjectile`) hit paths stay active.

Also: `Player.cs` set-bonus hunks (Pumpkin/CrystalNinja/Obsidian) were dead duplicates — the live ones are in `DataStructures/ArmorSetBonuses.cs.patch`, which applied cleanly. Vanilla removed the `UpdateEquips` wing loop (`hasWings` field now); `equippedWings` is set alongside it. Ammo selection moved into `PickAmmo_PickAmmoItem`/`PickAmmo_IterateRange`; `ItemLoader.CanChooseAmmo` is applied in the latter.

## Status (2026-08-21, end of first pass)

All three patch layers re-diffed against the 1.4.5.7 decompile. tModLoader layer: 36 failed hunks and all 128 fuzzy hunks under 80% quality hand-reviewed per file (plus many ≥80% hunks found wrong along the way and fixed). **Not yet compiled** — the next step is `dotnet build tModLoader.sln` (Debug) and working through compile errors; expect some in files that were not in the review set (e.g. `NetMessage.cs` still references the removed `ProjectileID.Sets.NeedsUUID`). Multiplayer (pending-damage/packet 28, packet 21 ownership) is untested. `MigrationGuide_1.4.5.md` entries and tModPorter rewrites for the modder-facing changes (tag effects, `NewItem`, `ApplyDamageToNPC`) are still to do.

## Build status (2026-08-21, `dotnet build tModLoader.sln -c Debug`)

Declaration-level errors fixed (vanilla now has `Item()`/`NPC()` ctors, `Netplay.ServerIP`, `DynamicSpriteFont.SpriteCharacterData` is a class in an array, `IMapLayer.Visible` on the new `DigtoiseMapLayer`, `Terraria.Testing.Cloning.CloneByReference` collides with TML's attribute, duplicated `CreateImpactExplosion`/`ToggleCore0Pinning` from fuzzy hunks, `DustID.FrostHydra` upstreamed, mangled `Item.SetDefaults_Inner` tail). Remaining: **238 distinct errors** in 28 files, mostly renamed vanilla locals referenced by fuzzy-applied hunks.

By file:

- Terraria/Main.cs: 76
- Terraria/NPC.cs: 31
- Terraria/WorldGen.cs: 21
- Terraria/Initializers/UILinksInitializer.cs: 19
- Terraria/Projectile.cs: 19
- Terraria/Player.cs: 18
- Terraria/GameContent/Drawing/ParticleOrchestrator.cs: 10
- Terraria/WorldItem.cs: 8
- Terraria/GameContent/ItemDropRules/CommonCode.cs: 6
- Terraria/Social/Steam/WorkshopSocialModule.TML.cs: 5
- Terraria/GameContent/UI/NPCChatPanel.cs: 3
- Terraria/Testing/StateSnapshot.cs: 3
- Terraria/GameContent/UI/EmoteBubble.cs: 2
- Terraria/Item.cs: 2
- Terraria/ModLoader/ProjectileLoader.cs: 2
- Terraria/Audio/ActiveSound.cs: 1
- Terraria/Audio/SoundPlayer.cs: 1
- Terraria/GameContent/Creative/ItemFilters.cs: 1
- Terraria/GameContent/Drawing/TileDrawing.cs: 1
- Terraria/GameContent/Items/ItemVariants.cs: 1
- Terraria/GameContent/Items/WhipTagEffect_ElectricEel.cs: 1
- Terraria/ID/CloudID.cs: 1
- Terraria/Main.TML.cs: 1
- Terraria/Map/MapHelper.cs: 1
- Terraria/ModLoader/TileLoader.cs: 1
- Terraria/Testing/ChatCommands/ToolkitDebugCommands.cs: 1
- Terraria/UI/Chat/ChatManager.TML.cs: 1
- Terraria/Wiring.cs: 1

By code: CS0103 ×147, CS1061 ×26, CS1503 ×19, CS0117 ×13, CS0029 ×8, CS0030 ×6, CS0246 ×3, CS0136 ×3, CS7036 ×2, CS0128 ×2, CS0411 ×2, CS0619 ×1, CS0206 ×1, CS0126 ×1, CS0022 ×1, CS1501 ×1, CS1739 ×1, CS0266 ×1

Full list: run the build; the per-line list is not committed.

## Per-file review of the tModLoader layer

(filled in as files are reviewed)

## Linux frame limiter (fixed)

`Main.EndDraw` busy-waits until `UpdateTimeAccumulator` reaches one frame, accumulating with
`Utils.SWTicksToTimeSpan(delta).TotalSeconds`. That helper builds a `TimeSpan`, whose resolution is
100ns. On Windows `Stopwatch.Frequency` is 10MHz so one stopwatch tick is exactly one TimeSpan tick and
nothing is lost. On Linux the frequency is 1GHz, and each spin iteration takes ~66ns, so every iteration
truncated to **zero** accumulated time. The loop needed ~30 million iterations (~2 seconds) to register
one 16.6ms frame: the game ran at ~1 fps with a fully pegged core, on the menu and in game alike.

Vanilla 1.4.5.7 has no Linux build, so upstream has not hit this. The fix accumulates the delta as
seconds directly (`(double)(presentTimestamp - num) / Stopwatch.Frequency`) instead of routing it
through `TimeSpan`.

Symptoms to recognise if it regresses: `update=0ms draw=0ms present=0ms` per frame with all wall clock
time inside `EndDraw`, and stack samples parked in `Main.EndDraw` → `Thread.PollGC`.
