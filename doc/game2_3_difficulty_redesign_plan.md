# Plan: Game2-3 難度曲線重設計

---

## 遊戲機制（Game2-3 背景）

Game2-3 是一個「辨識落下物件大小並按對應按鈕」的反應遊戲：

- **落下物件**：從畫面上方落下，外觀分 3 種（水/石/柱），每種各有 5 個尺寸變體（variant 0–4），由 `random_m = Mathf.RoundToInt(4 * rr)` 決定（等機率 0–4）
- **按鈕**：畫面下方固定 5 個按鈕，顯示數字 7 / 8 / 9 / 10 / 11（btn1–5）
- **正確答案**：`GetCorrectButton() = currentMonsterVariant + 1`，即 variant 0 → btn1（7），variant 4 → btn5（11）；**數字範圍永遠固定，不隨等級增加**
- **外觀與答案的關係**：落下物件的「類型（水/石/柱）」只是視覺變化，不影響正確答案；**正確答案完全由「尺寸 variant」決定**
- **生命**：3 條心，誤按或子彈落地各扣一條
- **等級**：從玩家歷史最高分 -10 開始（`mylevel = bestScore - 10`），答對一題 myscore +1，myscore 超過 mylevel 時 mylevel 也跟著 +1

**難度的唯一可控維度**：由於數字範圍固定（永遠 5 選 1），難度只能靠兩個參數調整：
1. 子彈落下速度（越快越難辨識和反應）
2. 子彈生成間隔（越短同屏壓力越大）

---

## Context

game2-3 的難度曲線有兩個問題：
1. **速度在 lv50 封頂**：使用離散步進公式，lv50 之後速度完全停止成長
2. **生成間隔 bug**：動態縮短的計算被後一行 `timeLeft = 5f` 蓋掉，實際上間隔永遠 5 秒

目標：套用 game4 的連續成長曲線結構，同時以 lv40（小六極限）、lv110（金氏紀錄極限）為設計里程碑。

---

## 修改檔案

唯一要改的檔案：`Assets/Scripts/Game2/score2_3.cs`

---

## 修改一：速度公式 + 烏龜懲罰（~line 439–444）

### 現況（刪除）
```csharp
float speed = Mathf.FloorToInt(mylevel / 10);
if (speed > 5) speed = 5;
if (finalParams.showCatEffect) speed = 0;
Obj_Clone.GetComponent<move2>().speed = (0.01f + 0.0035f * speed) * finalParams.speedMultiplier;
```

### 改為
```csharp
const float TURTLE_SPEED_BONUS = 0.025f; // 烏龜懲罰固定加速值（低等級相對效果大，高等級自然遞減）

float baseSpeed = 0.010f + mylevel * 0.001f;
if (finalParams.showCatEffect)
    baseSpeed = 0.010f;

bool isTurtlePenalty = finalParams.speedMultiplier > 1.0f;
float bulletSpeed = isTurtlePenalty ? baseSpeed + TURTLE_SPEED_BONUS : baseSpeed;

Obj_Clone.GetComponent<move2>().speed = bulletSpeed;
```

**設計說明：加法懲罰 → 相對效果自然遞減**

固定加 0.025f，不用 hard cap 或分段切換：

| 等級 | 正常速度 (u/s) | 烏龜速度 (u/s) | 懲罰倍率 |
|------|--------------|--------------|--------|
| 0    | 0.60 | 2.10 | **3.5×** |
| 20   | 1.80 | 3.30 | 1.83× |
| 40   | 3.00 | 4.50 | 1.50× |
| 80   | 5.40 | 6.90 | 1.28× |
| 110  | 7.20 | 8.70 | 1.21× |
| 200  | 12.6 | 14.1 | **1.12×**（輕微，感覺不大）|

低等級（lv0）懲罰感受強烈，高等級懲罰感受自然淡化，**不需要 hard cap 或切換邏輯**。

---

## 修改二：生成間隔 + 烏龜間隔懲罰（~line 297–304）

### 現況（刪除）
```csharp
timeLeft=5f-(myscore/10);

if(timeLeft<1f){
    timeLeft=1f;
}

timeLeft = 5f;   // ← bug：蓋掉上面所有計算
```

### 改為
```csharp
timeLeft = Mathf.Max(5.0f - mylevel * 0.03f, 1.0f);
```

**說明：**
- 改用 `mylevel`（玩家整體水準），不用 `myscore`（當局即時分數）
- lv0 = 5.0s，每升 1 級縮短 0.03s，lv133 觸及下限 1.0s
- 烏龜懲罰只影響速度（修改一），生成間隔不受影響

---

## 難度里程碑

| 等級 | 落速 (u/s) | 子彈落地時間 | 生成間隔（正常）|
|------|-----------|------------|--------------|
| 0    | 0.60 | ~20s | 5.0s |
| 20   | 1.80 | ~6.7s | 4.4s |
| **40** | **3.00** | **4.0s** | **3.8s** ← 小六極限 |
| 80   | 5.40 | ~2.2s | 2.6s |
| **110** | **7.20** | **1.67s** | **1.7s** ← 金氏紀錄極限 |
| 133  | 8.58 | ~1.4s | **1.0s ← 封頂** |
| 200  | 12.6 | ~1.0s | 1.0s |

---

## 不需要修改的部分

| 元件 | 理由 |
|------|------|
| 子彈 prefab（c_water/c_stone/c_pillar）| 視覺內容不變 |
| random_m / random_type | 5 選 1 機制不變 |
| Miss_score / Miss_score2 / get_heart | 生命扣除邏輯不變 |

---

## 驗收標準

1. lv0 子彈落速 0.6 u/s，肉眼緩慢
2. lv50 落速 3.6 u/s（舊版 lv50 為 1.65 u/s），確認難度提升
3. 生成間隔不再固定 5 秒，Console log 確認 timeLeft 隨 mylevel 縮短
4. 貓咪能力觸發 baseSpeed = 0.010f（最低速）
5. lv100+ 速度持續線性成長，無封頂
6. 烏龜懲罰 lv0：速度 0.035f（2.1 u/s），約 3.5× 正常速度
7. 烏龜懲罰 lv40：速度 0.075f（4.5 u/s），約 1.5× 正常速度
8. 烏龜懲罰 lv110：速度 0.145f（8.7 u/s），約 1.2× 正常速度（懲罰感輕微）
