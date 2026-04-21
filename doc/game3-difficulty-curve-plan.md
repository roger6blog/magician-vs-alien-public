# Plan: Game3 難度曲線重設計（含節奏系統整合）

---

## Context

節奏系統已上線，計分改為「基礎 1 分 + 精度定額紅利 0-3 分」。  
原本難度曲線是針對純計數設計，現在需要同時考慮節奏難度的漸進。  
本計畫包含**原始版本（現況）**與**建議新版本**兩套對照，供 review 後選擇。

---

## 原始版本（現況）對照表

### 難度分級（`Flow_InitializeQuestion`）

| 階段 | playerAnswerRightCount | 外星人數量 | Mode | 轉移次數 |
|------|----------------------|-----------|------|--------|
| 教程 1 | 0–2 | 1–3 | intro | 0 |
| 教程 2 | 3–6 | 1–5 | intro | 0 |
| 簡單 1 | 7–10 | 3–6 | easy | 1 |
| 簡單 2 | 11–15 | 3–8 | easy | 1 |
| 困難 1 | 16–18 | 5–10 | hard | 2 |
| 困難 2 | 19+ | lv/4 ~ lv/2 | hard | 2 |

### 外星人速度（`PlayStepAnimation`）
```
duration = 3 - playerAnswerRightCount * 0.03f  (最小 1.0s)
lv 0  → 3.0s / lv 20 → 2.4s / lv 50 → 1.5s / lv 67+ → 1.0s (上限)
```

### 速度隨機化（`WarehouseController.CreateSelfAlien`）
```
alienDuration = duration * Random.Range(0.6f, 1.4f)  // 所有難度一律 ±40%
```

### 進屋順序隨機化（現況）
```
Random.value > 0.5f → 左先 or 右先  // 所有難度一律 50% 隨機
```

### 答題時間（現況）
```
固定 9 秒，所有難度相同
```

### 節奏系統（現況）
- 速度變異 ±40% → lv0 就是最大混亂度，玩家無法學習節奏
- 進屋隨機 50% → lv0 就是完全隨機，無漸進學習

---

## 問題診斷

| 問題 | 說明 |
|------|------|
| 節奏難度沒有漸進 | 速度變異 ±40% 從第一題就全開，玩家來不及學習節奏機制 |
| 進屋隨機沒有漸進 | 順序一直是 50% 隨機，低等級玩家無法建立「左→右」的預測模式 |
| 高難度速度上限過早 | lv67 就達到最小 1.0s，之後只靠數量加壓，缺乏速度層面的持續挑戰 |
| 答題時間固定 | 高難度外星人數更多、動畫更快，9 秒壓力其實一樣（相對工作記憶負荷不等） |

---

## 建議新版本

### 總設計原則

- **lv 0–10**：學習期。外星人速度接近固定，進屋順序固定，讓玩家先學會「看哪邊多」的工作記憶任務
- **lv 11–20**：節奏引入期。開始出現輕微速度變異和偶爾的順序隨機，玩家開始需要同時處理節奏
- **lv 21–35**：節奏成熟期。變異增加，轉移頻繁，節奏和計數雙重壓力
- **lv 36+**：專家期。全開隨機、全開速度變異、答題時間略微壓縮

---

### 新難度分級表

| 階段 | lv 範圍 | 外星人數量 | Mode | 轉移 | 速度變異 | 進屋隨機率 | 答題時間 |
|------|--------|-----------|------|------|---------|-----------|--------|
| 教程 1 | 0–3 | 1–3 | intro | 0 | ±5% | 0%（固定左先） | 10s |
| 教程 2 | 4–8 | 2–5 | intro | 0 | ±10% | 10%（右側緩衝） | 10s |
| 簡單 1 | 9–14 | 3–6 | easy | 1 | ±15% | 20% | 9s |
| 簡單 2 | 15–20 | 3–8 | easy | 1 | ±25% | 35% | 9s |
| 困難 1 | 21–28 | 5–10 | hard | 2 | ±30% | 50% | 9s |
| 困難 2 | 29–40 | lv/4~lv/3 | hard | 2 | ±35% | 50% | 8s |
| 專家 | 41+ | lv/4~lv/2 | hard | 2 | ±40% | 50% | 7s |

---

### 各變因詳細設計

#### 1. 速度變異（替代現況 ±40% 固定值）

```csharp
float GetSpeedVariation(int lv)
{
    if (lv <= 3)  return 0.05f;   // ±5%
    if (lv <= 8)  return 0.10f;   // ±10%
    if (lv <= 14) return 0.15f;   // ±15%
    if (lv <= 20) return 0.25f;   // ±25%
    if (lv <= 28) return 0.30f;   // ±30%
    if (lv <= 40) return 0.35f;   // ±35%
    return 0.40f;                  // ±40%（現況最大值）
}

// 使用：alienDuration = duration * Random.Range(1f - variation, 1f + variation)
```

需在 `PlayStepAnimation` 計算後傳入 `CreateSelfAlien` / `MoveAlien`，並更新兩者的簽名加 `float speedVariation` 參數。

#### 2. 進屋順序隨機率

```csharp
float GetEntryRandomChance(int lv)
{
    if (lv <= 3)  return 0f;    // 固定左先（建立基本預測模式）
    if (lv <= 8)  return 0.10f; // 10% 右側機率：讓玩家知道右邊也會出怪，緩解 lv9 的衝擊
    if (lv <= 14) return 0.20f;
    if (lv <= 20) return 0.35f;
    return 0.50f;               // 之後維持 50%
}

// 使用（替換現況 `Random.value > 0.5f`）：
float entryChance = GetEntryRandomChance(playerAnswerRightCount);
bool rightFirst = Random.value < entryChance * 0.5f; // 右先的機率
if (!rightFirst) { 左先 } else { 右先 }
```

#### 3. 外星人數量範圍（微調）

```
lv 0–3:   1–3（現況不變）
lv 4–8:   2–4（現況 1–5 調整，避免太早出現大數量）
lv 9–14:  3–6（現況不變）
lv 15–20: 3–8（現況不變）
lv 21–28: 5–9（現況 5–10，上限略降避免節奏過於密集）
lv 29–40: lv/4 ~ lv/3（現況 lv/4 ~ lv/2 上限降低）
lv 41+:   lv/4 ~ lv/2（現況不變）
```

#### 4. 外星人速度（duration 公式）

```csharp
float GetBaseDuration(int lv)
{
    if (lv <= 70)
        // 3.0s 平滑降到 1.0s，自然在 lv 67 觸底並維持
        return Mathf.Max(3f - lv * 0.03f, 1.0f);
    else
        // lv 71 起突破 1.0s，每級 -0.01s，lv 90 達到 0.8s 下限
        return Mathf.Max(1.0f - (lv - 70) * 0.01f, 0.8f);
}
```

> ⚠️ 不在 lv 41 強制切換到 1.0s：原公式在 lv 40 時 duration = 1.8s，若 lv 41 直接跳 1.0s 會造成速度瞬間暴增 44%，玩家體感斷崖。讓公式自然收斂至 lv 67 才是平滑過渡。

#### 5. 答題時間

```csharp
float GetAnswerTime(int lv)
{
    if (lv <= 8)  return 10f;  // 學習期多給時間
    if (lv <= 28) return 9f;   // 現況
    if (lv <= 40) return 8f;
    return 7f;                 // 專家期
}
```

UI 文字使用 `string.Format("有{0}秒時間可以選擇！", (int)GetAnswerTime(lv))` 動態顯示，確保 i18n 友善。

#### 6. 轉移觸發門檻（微調）

```
intro (無轉移): lv 0–8（現況 0–6，稍微延長學習期）
easy (1次轉移): lv 9–20（現況 7–15）
hard (2次轉移): lv 21+（現況 16+）
```

---

### 各難度預期節奏體驗

| 等級 | 節奏體驗描述 |
|------|------------|
| lv 0–3 | 外星人速度幾乎一致，順序固定左先，節奏機制輕鬆入門 |
| lv 4–8 | 速度極輕微差異，偶爾（10%）右先出現，讓玩家建立「右邊也有外星人」的預期 |
| lv 9–14 | 偶爾有快一點或慢一點的外星人，20% 機率右先出現 |
| lv 15–20 | 明顯感受到速度差異，順序更不固定，開始需要同時追蹤節奏 |
| lv 21–28 | 雙轉移加上 ±30% 速度變異，節奏和計數雙重壓力顯著 |
| lv 29–40 | 數量快速增長（lv/3 上限），速度變異 ±35%，答題時間縮短至 8s |
| lv 41+ | 完整難度開放，偶爾外星人超越，答題時間 7s |

---

## 實作步驟

### Step 1：`Game3_MainController.cs`

- 新增三個 helper：`GetSpeedVariation(lv)`、`GetEntryRandomChance(lv)`、`GetAnswerTime(lv)`、`GetBaseDuration(lv)`
- 修改 `Flow_InitializeQuestion` 的 level bracket（外星人數量、mode、轉移門檻）
- 修改進屋順序隨機邏輯（使用 `GetEntryRandomChance`）
- 修改 `Flow_WaitForUserAnswer` 的 WaitForSeconds 從固定 9f 改為 `GetAnswerTime(playerAnswerRightCount)`
- 修改 `ShowStepNarration` 裡「有9秒時間」的 UI 文字：使用 `string.Format("有{0}秒時間可以選擇！", (int)GetAnswerTime(playerAnswerRightCount))` 確保 i18n 友善
- 修改 `PlayStepAnimation` 計算 `speedVariation` 並傳入 CreateSelfAlien/MoveAlien
- 修改 `PlayStepAnimation` 使用 `GetBaseDuration(playerAnswerRightCount)` 替代舊公式

### Step 2：`Game3_WarehouseController.cs`

- `CreateSelfAlien` 加 `float speedVariation = 0.4f` 參數（預設值向下相容）
- `MoveAlien` 同上
- 內部 `Random.Range(0.6f, 1.4f)` 改為 `Random.Range(1f - speedVariation, 1f + speedVariation)`

---

## 關鍵檔案

| 檔案 | 改動 |
|------|------|
| `Assets/Scripts/Game3/Game3_MainController.cs` | 難度分級、helper 函式、答題時間動態化、speedVariation 傳遞 |
| `Assets/Scripts/Game3/Game3_WarehouseController.cs` | CreateSelfAlien / MoveAlien 加 speedVariation 參數 |

---

## 驗證方式

1. lv 0–3 期間：所有題目 100% 固定左先進屋，外星人速度視覺上幾乎一致
2. lv 4–8 期間：90% 左先進屋，偶爾（10%）右先出現以打破預期；速度仍接近一致
3. lv 9 起：右先機率升至 20% 以上，外星人速度開始有輕微差異可感知
4. lv 21+：雙轉移正確觸發，順序 50% 隨機
5. lv 41+：答題倒數 UI 顯示 7 秒，確認 GetAnswerTime 正確回傳
6. lv 71+：確認 duration 低於 1.0s 但不低於 0.8s
7. 節奏命中率在 lv 0 期間應明顯高於 lv 41+（速度變異更小 → 更易命中）
