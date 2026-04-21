# 魔數動物充能公式完整參考

> 最後更新：2026-04-21
> 對應程式碼：`Assets/Scripts/AbilityManager.MagicAnimal.cs`

---

## 一、總公式

```
gain = Clamp01( baseGain × baseMultiplier + uniqueBonus × uniqueMultiplier )
```

充能後存入 `magicAnimalChargeProgress[gameName]`（0.0 ~ 1.0，觸發閾值 = 1.0）。

### 動物等級乘數

| 動物等級 | baseMultiplier | uniqueMultiplier |
|---------|---------------|-----------------|
| Lv1     | 1             | 1               |
| Lv2     | 1             | **2**           |
| Lv3     | **2**         | **2**           |

> Lv2 只放大 unique；Lv3 同時放大 base 和 unique。

---

## 二、T 值（動畫秒數）說明

| 遊戲 | T 值來源 | 典型範圍 |
|------|---------|---------|
| Game3 | 該題外星人動畫時間（`_currentAnimationDuration`） | ~1.0s（高 lv）~ 9.0s（低 lv） |
| Game1 / 2 / 4 | 固定為 0（不傳入） | — |

T 值決定 Game3 中所有以 `T×係數` 表示的 base 與 unique 加成幅度：外星人數量越多、速度越慢，T 越大，充能越快。Game1/2/4 使用 flat 加成，不依賴 T。

---

## 三、各動物充能公式

### 🦁 獅王雷歐（Lion）— 難度等級加成

答錯或失命時 combo 重置，但 Lion 本身沒有 combo 機制（unique 依 level 分段）。

| 組成 | Game3 | Game2 | Game1 / 4 |
|------|-------|-------|-----------|
| **base** | 0.020 + T×0.002 | 0.020 | 0.020 |
| **unique**（依遊戲難度 level） | lv<10：**T×0.001**<br>lv10–19：T×0.002<br>lv20+：T×0.004 | lv<10：**0**<br>lv10–19：0.010<br>lv20+：0.015 | lv<3：**0**<br>lv3–5：0.010<br>lv6+：0.015 |

> **Game2 備註**：`level` = 當前分數；lv10 = 分數達到 10，進入中期玩家才有 unique，前期無加成。Gate 較晚但可接受。
> **Game3 備註**：lv<10 原為 0，已調整為 T×0.001，讓低難度玩家也有輕微充能，避免初學者感覺過慢。

---

### 🐶 好朋友迪米（Dog）— Combo 連答加成

Combo 計數在 `MagicAnimalChargeCalculator._combo` 內部維護。答對 +1，答錯/失命呼叫 `ResetCombo()` 歸零。

| 組成 | Game3 | Game1 / 2 / 4 |
|------|-------|--------------|
| **base** | 0.020 + T×0.002 | 0.020 |
| **unique**（依 Combo 數） | combo<3：0<br>combo 3–5：**T×0.001**<br>combo≥6：**T×0.003** | combo<5：0<br>combo 5–9：0.005<br>combo≥10：0.010 |

> **設計考量**：Game3 的 unique 從 flat 值改為 T 驅動，讓高 T（慢速難題）的 combo 獎勵更有意義；快速題（低 T）即使連答也不會過度充能。
> Game1/2/4 的 combo 門檻為 5（vs Game3 的 3），對應兩者題目答題節奏的差異。

---

### 🐢 小管家圖圖（Turtle）— 時間加成

| 組成 | Game3 | Game1 / 2 / 4 |
|------|-------|--------------|
| **base** | 0.020 | 0.020 |
| **unique** | T×0.004 | 固定 0.006 |

> Game1/2/4 的 T=0，若改為 T×係數 unique 會變 0，故保留 flat 0.006。
> 以 T=1.5s（Game3 高 lv）為基準：unique = 0.006，與 Game1/2/4 持平；T=3s：0.012（2 倍），鼓勵挑戰較難題目。

---

### 🐱 喵星人蜜亞（Cat）— 答對充能 + 失命充能 + 每日結算溢出

Cat 在「答對題目」的 unique 恆為 0，主要充能機制是失命事件與每日結算。

#### 答對題目（`OnCorrectAnswer`）

| 組成 | Game3 | Game1 / 2 / 4 |
|------|-------|--------------|
| **base** | 0.020 + T×**0.0035** | 0.020 |
| **unique** | 0 | 0 |

> Cat 沒有 unique 充能（在答對路徑），所以給予較高的 T 係數（0.0035 vs 其他動物的 0.002）作為補償。

#### 失命充能（`OnHeartLost`）

| 等級 | Game3 | Game1 / 2 / 4 |
|------|-------|--------------|
| Lv1 | +0.300 | +0.150 |
| Lv2 / Lv3 | +0.300 **+0.150** = **0.450** | +0.150 **+0.150** = **0.300** |

> **Lv2+ 特例**：其他動物的 unique 在 Lv2 為 ×2 倍率，但 Cat 若 ×2 在 Game3 會達到 +0.600（過強），故改為固定 +0.150 的平坦加成。

#### 每日時間結算溢出（`OnDailyTimeExhausted`，Game3 限定）

當 Game3 每日遊戲時間耗盡時（`Flow_NoTime`），若動物為 Cat：

```
溢出充能 = remainingLives × 0.150
newCharge = currentCharge + 溢出充能   ← 刻意不 Clamp01，允許 > 1.0
```

| 剩餘命數 | 溢出加成 | 示例（當前 charge=0.8） |
|---------|---------|----------------------|
| 1 命 | +0.150 | → 0.950 |
| 2 命 | +0.300 | → 1.100（溢出） |
| 3 命 | +0.450 | → 1.250（溢出） |

> 剩餘愛心視為「本局省下的失命充能機會」，作為每日補償機制，可超過 1.0 形成溢出能量。
> 溢出能量在 UI 仍顯示為滿格（`Mathf.Min(charge, 1f)`），觸發能力後以 remainder 保留（見第四章）。

---

### 🦉 魔法師歐洛（Owl）— 分數表現加成

| 組成 | Game3 | Game1 / 2 / 4 |
|------|-------|--------------|
| **base** | 0.020 + T×0.002 | 0.020 |
| **unique**（依分數表現） | bestScore≥10 且 score≥best×0.8：**base×0.5**<br>否則：0 | bestScore<10（新手）：0.005<br>score≥best×0.8（接近最高）：**0.015**<br>否則（老手爛局）：**0.005** |

> **Game1/2/4 設計說明**（歷次調整後的版本）：
> - 原本「老手爛局 = 0」導致老手在狀態差的局比新手充能更慢，產生「進度懲罰」感。
> - 已修正為老手爛局提供與新手相同的底線 0.005。
> - 接近最高分的加成從 0.020 降為 0.015，避免好局充能過快。
>
> **Game3 設計說明**：
> base×0.5 代表「額外 50% base 充能」，觸發門檻為 bestScore≥10 且 score≥best×0.8。

---

## 四、溢出能量機制（Overcharge）

### 設計原則

一般充能受 `Clamp01` 限制，最高只能達到 1.0。能力觸發後，舊設計會將 charge 歸零；**新設計改為保留溢出部分**：

```
remainder = Max(0, charge - 1.0)
charge 設為 remainder（而非 0）
```

### 溢出能量的來源

| 來源 | 動物 | 機制 |
|------|------|------|
| Cat 每日結算 | Cat | `OnDailyTimeExhausted`，可注入 charge > 1.0 |
| 一般充能 | 所有 | `IncreaseMagicAnimalCharge` 使用 Clamp01，**無法超過 1.0** |

> 目前只有 Cat 的每日結算能夠產生真正的溢出能量（charge > 1.0）。其他動物的充能路徑受 Clamp01 保護，不會溢出。

### 溢出能量的消耗流程

```
[觸發能力]
  beforeCharge = GetGameChargeProgress()    // e.g. 1.25
  ExecuteMagicAnimalAbility()               // 執行能力效果
  remainder = Max(0, beforeCharge - 1.0)   // 0.25
  SetLocalCharge(remainder)                 // 保留 0.25
  PushChargeToServer(remainder)             // 同步到 server
  OnMagicAnimalChargeChanged(Min(remainder, 1f))  // UI 更新（顯示 0.25）
```

---

## 五、充能速度比較表（Lv1，game1/4 基準）

| 動物 | 最佳條件 | 每題 gain | 填滿需答對 |
|------|---------|-----------|---------|
| 🦁 Lion | lv6+（高難度） | 0.035 | **~29 題** |
| 🦁 Lion | lv<3（初學） | 0.020 | 50 題 |
| 🐶 Dog | combo≥10 | 0.030 | ~33 題 |
| 🐶 Dog | combo<5 | 0.020 | 50 題 |
| 🐢 Turtle | 無條件 | 0.026 | ~38 題 |
| 🐱 Cat | 答對（+失命） | 0.020 + 失命 | 50題（+失命即時大充能） |
| 🦉 Owl | 接近最高分 | 0.035 | **~29 題** |
| 🦉 Owl | 新手 / 老手爛局 | 0.025 | 40 題 |

> Lv2 時 unique×2，Lv3 時 base×2 + unique×2，整體效率倍增。
> Cat 的答對充能最慢，但失命時 +0.150（Lv1）/+0.300（Lv2+）是即時大幅充能，整體不落後。

---

## 六、平衡性評估與設計決策記錄

### 已確認問題與修正

| 時間 | 問題 | 修正 |
|------|------|------|
| 2026-04 | Dog/Owl base (Game3) 誤設為 T×0.0035 | 改回 T×0.002（Cat base 同時升為 T×0.0035 補償） |
| 2026-04 | Dog unique (Game3) 使用 flat 值（0.030/0.060），高 T 題目不夠有區分性 | 改為 T×0.001 / T×0.003，與動畫時長掛鉤 |
| 2026-04 | Lion unique (Game3) lv<10 為 0，初學者完全沒有 unique | 改為 T×0.001，讓低難度也有輕微加成 |
| 2026-04 | Owl (Game1/2/4) 老手爛局 unique=0，比新手慢充能 | 老手爛局改為 0.005（與新手底線對齊） |
| 2026-04 | Owl (Game1/2/4) 接近最高分 unique=0.020，好局最快 25 題填滿（其他動物 33–50 題） | 降為 0.015，填滿需~29 題，與 Lion lv6+ 對齊 |

### 設計確認（維持現狀的決策）

| 項目 | 結論 | 理由 |
|------|------|------|
| Lion Game2 lv<10 = 0 unique | ✅ 可接受 | game2 `level` = 當前分數，lv10 在正常遊戲節奏中很快達到 |
| Dog combo 門檻 game1/2/4=5 vs game3=3 | ✅ 有意設計 | game3 題目節奏更快，連答三題即觸發比較合理 |
| Cat unique (答對) = 0 | ✅ 有意設計 | Cat 主要充能在失命，答對的 base 係數已提高補償 |
| Cat Lv2+ 失命用 +0.15 flat 而非 ×2 | ✅ 有意設計 | 若 ×2 在 game3 = +0.600，過強，改用 flat 控制上限 |
| Owl Game3 bestScore<10 = 0 unique | ✅ 有意設計 | Game3 bestScore 的分數意義不同（分數=答對題×節奏），保留 unique=0 |

### 已知的架構差異（非 bug）

| 項目 | 說明 |
|------|------|
| game2-1 (`score.cs`) 充能從 setter 觸發 | `myscore` setter 中 `value > _myscore` 才呼叫，等效於答對觸發 |
| game2-1 combo 重置路徑 | `Miss_score*` → `AddEnergyOnHeartLost` → `OnHeartLost` → `ResetCombo()`（vs 其他遊戲直接呼叫 `ResetEnergyCombo()`，效果等效） |
| game2-1 每次答錯 = 失命 | 故 Cat 在 game2-1 每次答錯都獲得失命充能，game2-2/2-3 答錯與失命是獨立事件 |

---

## 七、相關程式碼位置

| 功能 | 檔案 | 關鍵位置 |
|------|------|---------|
| 充能計算核心 | `Assets/Scripts/AbilityManager.MagicAnimal.cs` | 全檔 |
| 充能入口（答對） | `Assets/Scripts/AbilityManager.cs` | `AddEnergyOnCorrectAnswer` |
| 充能入口（失命） | `Assets/Scripts/AbilityManager.cs` | `AddEnergyOnHeartLost` |
| 充能入口（每日結算） | `Assets/Scripts/AbilityManager.cs` | `OnDailyTimeExhausted` |
| 能力觸發與溢出處理 | `Assets/Scripts/AbilityManager.cs` | `TriggerMagicAnimal`, `ExecuteMagicAnimalAbility` |
| 充能存取 | `Assets/Scripts/AbilityManager.cs` | `SetLocalCharge`, `GetGameChargeProgress` |
| Game3 充能呼叫 | `Assets/Scripts/Game3/Game3_MainController.cs` | `Flow_CheckCorrect`, `Flow_OnHeartLost`, `Flow_NoTime` |
| Game4 充能呼叫 | `Assets/Scripts/score_g4.cs` | 答對/答錯回呼附近 |
| Game2-1 充能呼叫 | `Assets/Scripts/Game2/score.cs` | `myscore` setter |
| Game2-2/2-3 充能呼叫 | `Assets/Scripts/Game2/score2_2.cs`, `score2_3.cs` | 答對/答錯 callback |
