# Palworld Offline Cheat Catalog — Ranked Simple → Harder

**Target:** PPSA23126 / Palworld 01.102.999  
**Basis:** offline `dump.bin` + SDK headers only. No console probing, no external tools, no debug/cheat-menu features.  
**Purpose:** a feasibility-ranked build list for new Reaper patches. Every entry still needs an in-game test; the addresses and hook shapes are verified from the dump/SDK, not from gameplay.

---

## 0. Status snapshot

| Item | State |
|---|---|
| Player Speed (Character Only) | Updated with `0x7AEEAC8` player-actor gate; **awaiting retest** (player faster, wild/base pals normal, party-pal follow behavior separated). |
| Craft Anything Fix V3 | Generated and validated; patch index 12. **Awaiting gameplay confirmation.** Enable only one craft patch. |
| Craft guide | Delivered: 9 sections + 4 appendices, `uedumper` + `uelive` + Reaper Studio methodology. |
| Master Code / Infinite Stamina / Never Hungry / EXP x100 / Heat–Cold / High Crit / high capture | Proven working in prior tests. |
| New catalog entries below | Offline-verified hook sites; not yet built/tested unless marked “existing”. |

**Address convention in this document**

- `native` = offset inside `dump.bin` (file offset).
- `VA` = runtime virtual address = `0x400000 + native`.
- Data/vtable addresses are VAs, because `dump.bin` is partial and does not contain them.
- `0x7AEEAC8` = player **character actor** vtable (same gate as Master Code / Infinite Stamina).
- `0x7AEF6C8` = player **controller** vtable (used by the EXP patch).
- `[param+0x480]` = `SaveParameter.IsPlayer` on `UPalIndividualCharacterParameter`.
- `APalCharacter::CharacterParameterComponent` = `+0x630`; component `IndividualParameter` = `+0x178`.

**Useful SDK offsets**

| Object | Offset | Meaning |
|---|---:|---|
| `UPalIndividualCharacterParameter` | `0x318` | `IndividualActor` (`APalCharacter*`) |
| `UPalIndividualCharacterParameter` | `0x3D0` | start of embedded `FPalIndividualCharacterSaveParameter` |
| parameter | `0x3F1` | `Rank` |
| parameter | `0x3F4–0x3F7` | `Rank_HP / Attack / Defence / CraftSpeed` |
| parameter | `0x3F8` | `Exp` |
| parameter | `0x448` | `HP` |
| parameter | `0x450–0x453` | `Talent_HP / Melee / Shot / Defense` |
| parameter | `0x454` | `FullStomach` |
| parameter | `0x480` | `IsPlayer` |
| parameter | `0x490` | `OwnerPlayerUId` (16 bytes) |
| parameter | `0x4B0` | `MaxHP` |
| parameter | `0x4BC` | `CraftSpeed` |
| parameter | `0x4FC` | `SanityValue` |
| parameter | `0x53C` | `MaxFullStomach` |
| `APalCharacter` | `0x630` | `CharacterParameterComponent` |
| `UPalCharacterParameterComponent` | `0x178` | `IndividualParameter` |
| `UPalPlayerInventoryData` | `0x184 / 0x188` | `NowItemWeight / MaxInventoryWeight` |
| `UPalTechnologyData` | `0x150 / 0x154` | `TechnologyPoint / bossTechnologyPoint` |
| `UPalPartnerSkillParameterComponent` | `0x1D4 / 0x1D8` | `CoolDownTime / CoolDownTimeMax` |
| `UPalActiveSkillSlot` | `0x28` | `SelfActor` |

**GameSetting walk** (same skeleton as High Crit):

```
GWorld [0x87D5C90] -> +0x1B8 GameInstance -> +0x498 GameSetting
```

Group every GameSetting write into **one** ProcessEvent patch with toggles. Separate ProcessEvent patches at the same entry will conflict.

---

## 1. Ranking legend

| Tier | Meaning | Typical effort |
|---|---|---|
| **S** | Very easy: one site, pure replacement or tiny cave, no complex gate | 15–45 min |
| **A** | Easy: one cave + one player/pal gate, or one settings write | 45 min–2 h |
| **B** | Medium: multi-instruction cave, return-site edit, or side effects to test | 2–4 h |
| **C** | Hard/research: central formula, array/struct, server path, or unknown tick | 4 h+ |

Difficulty is implementation effort, **not** risk. Risk is called out separately.

---

## 2. Tier S — very easy

| # | Cheat | Hook / function (native → VA) | Scope | Approach | Risk / notes |
|---:|---|---|---|---|---|
| S1 | **No Fall Damage** | `IsNoFallDamageLastJumpedLocation` `0x28940F0` → `0x2C940F0` | All characters; only player matters | Replace entry with `mov al,1; ret` | Low. BP `BP_Status_FallDamage_C::Reset_Fall_Start_Pos` calls it. Test cliffs and high-jump falls. |
| S2 | **No Weapon Durability** | `APalWeaponBase::DecreaseDurability` `0x2C92C50` → `0x3092C50` | All weapons/tools | Replace entry with `ret` | Low. Test weapon and tool durability UI; skipping the delegate should leave the bar unchanged. If durability still drops, check `DecreaseDurabilityWithValue` (SDK `0x24628F0`) as a second path. |
| S3 | **Infinite Ammo** | `UPalDynamicWeaponItemDataBase::UseBullet` `0x27C2F30` → `0x2BC2F30` | All ranged weapons | Return `al=1`; or NOP `FF 4B 84` at `0x27C2F64` and `89 43 EC` at `0x27C2F77` | Medium/test. Global hook: enemies also benefit. If inventory bullets still drain, also hook `DecrementBullet` `0x24631B0` or set `IsInfinityMagazine` `+0x4F8`. |
| S4 | **Zero Weight** | `GetNowItemWeight` `0x2ABFF90` → `0x2EBFF90` | Player inventory | `xorps xmm0,xmm0; ret` | Low. UI shows 0 weight. If overweight logic reads the field directly, use A1 instead. Choose one of S4/A1. |
| S5 | **Hunger Lock (Player + Pals, replaces Never Hungry)** | Same site as Never Hungry: `0x28909A7` → `0x2C909A7` (`vmovss [rbx+0x454],xmm0`) | Player and pals/NPCs | Cave writes `MaxFullStomach [rbx+0x53C]` into `[rbx+0x454]`; player branch can use `100.0f`; return offset 8 | Low. Do **not** enable with Never Hungry: same AOB. A pal-only version can gate `[rbx+0x480]==0`. |
| S6 | **Infinite Technology Points** | `RequestUnlockRecipeTechnology` real `0x2BCFF10` → `0x2FCFF10` | Player | At entry set `[rdi+0x150]=9999` and `[rdi+0x154]=9999`, then run original prologue and jump to `0x2BCFF16` | Low/medium. If the unlock button stays grayed out, also force `IsUnlockableRecipeTechnology` real `0x2BD0720` to true or refresh the tech UI. |
| S7 | **Sell Price x10** | GameSetting `SellItemRate` `+0x708` | Global economy | One ProcessEvent settings write: `10.0f` | Low. Only selling is affected; buying is unchanged. If you build the A13 bundle, put Sell x10 there instead of a separate patch. |
| S8 | **Partner Skill No Cooldown** | `UPalPartnerSkillParameterComponent::IsCoolDown` real `0x2568CE0` → `0x2968CE0` | All partner-skill components | Replace entry with `xor eax,eax; ret` | Medium/test. The getter also reads `bIsRunning`/`bIsOverheat`; if the skill still blocks, hook the cooldown timer or set `CoolDownTimeMax` to 0. |

### S1–S5 cave shapes

**S1 — No Fall Damage**

```asm
; at 0x28940F0
mov al, 1
ret
```

**S2 — No Durability**

```asm
; at 0x2C92C50
ret
```

**S3 — Infinite Ammo (magazine path)**

```asm
; at 0x27C2F30
mov al, 1
ret
; fallback: NOP the decrements at 0x27C2F64 / 0x27C2F77
```

**S4 — Zero Weight**

```asm
; at 0x2ABFF90
xorps xmm0, xmm0
ret
```

**S5 — Hunger Lock**

```asm
; source: vmovss [rbx+0x454], xmm0
; at 0x28909A7, return to 0x28909AF
cmp byte ptr [rbx+0x480], 0
jne player
vmovss xmm0, dword ptr [rbx+0x53C]   ; MaxFullStomach (pal)
jmp store
player:
mov eax, 0x42C80000                  ; 100.0f fallback for player
vmovd xmm0, eax
store:
vmovss dword ptr [rbx+0x454], xmm0
jmp 0x28909AF
```

For **pal-only**, add:

```asm
cmp byte ptr [rbx+0x480], 0
jne original
; ... write MaxFullStomach ...
jmp return
original:
vmovss dword ptr [rbx+0x454], xmm0
jmp return
```

---

## 3. Tier A — easy

| # | Cheat | Hook / function (native → VA) | Scope | Approach | Risk / notes |
|---:|---|---|---|---|---|
| A1 | **Carry Weight x10** | `UPalPlayerInventoryData::GetMaxItemWeight` `0x2ABFFA0` → `0x2EBFFA0` | Player | Cave: `vmovss xmm0,[rdi+0x188]; mulss xmm0,10.0; ret` | Low. Test overweight state; 31 xrefs suggest the getter is used by gameplay. |
| A2 | **Pal HP Lock** | Extend **Master Code** SetHP source | Party/base pals | After `cmp rsi,[rdi+0x448]`, skip player; require `OwnerPlayerUId [rdi+0x490]` nonzero; set `rsi=[rdi+0x4B0]` (MaxHP) | Low/medium. Do not install a second patch at the SetHP AOB; add a toggle to Master Code. Offline only: other players’ pals also have owner UIDs. |
| A3 | **Pal Stamina Lock** | Infinite Stamina source VA `0x28D47E0` (native `0x24D47E0`) | Non-player pals (party/mount/base in practice) | Reuse the proven cave; invert the vtable gate: if player → original, if non-player → `ret` | Low/medium. Same AOB as Infinite Stamina; make it a toggle in that patch, not a second patch. The non-player gate also catches wild/enemy pals; add an owner-UID check for strict party/base scope. Test mounted sprint and pal skills. |
| A4 | **Pal EXP xN** | EXP source VA `0x2C8AFB7` (native `0x288AFB7`) | Non-player pals (party/base in practice) | Reuse EXP cave; replace controller-vtable gate with `cmp byte [r14+0x480],0; jne original`, then multiply gain | Low/medium. Same 7-byte AOB as EXP x100; combine as toggles. EXP is normally granted only to player-owned pals, so the non-player gate is usually safe. |
| A5 | **Pal Sanity Lock** | Sanity write `0x28911A3` → `0x2C911A3` (`vmovss [rbx+0x4FC],xmm1`) | Non-player pals (base/party in practice) | Non-player gate; write `100.0f` to `[rbx+0x4FC]` and `[rbp-0x20]`; jump to `0x28911B0` | Low/medium. Preserve `xmm3` and the delegate call; test base-worker sanity. Sanity matters mainly for base workers; wild pals are unaffected in practice. |
| A6 | **Pal Work Speed x3** | `GetCraftSpeed_withBuff` return `0x288F568` → `0x2C8F568` | Non-player pals (base/party in practice) | Cave: `vcvtss2si eax,xmm0; sar eax,1; cmp [rbx+0x480],0; jne skip; imul eax,eax,3; skip: add rsp,8; pop rbx; pop rbp; ret` | Medium/test. If no effect, hook the suitability variant `0x288F6E0` return. Does not speed the player. Only owned/base pals actually work. |
| A7 | **Instant Egg Hatch** | `UPalMapObjectHatchingEggModel::UpdateWorkAmountBySec` `0x2966A00` → `0x2D66A00` | Eggs | Multiply incoming `xmm0` by 10 at entry; run original prologue; jump to `0x2966A06` | Low/medium. Test one egg; large multipliers may skip animation/state. |
| A8 | **Perfect Capture Stats** | `UPalUtility::PalCaptureSuccess` real `0x2C6A050` → `0x306A050` | All captured pals | `rsi` = monster; `rax=[rsi+0x630]` → `[+0x178]`; write Rank + Rank_* (max 4, verify) and Talents 100; then original prologue | Medium. Test capture, save/load, and pal-box stats. If stats do not stick, hook the post-capture storage function instead. |
| A9 | **Player Attack x2** | `GetShotAttack/GetMeleeAttack` return `0x288CB8E` → `0x2C8CB8E` (`add eax,r12d`) | Player | Cave: `add eax,r12d; cmp [rbx+0x480],0; je skip; imul eax,eax,2; skip: mov rcx,[r15]; jmp 0x288CB94` | Medium. Affects shot and melee final attack. Test both; canary check is preserved. |
| A10 | **No Active Skill Cooldown** | `UPalActiveSkillSlot::IsCoolTimeFinish` real `0x25D2650` → `0x29D2650` | Player | Gate `[rdi+0x28]` SelfActor vtable `0x7AEEAC8`; player → `mov al,1; ret`; non-player → original two instructions then `jmp 0x25D2656` | Medium. Test skill spam/animations; may allow skills during states that expect cooldown. |
| A11 | **High Jump** | `GetJumpZVelocityMultiplier` `0x24B8910` → `0x28B8910` | Player | Gate owner `[rdi+0x168]` vtable; return `1.35–1.5f`; non-player original prologue | Medium. Jump height scales with velocity squared (1.5× ≈ 2.25× height). Pair with S1; test map bounds. |
| A12 | **Low Gravity** | `GetGravityZMultiplier` `0x24B8D50` → `0x28B8D50` | Player | Same owner gate; return `0.5f`; non-player original prologue | Medium. Test jumps, gliding, falling, and physics props. |
| A13 | **GameSettings Bundle** | GWorld walk, GameSetting fields | Player/global | One ProcessEvent patch with toggles: Sell x10 `+0x708`, Player Work Speed `+0x948`, Tech/Status per level `+0x15B4`/`+0x1D08`, Pal→Player damage `+0x348` | Low/medium. Do not create separate ProcessEvent hooks. Player work speed may not affect Instant Craft/Build (already instant). |

### A1–A8 cave shapes

**A1 — Carry Weight x10**

```asm
; at 0x2ABFFA0
vmovss xmm0, dword ptr [rdi+0x188]
mov eax, 0x41200000          ; 10.0f
vmovd xmm1, eax
vmulss xmm0, xmm0, xmm1
ret
```

Use `0x40800000` for 4.0×.

**A2 — Pal HP Lock (extend Master Code)**

```asm
cmp rsi, qword ptr [rdi+0x448]
jge original
cmp byte ptr [rdi+0x480], 0
jne original                 ; player: leave to Infinite HP
mov rax, qword ptr [rdi+0x490]
or  rax, qword ptr [rdi+0x498]
jz  original                 ; no owner UID: wild/enemy
mov rsi, qword ptr [rdi+0x4B0] ; MaxHP
original:
; Master Code's original prologue / return
```

**A3 — Pal Stamina Lock**

Reuse the proven Infinite Stamina cave. Change the final branch:

```asm
mov rax, qword ptr [rax]
mov rcx, 0x7AEEAC8
cmp rax, rcx
je  InfiniteStaminaOriginal  ; player -> original
ret                          ; non-player -> no SP loss
```

**A4 — Pal EXP xN**

Reuse the EXP cave, but gate on `IsPlayer`:

```asm
push rax                     ; keep the EXP cave's RAX preservation
cmp byte ptr [r14+0x480], 0
jne ExpOriginal              ; player -> original (handled by EXP x100)
mov rax, qword ptr [r14+0x3F8]
sub rcx, rax
imul rcx, rcx, N
add rcx, rax
mov qword ptr [r14+0x3F8], rcx
pop rax
jmp ExpReturn
```

**A5 — Pal Sanity Lock**

```asm
; at 0x28911A3, return to 0x28911B0
cmp byte ptr [rbx+0x480], 0
jne original
mov eax, 0x42C80000          ; 100.0f
mov dword ptr [rbx+0x4FC], eax
mov dword ptr [rbp-0x20], eax
jmp 0x28911B0
original:
vmovss dword ptr [rbx+0x4FC], xmm1
vmovss dword ptr [rbp-0x20], xmm1
jmp 0x28911B0
```

**A6 — Pal Work Speed x3**

```asm
; hook at 0x288F568; this replaces the original epilogue
vcvtss2si eax, xmm0
sar eax, 1
cmp byte ptr [rbx+0x480], 0
jne skip
imul eax, eax, 3
skip:
add rsp, 8
pop rbx
pop rbp
ret
```

`rbx` is the parameter (`mov rbx,rdi` at `0x288F548`).

**A7 — Instant Egg Hatch**

```asm
; at 0x2966A00
mov eax, 0x41200000          ; 10.0f
vmovd xmm1, eax
vmulss xmm0, xmm0, xmm1
push rbp
mov rbp, rsp
push r15
jmp 0x2966A06                ; original continues at push r14
```

**A8 — Perfect Capture Stats**

```asm
; at 0x2C6A050, rsi = captured APalCharacter*
test rsi, rsi
jz original
mov rax, qword ptr [rsi+0x630]
test rax, rax
jz original
mov rax, qword ptr [rax+0x178]
test rax, rax
jz original

mov byte ptr [rax+0x3F1], 4  ; Rank (verify max: 4 vs 5)
mov byte ptr [rax+0x3F4], 4  ; Rank_HP
mov byte ptr [rax+0x3F5], 4  ; Rank_Attack
mov byte ptr [rax+0x3F6], 4  ; Rank_Defence
mov byte ptr [rax+0x3F7], 4  ; Rank_CraftSpeed
mov byte ptr [rax+0x450], 100
mov byte ptr [rax+0x451], 100
mov byte ptr [rax+0x452], 100
mov byte ptr [rax+0x453], 100

original:
push rbp
mov rbp, rsp
push r15
jmp 0x2C6A056
```

Do not set Level or MaxHP here; those need a stat recalculation path.

---

## 4. Tier B — medium

| # | Cheat | Hook / function (native → VA) | Scope | Approach | Risk / notes |
|---:|---|---|---|---|---|
| B1 | **Player Defense x2** | `GetDefense_withBuff` return `0x288F444` → `0x2C8F444` | Player | Gate `[rbx+0x480]`; multiply `eax` by 2; reconstruct epilogue | Medium. Redundant with Master Code. Test both `GetDefense` and buffed defense; avoid double-scaling. |
| B2 | **No Shield Damage** | `DecreaseShieldHPByDamage` real `0x288FF10` → `0x2C8FF10` | Player | Gate `[rdi+0x480]`; return before applying damage; non-player original prologue | Low/medium. Redundant with Master Code; test shield UI/delegates. |
| B3 | **Item Gain x2** | `AddItem_ServerInternal` `0x2ABB8E0` → `0x2EBB8E0` | All item additions | At entry `test edx,edx; jle original; imul edx,edx,2`, then original prologue; `edx` = Count | High risk. Doubles pickups, rewards, crafting outputs, money, shop trades. Do **not** double negative counts (item removal). Test stack limits and quests; a pickup-only variant is safer. |

Buildable but riskier research items (free shopping, breeding, unlock-all, fast travel, food timers) are in Tier C.

---

## 5. Tier C — hard / research only

These are **not build-ready**. They are listed so the research queue is visible; the S/A/B entries are the feasible catalog. Several C entries still need a verified hook site before they can become patches.

| # | Cheat | Target | Why it is hard |
|---:|---|---|---|
| C1 | **Player Damage multiplier (central formula)** | `CalcDamageCharacter` real `0x2C58CC0` / `CalcDamage` real `0x2C589B0` | Complex struct args and multiple damage paths; the getter hook (A9) is much safer. |
| C2 | **Pal Damage multiplier** | `GetShotAttack/GetMeleeAttack` with non-player gate | Would buff enemies too. Party-only needs an owner-UID gate and careful testing. |
| C3 | **Capture rate 100% (native)** | `UPalUtility::JudgePalCapture_TryAllPhase` `0x243E0D0`; `PalCaptureSuccess` real `0x2C6A050` (thunk `0x243BE10`) | BP `CalcCaptureRate` is a thunk; existing high-capture patch already uses sphere settings. Force-capturable (see below) is experimental. |
| C4 | **Pal Work Suitability / rank editing** | Work-suitability arrays in save parameter | Array/struct edits; high save-corruption risk. |
| C5 | **Base camp limits / worker cap** | Base camp manager / GameSetting fields | Need the exact cap check and replication path. |
| C6 | **No build limits / infinite base range** | Build manager checks | Multiple UI/server checks; high regression risk. |
| C7 | **No recoil / no spread / rapid fire** | `APalWeaponBase` fields `+0x380`, `+0x384`, `+0x620`, `+0x4E4` | Needs a weapon-init or cooldown-tick hook; infinite ammo is easier. |
| C8 | **Free Shopping / Infinite Money** | Shop buy path `UPalUIItemShopBase::TryBuy` `0x23F64E0` / `RequestBuyProduct_ToServer` `0x229F380` | Needs server-path disassembly; Sell Price x10 (S7) is safer. |
| C9 | **Instant Breeding** | Breed farm fields `BreedRequiredRealTime +0x24C`, `BreedProgressTime +0x250`; `GetBreedProgress` real `0x291BFE0` | `GetBreedProgress` is a getter; returning 1.0 only changes UI. The progress tick is a virtual override that still needs to be located. |
| C10 | **Unlock All Recipes / Tech Tree** | `UPalTechnologyData` arrays, `UnlockedTechnologyNameArray` | Bulk save-array editing; high corruption risk. Infinite tech points (S6) is simpler and safer. |
| C11 | **Fast Travel Anywhere / Map Unlock** | Fast-travel save arrays / unlock flags | Save-structure editing; high corruption risk. |
| C12 | **Food Spoilage / Buff Duration** | Item container timers / food effect duration | Many item types and timer paths; test save/load. |

### Experimental capture entry (use with care)

**Force Capturable** — `IsForceCapturable` real `0x2894170` → `0x2C94170`: replace with `mov al,1; ret`. This flips the force-capturable flag, **not** a guaranteed 100% capture; the capture rate/minigame still applies. It may affect bosses/NPCs and is riskier than the existing high-capture patch. Do not enable both without testing.

---

## 6. What is already covered

| Existing patch | Covers | Catalog overlap |
|---|---|---|
| Master Code | Player Infinite HP; enemy OHK / 1 HP | A2 adds party/base pal HP as a toggle |
| Infinite Stamina | Player SP | A3 is the pal variant |
| Never Hungry | Player FullStomach | S5 replaces/extends it to player + pals |
| EXP x100 — Player Only | Player EXP | A4 is the pal variant |
| Heat and Cold Resistance 500 | Player temperature | No new entry needed |
| Player Speed (Character Only) | Player movement speed | No new entry needed |
| High Crit Bonus +60% | Capture crit | No new entry needed |
| high capture chance | Capture rate | Experimental force-capturable is separate |
| Build Anything / Craft Anything V3 | Building/crafting materials | Keep only one craft patch |
| Instant Building / Instant Craft | Work progress completion | A6/A13 are different speed paths |
| Resistance | Work-in-progress / not part of this catalog | No coverage |

---

## 7. Conflicts and “do not enable together”

1. **Never Hungry + S5 Hunger Lock** — same source AOB at `0x28909A7`. Replace Never Hungry with the combined lock, or enable only one.
2. **EXP x100 (player) + A4 Pal EXP** — same 7-byte AOB at VA `0x2C8AFB7`. Combine as two toggles.
3. **Infinite Stamina + A3 Pal Stamina** — same source AOB at VA `0x28D47E0`. Combine as two toggles.
4. **Master Code + A2 Pal HP Lock** — same SetHP source. Add a toggle to Master Code; do not install two patches at one AOB.
5. **Multiple GameSettings/ProcessEvent patches** — combine Sell Price, Work Speed, Tech/Status points, and Pal→Player damage into one settings patch.
6. **High capture chance + Force Capturable** — different functions, but test one at a time; both alter capture behavior.
7. **Craft Anything V3 + Craft Anything V1/V2** — enable only one craft patch. V3 is the recommended one.
8. **Instant Craft/Build + Pal Work Speed** — different hook sites, but test together; if the work tick is shared, combine toggles.
9. **Player Attack/Defense + Master Code** — not a conflict, but Master Code already removes most combat risk; test for double-scaling or UI weirdness.

---

## 8. Validation checklist for each new patch

- [ ] Back up the save; test offline only.
- [ ] Confirm the AOB finder reports **one** match before enabling.
- [ ] For a replacement patch, verify the return offset (source offset) and that the replacement is not longer than the AOB.
- [ ] Test the trigger in a safe area: fall, shoot, capture, hatch, work, level-up, etc.
- [ ] Test both player and pal behavior for shared functions.
- [ ] Save, reload, and re-test; check the pal box and base camp.
- [ ] Disable the patch and confirm the game returns to normal without a restart (where possible).
- [ ] Do not stack two patches that write the same field or hook the same AOB.

---

## 9. Recommended next five

1. **No Fall Damage (S1)** — 10 minutes, pure replacement, immediately useful with high jump/exploration.
2. **No Weapon Durability (S2)** — 15 minutes, pure replacement, low risk.
3. **Infinite Ammo (S3)** — 30 minutes, test both magazine and direct-bullet weapons.
4. **Carry Weight x10 (A1)** — 30 minutes, one cave, high quality-of-life.
5. **Infinite Technology Points (S6)** — 45 minutes, high progression value; verify UI unlock button.

Then, in order of value:

6. **Perfect Capture Stats (A8)** — highest-value pal collection cheat; test save/load.
7. **Pal HP Lock + Pal Stamina + Pal Hunger/Sanity** — makes base camps self-sufficient.
8. **Player Attack x2 (A9)** — simple final-return multiplier; test shot and melee.
9. **Pal Work Speed x3 (A6)** — base production; verify the work tick uses the craft-speed getter.
10. **No Active Skill Cooldown (A10)** — popular combat cheat; test animations/state.

---

## 10. Excluded by design

- `UPalCheatManager` and debug/cheat-menu functions: `CaptureSuccessAlways`, `CaptureFailAlways`, `TogglePartnerSkillNoDecrease`, spawners, admin commands, etc.
- Debug/cheat-menu items and console commands.
- Online/network exploits; this catalog is offline-only.
- Any feature requiring console probing, ps5debug-NG, or external runtime tools.
- Bulk save editing (unlock-all arrays, fast-travel flags) except as marked research-only.

---

## 11. Methodology notes

- SDK classes/fields/functions came from `Pal_classes.hpp`, `Pal_structs.hpp`, `Engine_classes.hpp`, and `GameDefines.hpp`.
- Hook sites were checked by disassembling `dump.bin` and following SDK thunks to their first direct call target. Many SDK “native” addresses are UFunction thunks; the real implementation is the call target used above.
- AOB uniqueness was checked against the local dump. The dump is partial; data/vtables at `0x7AEEAC8` / `0x7AEF6C8` and the GameSetting object are outside the file and come from proven patches.
- The local dump’s Infinite Stamina site appears to contain a `jmp` (patched state); reuse the AOB from the existing Infinite Stamina patch rather than re-deriving it from the dump bytes.
- Gameplay semantics are **not** proven by this catalog. Each entry needs a short in-game test, especially shared functions and save-affecting fields.
