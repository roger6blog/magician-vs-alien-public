# Plan: Game4 飛彈追蹤修正（Homing Missile）

---

## Context

### 玩家描述的症狀

> 「目前 game4 的飛彈都是垂直往上射，直到飛彈碰到子彈落下時的 Y 座標才消失。  
> 如果子彈落下時的 X 座標跟發射的飛彈差太遠，  
> 那就會變成子彈和飛彈完全沒碰到，但還是命中的詭異情況。  
> 希望飛彈射出去的時候能追蹤到落下的子彈軌跡，然後正確地命中。  
> 飛彈應該從原本位置發射，斜向追蹤到落下的子彈，而不是直接調整發射 X 座標。」

---

### 實際碰撞機制（已驗證）

```
hit_obj_s（玩家飛彈）
    BoxCollider2D: isTrigger=true
    local size X = 5.18，localScale X = 2.68
    → world 寬度 ≈ 13.88 units（X: -6.94 ~ +6.94，覆蓋全螢幕）

enemyBulletRoot（right_t / left_t）
    CircleCollider2D: radius=0.5，isTrigger=false
    位置：(spawnX ≈ 0, 6, 0) → 隨怪物波下落
    ├── bullet_Right（視覺怪物，世界 X: Random.Range(2,5)）
    └── bullet_Left（視覺怪物，世界 X: Random.Range(-2,-5)）
```

碰撞原理：飛彈的全寬 trigger 上升時，任何 Y 相交的 CircleCollider2D 都會觸發，**X 位置完全無關**。  
→ 這就是「沒碰到卻算中」的根本原因：collision 是純 Y 軸判斷。

### 為什麼「只加 homing」不夠

即使飛彈往 X=3 飛去，其 BoxCollider2D 仍橫跨 ±6.94。  
只要飛彈 Y 到達 container Y，trigger 就先觸發，此時飛彈可能還在 X=1，  
視覺上仍是「沒到怪物身上就算中了」。

### 正確修法

**縮小玩家飛彈的 BoxCollider2D 寬度（X 方向）** + **container 移至正確答案側視覺 X** + **飛彈 homing**

這樣：
- 飛彈 collider 縮窄後，X 距離夠大就不觸發
- Container 對齊怪物 X，飛彈 home 過去時才真正重疊
- 視覺、碰撞、邏輯三者一致

### 錯誤答案的行為（新）

- 玩家按錯方向 → 飛彈 home 到「錯誤側怪物」(X ≈ ±3)
- Container（正確答案）在另一側
- 飛彈飛到錯誤側，窄 collider 不觸碰正確 container → 飛出螢幕
- Container 落到 end_bar → `Miss_score2()` 扣心（延遲判定）

> ⚠️ 差異：現行按錯立即 Miss_score；新行為等 container 落地才扣。
> 視覺上更直觀（「打到空，它落下來了」），可接受。若需立即反饋可後續加入。

---

## 修改計畫

### 零、縮小玩家飛彈的 BoxCollider2D（Unity Inspector）

**位置：** 場景中 `hit_obj_s`（hit_right）和對應的 hit_left 物件

**為何需要縮小？**

飛彈雖然從 (0, -6) 發射並斜向飛去，但其 BoxCollider2D 寬度 ≈13.88 units，
「還在 X=1 就已觸碰到 X=3 的 container」。縮窄 collider 讓 X 位置有意義，
飛彈必須真正飛到 container 附近才觸發碰撞。

> 飛彈**發射位置不變**（X=0, Y=-6），只改 collider 寬度。

- 目前 BoxCollider2D size.x = 5.18（world ≈ 13.88，覆蓋全螢幕）
- **改為 size.x ≈ 0.3**（world ≈ 0.8），讓 X 精準判斷

> 注意：`hit_obj_s` 的 localScale.x = 2.68，local size.x = 0.8 / 2.68 ≈ 0.3

同樣操作 hit_left 對應的物件。

---

### 一、score_g4.cs

**位置：** `Assets/Scripts/score_g4.cs`

#### 1-1. 新增兩個公開欄位（在欄位區，~line 65 附近）

```csharp
/// <summary>飛彈 homing 用目標：右側視覺怪物 Transform</summary>
public Transform currentRightTarget;
/// <summary>飛彈 homing 用目標：左側視覺怪物 Transform</summary>
public Transform currentLeftTarget;
```

#### 1-2. SpawnBullet() 開頭，重置目標（在現有 `hasFiredThisQuestion = false;` 下方）

```csharp
currentRightTarget = null;
currentLeftTarget = null;
```

#### 1-3. SpawnBullet() 怪物分支：把 container（落下的怪物容器）移到正確答案側的 X

> **注意**：這是改變「從上落下的怪物容器」的生成 X，不是改變「底部飛彈」的發射 X。
> 飛彈仍從 (0, -6) 發射，斜向飛往 container 所在位置。

**現行（~line 231）：**
```csharp
Vector3 enemyBulletRootPosition = new Vector3(spawnX, 6f, 0);
```

**改為（移到怪物分支內，在 bullet_Right / bullet_Left 生成後、container 生成前）：**
```csharp
// container 放在正確答案側的視覺 X，讓飛彈 homing 時物理接觸正確
Vector3 enemyBulletRootPosition = (random_m2_r > random_m2_l)
    ? new Vector3(bullet_Right_Position.x, 6f, 0)  // 右側是正確答案
    : new Vector3(bullet_Left_Position.x, 6f, 0);  // 左側是正確答案
```

注意：原始的 `Vector3 enemyBulletRootPosition = new Vector3(spawnX, 6f, 0);`（~line 231）
改成只留 `spawnX` 的宣告（item 分支仍需用），monster 分支內覆寫位置。

實際做法：

```csharp
// ~line 231（保留 spawnX 計算，但移除 enemyBulletRootPosition）
float spawnX = 0f;
if (AlienFaceController.instant != null)
    spawnX = AlienFaceController.instant.transform.position.x;
Vector3 enemyBulletRootPosition = new Vector3(spawnX, 6f, 0); // item 分支用，monster 分支下方覆寫
```

在 `if (random_m < 6)` 怪物分支內，bullet_Right / bullet_Left 生成後（~line 263 後），container 生成前（~line 265）：

```csharp
// 覆寫 container 位置：對齊正確答案側的視覺怪物
enemyBulletRootPosition = (random_m2_r > random_m2_l)
    ? new Vector3(bullet_Right_Position.x, 6f, 0)
    : new Vector3(bullet_Left_Position.x, 6f, 0);
```

#### 1-4. SpawnBullet() 怪物分支：生成 container 後設定 target

在 `bullet_Right.transform.parent = ...` 和 `bullet_Left.transform.parent = ...` 之後：

```csharp
currentRightTarget = bullet_Right != null ? bullet_Right.transform : null;
currentLeftTarget  = bullet_Left  != null ? bullet_Left.transform  : null;
```

---

### 二、hit_right.cs

**位置：** `Assets/Scripts/hit_right.cs`

**修改 FixedUpdate（~line 79–98）：** 原本直線向上，改為 homing

**現行：**
```csharp
void FixedUpdate () {
    if (_btn_right_click.btn_r_cc ) {
        rb.MovePosition(rb.position + new Vector2(0, 0.3f));
        if(rb.position.y>8){
            gameObject.transform.position = new Vector3 (0, -6, 0);
            _btn_right_click.btn_r_cc=false;
        }
    }
}
```

**改為：**
```csharp
void FixedUpdate () {
    if (_btn_right_click.btn_r_cc ) {
        Transform target = _score != null ? _score.currentRightTarget : null;

        // 飛彈速度：homing 法則要求至少 3–4× 目標速度
        // 怪物最高速 0.20 → 每秒 12 units；1.2f × 50Hz = 60 units/s（5×，確保追得上）
        float missileSpeed = 1.2f;

        if (target != null) {
            Vector2 dir = ((Vector2)target.position - rb.position).normalized;
            rb.MovePosition(rb.position + dir * missileSpeed);
        } else {
            // Fallback：直線向上（目標已消失或 item 波）
            rb.MovePosition(rb.position + new Vector2(0, missileSpeed));
        }
        // 速度加快後，邊界從 8 調高至 10，確保特效有足夠空間播完
        if(rb.position.y > 10){
            gameObject.transform.position = new Vector3 (0, -6, 0);
            _btn_right_click.btn_r_cc = false;
        }
    }
}
```

---

### 三、hit_left.cs

**位置：** `Assets/Scripts/hit_left.cs`

**修改 FixedUpdate（~line 82–101）：** 同上，使用 `currentLeftTarget`

```csharp
void FixedUpdate () {
    if (_btn_left_click.btn_l_cc ) {
        Transform target = _score != null ? _score.currentLeftTarget : null;
        float missileSpeed = 1.2f;
        if (target != null) {
            Vector2 dir = ((Vector2)target.position - rb.position).normalized;
            rb.MovePosition(rb.position + dir * missileSpeed);
        } else {
            rb.MovePosition(rb.position + new Vector2(0, missileSpeed));
        }
        if(rb.position.y > 10){
            gameObject.transform.position = new Vector3 (0, -6, 0);
            _btn_left_click.btn_l_cc = false;
        }
    }
}
```

---

## 不需要修改的部分

| 元件 | 理由 |
|------|------|
| `end_bar_g4.cs` | container 仍只有一個，不會 double-call Miss_score2 |
| `OnTriggerEnter2D` 邏輯 | `right_t`/`left_t` 判斷邏輯不變 |
| `hasFiredThisQuestion` | 已實作，不動 |
| item 分支（heart/laser/rocket） | spawnX 邏輯不動，items 無視覺偏移問題 |
| 速度計算、特效、QuestionLogger | 全部不動 |

---

## 狗狗能力（迪米）三階段相容性

迪米的動畫讓子彈從 Y=6 推到 Y≈8 再懸浮、緩降，飛彈追蹤設計需能處理全程：

| Phase | 時長 | 子彈行為 | 飛彈追蹤狀況 |
|-------|------|---------|------------|
| Phase 1 | 0.5s | DOTween 往上推 2 units（Y: 6→8，Ease.OutQuad） | 目標上升中，飛彈斜上追蹤。60 u/s >> DOTween 上推速度 4 u/s，**0.23s 內命中** ✅ |
| Phase 2 | 3.0s | 懸浮在 Y≈8 正弦波 bob（幅 0.1, 2Hz） | 飛彈直飛至 Y≈8，bob 幅度 0.1 units 遠小於飛彈速度，不影響命中 ✅ |
| Phase 3 | 1.5s | speed × 0.5f 緩降 | 目標以半速落下，飛彈輕易追上 ✅ |

**Y 邊界 > 10 對迪米的關鍵性**：Phase 1/2 期間目標位於 Y≈8。若邊界仍是 y>8，飛彈會在命中前被重置；改為 y>10 後才能追到懸浮中的子彈。

**`currentRightTarget` 無需特殊處理**：DOTween 操作 Transform 位置，不影響 GameObject 參考，飛彈持續追蹤同一個 Transform。

---

## 邊界情況

| 情況 | 行為 |
|------|------|
| 目標被提前 Destroy（烏龜重置等） | `target != null` 為 false → fallback 直線向上 |
| Item 波（心、雷射、red_rocket） | `currentRightTarget = null` → fallback 直線向上 |
| 飛彈飛出螢幕頂（未命中） | y > 10 重置，`btn_r_cc = false`，`hasFiredThisQuestion` 仍 true |
| 正確答案在右，玩家按左 | 飛彈 home 到左側，container 在右側 → 未碰 → container 落地 → Miss_score2 |
| 迪米 Phase 1 發射飛彈 | 飛彈斜上追蹤上升目標，約 0.23s 內命中 |
| 迪米 Phase 2 發射飛彈 | 飛彈飛至 Y≈8 命中懸浮目標（y>10 邊界確保不提早重置） |

---

## 驗收標準

1. 飛彈發射後視覺上斜向朝怪物飛去（非直線向上）
2. 命中時爆炸特效出現在怪物實際位置（X 對齊）
3. 正確答案時 score 正常累積，無鬼擊（不對齊就不算中）
4. 按錯方向 → 飛彈飛到錯誤側，打空，container 落地扣心
5. Item 波（heart/laser）：飛彈仍直線向上（target=null fallback）
6. 高等級（怪物速度 0.20）時，飛彈仍能追上（1.2f × 50Hz = 60 u/s，約 5× 怪物速度）
7. 命中爽快感：飛彈雷霆萬鈞，不是慢慢追趕
8. 迪米 Phase 1/2 期間發射飛彈，仍能追蹤並命中懸浮目標
