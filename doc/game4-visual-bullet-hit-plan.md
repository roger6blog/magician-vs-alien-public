# Plan: Game4 視覺碰撞對齊修正（Visual Bullet as Hit Target）

---

## Context

### 問題描述

前一版實作為了讓飛彈能物理接觸到正確答案，把 container 的生成 X 從 spawnX（≈0）改到了正確答案怪物的 X（±3）。這導致視覺流程改變（「怪物從旁邊冒出來」）且飛彈軌跡怪異。

### 正確架構

| 物件 | 職責 |
|------|------|
| Container（right_obj/left_obj）| 落下載體 + **碰撞體停用** |
| 視覺怪物（bullet_Right/bullet_Left）| 視覺計數 + **被飛彈打中判分** |

### Tech Lead 攔截的兩個 Bug

**Bug 1（Parent Collider Trap）：** Container 的 CircleCollider2D 在 X=0，飛彈也從 X=0 升起，瞬間觸發 → 鬼擊。Fix：停用 container 碰撞體。

**Bug 2（End Bar Fall-Through）：** 停用 container 碰撞體後，child 撞 end_bar；但 child 沒有 move2，原本的 `GetComponent<move2>()` 防呆會忽略 child → 怪物穿地板無限下落。Fix：改用 parent-destroy 邏輯。

**Tech Lead 觀察：** 按錯方向的飛彈仍應命中對面子彈並立即觸發答錯，不應依賴 end_bar 延遲判定。

---

## 關鍵設計：雙側子彈皆 tag，用 _score.ans 判對錯

| 子彈 | tag（固定）| 語意 |
|------|-----------|------|
| bullet_Right | `right_t` | 「我在右側」|
| bullet_Left | `left_t` | 「我在左側」|

hit_right/hit_left 的 OnTriggerEnter2D 改用 `_score.ans`（已為 public int）判斷是否答對：
- `ans == 1` → 右側是正確答案
- `ans == 2` → 左側是正確答案

---

## 修改計畫

### 一、score_g4.cs（`Assets/Scripts/score_g4.cs`）

#### 1-1. 移除 container 位置覆寫（~line 272–275）

刪除這 4 行，container 回到 spawnX（外星人正上方）：
```csharp
// 覆寫 container 位置：對齊正確答案側的視覺怪物，讓飛彈 homing 時物理接觸正確
enemyBulletRootPosition = (random_m2_r > random_m2_l)
    ? new Vector3(bullet_Right_Position.x, 6f, 0)
    : new Vector3(bullet_Left_Position.x, 6f, 0);
```

#### 1-2. parenting 之後：雙側子彈皆 tag + 停用 container 碰撞體

位置：~line 339（`bullet_Right.transform.parent = ...` 之後，monster branch 內）

```csharp
// 雙側子彈皆 tag（右側永遠 right_t，左側永遠 left_t）
// hit_right/hit_left 用 _score.ans 判斷是否答對
if (bullet_Right != null) bullet_Right.tag = "right_t";
if (bullet_Left  != null) bullet_Left.tag  = "left_t";

// Bug 1 修復：停用 container 碰撞體，避免飛彈在 X=0 誤撞
enemyBulletRoot.GetComponent<Collider2D>().enabled = false;
```

> 只在 monster branch（random_m < 6）執行，item 波的 bullet_Left 保留原本 tag。

---

### 二、hit_right.cs（`Assets/Scripts/hit_right.cs`）

**改寫 OnTriggerEnter2D 中的 right_t 和 left_t 分支：**

**right_t 分支（飛彈命中右側子彈）：**
```csharp
if (col.gameObject.tag == "right_t")
{
    ShowExplosion(col.transform);
    if (_s4_start_btn.start_var && _btn_right_click.btn_r_cc)
    {
        Transform containerRoot = col.transform.parent;

        if (_score.ans == 1) {
            // 正確：右側是答案，玩家選右
            music.Play();
            _score.myscore = _score.myscore + 1;
            _score.level_set();
            _score.RecordAnswer(true, 1);
            timer_f = 0f; timer_i = 0; time_start = true;
            rr = Random.value;
            random = Mathf.RoundToInt(2 * rr);
            if (random == 1)      good.transform.position = new Vector3(0, 0.5f, 0);
            else if (random == 2) nice.transform.position = new Vector3(0, 0.5f, 0);
            else                  right.transform.position = new Vector3(0, 0.5f, 0);
        } else {
            // 答錯：左側是答案，玩家選右
            _score.RecordAnswer(false, 1);
            miss_function();
        }

        gameObject.transform.position = new Vector3(0, -6, 0);
        _btn_right_click.btn_r_cc = false;
        Destroy(containerRoot != null ? containerRoot.gameObject : col.gameObject);
    }
}
```

**left_t 分支（飛彈誤入左側子彈，不應發生但保留安全網）：**
```csharp
if (col.gameObject.tag == "left_t")
{
    ShowExplosion(col.transform);
    if (_s4_start_btn.start_var)
    {
        _score.RecordAnswer(false, 1);
        Transform containerRoot = col.transform.parent;
        Destroy(containerRoot != null ? containerRoot.gameObject : col.gameObject);
        miss_function();
        gameObject.transform.position = new Vector3(0, -6, 0);
        _btn_right_click.btn_r_cc = false;
    }
}
```

---

### 三、hit_left.cs（`Assets/Scripts/hit_left.cs`）

對稱修改：

**left_t 分支（飛彈命中左側子彈）：**
- `_score.ans == 2` → 正確；else → 答錯
- `_score.RecordAnswer(true/false, 2)`
- `Destroy(col.transform.parent != null ? col.transform.parent.gameObject : col.gameObject)`

**right_t 分支（安全網）：**
- 同 hit_right 的 left_t 安全網，RecordAnswer(false, 2)

---

### 四、end_bar_g4.cs（`Assets/Scripts/end_bar_g4.cs`）

**改用 parent-destroy 邏輯（Bug 2 修復，丟掉 move2 判斷）：**

```csharp
if (col.gameObject.tag == "right_t" || col.gameObject.tag == "left_t")
{
    _score.Miss_score2();
    // child（視覺怪物）撞 end_bar → 刪 parent（container）連同所有 child
    Transform containerRoot = col.transform.parent;
    Destroy(containerRoot != null ? containerRoot.gameObject : col.gameObject);
    // （原本的 ResetPlayerBullets 等邏輯保持不變）
}
```

---

## 碰撞路徑總覽

| 玩家動作 | 飛彈目標 | 命中 tag | _score.ans | 結果 |
|---------|---------|---------|-----------|------|
| 按右（正確）| bullet_Right（right_t）| right_t | 1 | 得分 |
| 按右（答錯）| bullet_Right（right_t）| right_t | 2 | miss_function() 立即扣心 |
| 按左（正確）| bullet_Left（left_t）| left_t | 2 | 得分 |
| 按左（答錯）| bullet_Left（left_t）| left_t | 1 | miss_function() 立即扣心 |
| 沒射出（超時）| — | — | — | container+child 落地 → end_bar Miss_score2 |

---

## 不需要修改的部分

| 元件 | 理由 |
|------|------|
| FixedUpdate homing | currentRightTarget/currentLeftTarget 已指向視覺怪物，不變 |
| hasFiredThisQuestion | 已實作，不動 |
| 飛彈旋轉邏輯 | 已實作，不動 |
| item 分支（heart/laser/rocket）| 不進入 monster branch，tag 不被覆寫，不動 |
| ShowExplosion | 已實作，保留呼叫 |

---

## 驗收標準

1. Container 從外星人正上方落下，視覺流程恢復
2. 飛彈斜向飛向自己側的視覺怪物，命中時爆炸位置在視覺怪物 X 上
3. 正確命中 → 得分 + container 整體清除，不落地
4. 按錯 → 飛彈命中對面子彈 → 立即扣心 + container 整體清除
5. 超時（未射）→ child 落 end_bar → Miss_score2（無雙重扣心）
6. Item 波行為不受影響
