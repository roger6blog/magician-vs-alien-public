# 遊戲資料系統設計計畫

本文件整理遊戲本身需要記錄的資料，與研究數據系統（`research-data-system-plan.md`）互補但分開管理。

---

## 目錄

1. [資料儲存現況總覽](#資料儲存現況總覽)
2. [玩家基本資料](#一玩家基本資料)
3. [遊戲進度系統](#二遊戲進度系統)
4. [經濟系統](#三經濟系統)
5. [能力系統](#四能力系統)
6. [登入與活躍度](#五登入與活躍度)
7. [設定偏好](#六設定偏好)
8. [魔數動物系統](#七魔數動物系統)
9. [資料庫結構建議](#資料庫結構建議)
10. [實作優先級](#實作優先級)

---

## 資料儲存現況總覽

### 儲存位置分類

| 儲存位置 | 說明 | 特性 |
|---------|------|------|
| **PHP 後端（MySQL）** | 伺服器資料庫 | 跨裝置同步、永久保存 |
| **PlayerPrefs（本地）** | Unity 本地儲存 | 僅限單一裝置、可能遺失 |
| **記憶體** | 遊戲執行時暫存 | 關閉即消失 |

### 現況評估

| 系統 | 後端同步 | 狀態 | 備註 |
|-----|---------|------|------|
| 玩家基本資料 | ✅ | 大致完整 | 缺年級、學校欄位 |
| 遊戲分數 | ✅ | 完整 | - |
| 加權計分/星星 | ❌ | **只存本地** | 需要修復 |
| 金幣/結晶 | ✅ | 完整 | - |
| 能力系統 | ✅ | 完整 | 缺使用詳情 |
| 連續登入 | ✅ | 完整 | - |
| 設定偏好 | ❌ | 只存本地 | 可選同步 |
| 魔數動物 | ❓ | 待確認 | 需要實作 |

---

## 一、玩家基本資料

### 目前已有欄位

| 欄位 | 型別 | 說明 | 儲存位置 | 來源腳本 |
|-----|------|------|---------|---------|
| `user_name` | VARCHAR(100) | 帳號（登入用） | PHP user 表 | login_save.cs |
| `nickname` | VARCHAR(50) | 暱稱（顯示用） | PHP user 表 | UserProfileController.cs |
| `gender` | VARCHAR(10) | 性別（male/female/other） | PHP user 表 | UserProfileController.cs |
| `birthday` | DATE | 生日（YYYY-MM-DD） | PHP user 表 | UserProfileController.cs |

### 建議新增欄位

| 欄位 | 型別 | 說明 | 用途 |
|-----|------|------|------|
| `grade` | TINYINT | 年級（1-6 代表小一到小六） | 研究分群、難度適配 |
| `birth_year` | INT | 出生年份 | 快速查詢年齡（從 birthday 推算但獨立存） |
| `school_code` | VARCHAR(20) | 學校代碼 | 跨校研究比較 |
| `cohort` | VARCHAR(30) | 研究批次標籤（如 "2026S1"） | 批次管理、世代追蹤 |
| `created_at` | DATETIME | 帳號建立時間 | 玩家生命週期分析 |

### SQL 變更

```sql
ALTER TABLE user ADD COLUMN grade TINYINT DEFAULT NULL COMMENT '年級（1=小一, 2=小二, ...6=小六）';
ALTER TABLE user ADD COLUMN birth_year INT DEFAULT NULL COMMENT '出生年份';
ALTER TABLE user ADD COLUMN school_code VARCHAR(20) DEFAULT NULL COMMENT '學校代碼';
ALTER TABLE user ADD COLUMN cohort VARCHAR(30) DEFAULT NULL COMMENT '研究批次標籤';
ALTER TABLE user ADD COLUMN created_at DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT '帳號建立時間';
```

### 資料收集時機

- **nickname / gender / birthday**：basicinfo 場景（首次登入或設定頁面）
- **grade / school_code / cohort**：可在 basicinfo 場景新增，或由研究人員後台設定

---

## 二、遊戲進度系統

### 目前已有欄位

| 欄位 | 說明 | 儲存位置 | 來源腳本 |
|-----|------|---------|---------|
| `score` | 單場分數 | PHP training_history | DataManager.cs |
| `game_type` | 遊戲類型（1/2/3） | PHP training_history | DataManager.cs |
| `lv` | 達到的關卡 | PHP training_history | DataManager.cs |
| `training_time` | 訓練時間（秒） | PHP training_history | DataManager.cs |
| `game1/2/3_daily_time` | 當日遊玩時間 | PHP user 表 | 每秒自動存檔 |
| `game1/2/3_total_star` | 當日累計星星 | PHP user 表 | - |
| `game1/2/3_best_star` | 當日最佳星星 | PHP user 表 | - |
| `game1/2/3_daily_die` | 當日死亡次數 | PHP user 表 | - |

### ⚠️ 問題：加權計分只存本地

**WeightedScoreTracker.cs** 的加權分數是目前最精確的表現指標，但只存在 PlayerPrefs：

```csharp
// 目前的儲存方式（只存本地）
PlayerPrefs.SetFloat("WScore_game1", currentScore);
PlayerPrefs.SetString("WScore_date_game1", today);
```

**加權計分邏輯：**
```
最終分 = correctCount × difficultyMultiplier
星星門檻：
  5★ = 120+
  4★ = 80-119
  3★ = 45-79
  2★ = 20-44
  1★ = 1-19
```

### 建議：同步加權計分到後端

新增 `daily_weighted_score` 表或整合到 `game_session` 表：

```sql
CREATE TABLE daily_weighted_score (
    id              BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id         VARCHAR(100) NOT NULL,
    game_type       TINYINT NOT NULL,
    play_date       DATE NOT NULL,
    weighted_score  FLOAT NOT NULL,
    star_count      TINYINT NOT NULL,
    updated_at      DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    UNIQUE KEY idx_user_game_date (user_id, game_type, play_date),
    INDEX idx_user (user_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 修改 WeightedScoreTracker.cs

```csharp
// 新增：每次更新分數時同步到後端
public void SyncToServer(string gameName, float score, int stars)
{
    StartCoroutine(UploadWeightedScore(gameName, score, stars));
}
```

---

## 三、經濟系統

### 目前已有欄位

| 欄位 | 說明 | 儲存位置 | 來源腳本 |
|-----|------|---------|---------|
| `coins` | 金幣餘額 | PHP user 表 | CoinManager.cs |
| `crystals` | 結晶餘額 | PHP user 表 | CrystalManager.cs |

### 金幣系統

**CoinManager.cs** 提供的操作：
- `AddCoins(amount)` - 增加金幣
- `SpendCoins(amount)` - 花費金幣
- `GetBalance()` - 查詢餘額

**API 端點：** `coin_manager.php`

### 結晶系統

**MagicCrystalCalculator.cs** 的計算公式：

```
基礎結晶（依死亡次數）：
  0-1 死 = 5 結晶
  2-4 死 = 4 結晶
  5-8 死 = 3 結晶
  9-12 死 = 2 結晶
  13+ 死 = 1 結晶

時間係數（依有效遊玩時間）：
  ≥8 分鐘 = ×1.0
  6-8 分鐘 = ×0.7
  4-6 分鐘 = ×0.4
  <4 分鐘 = ×0（不給結晶）

最終結晶 = floor(基礎結晶 × 時間係數)
```

**API 端點：** `crystal_manager.php`

### 建議新增：交易紀錄表

用於追蹤金幣/結晶的來源和去向：

```sql
CREATE TABLE currency_transaction (
    id              BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id         VARCHAR(100) NOT NULL,
    currency_type   ENUM('coins', 'crystals') NOT NULL,
    amount          INT NOT NULL,                       -- 正數=獲得，負數=花費
    balance_after   INT NOT NULL,                       -- 交易後餘額
    source          VARCHAR(50) NOT NULL,               -- 來源/用途
    -- 來源類型：
    -- 'game_reward' - 遊戲獎勵
    -- 'daily_login' - 每日登入
    -- 'ability_purchase' - 購買能力
    -- 'ability_upgrade' - 升級能力
    -- 'admin_adjust' - 管理員調整
    reference_id    VARCHAR(100) DEFAULT NULL,          -- 關聯 ID（如 session_id）
    created_at      DATETIME DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_user (user_id),
    INDEX idx_created (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

---

## 四、能力系統

### 目前已有欄位

| 欄位 | 說明 | 儲存位置 | 來源腳本 |
|-----|------|---------|---------|
| `ability_id` | 能力編號（1-7） | PHP ability 表 | AbilityManager.cs |
| `level` | 能力等級（1-3） | PHP ability 表 | AbilityManager.cs |
| `daily_used` | 今日使用次數 | PHP ability 表 | AbilityManager.cs |
| `max_daily_uses` | 每日使用上限 | PHP ability 表 | AbilityManager.cs |

### 能力列表

| ID | 名稱 | 英文名 | 效果 | 購買成本 | 升級成本 | 最高等級 |
|----|-----|-------|------|---------|---------|---------|
| 1 | 護心盾牌 | HeartShield | 恢復一顆愛心 | 100 金幣 | 100 金幣 | 3 |
| 2 | 時間魔法棒 | TimeWand | 暫停怪物 5 秒 | 100 金幣 | 100 金幣 | 3 |
| 3 | 魔法眼鏡 | MagicGlasses | 顯示正確答案 | 100 金幣 | 100 金幣 | 3 |
| 4 | 脈衝引擎 | Hourglass | 加速時間流逝 | 100 金幣 | 100 金幣 | 3 |
| 5 | 流星閃避 | MeteorDodge | 跳過當前題目 | 100 金幣 | 100 金幣 | 3 |
| 6 | 幸運星星 | LuckyStar | 當日星星 ×2 | 100 金幣 | - | 1 |
| 7 | 緩時冰晶 | IceCrystal | 減緩時間流逝 | 100 金幣 | 100 金幣 | 3 |

### 能力分類

| 類型 | 能力 | 對研究數據的影響 |
|-----|------|----------------|
| **主動-汙染作答** | MagicGlasses, MeteorDodge | 該題不記入 trial_log |
| **主動-影響時間** | TimeWand | 該題不記入 trial_log |
| **主動-不影響** | HeartShield | 正常記錄 |
| **被動** | Hourglass, IceCrystal, LuckyStar | 記錄啟用狀態，不排除個別題目 |

### 建議新增：累計使用統計

```sql
ALTER TABLE user_abilities ADD COLUMN total_used INT DEFAULT 0 COMMENT '累計使用次數';
ALTER TABLE user_abilities ADD COLUMN last_used_at DATETIME DEFAULT NULL COMMENT '最後使用時間';
```

### 能力使用詳情（已在 research-data-system-plan.md 規劃）

詳細的使用時機記錄在 `ability_usage_log` 表，見研究數據系統文件。

---

## 五、登入與活躍度

### 目前已有欄位

| 欄位 | 說明 | 儲存位置 | 來源腳本 |
|-----|------|---------|---------|
| `consecutive_days` | 連續登入天數 | PHP user 表 | DataManager.cs |
| `last_login_date` | 最後登入日期 | PHP user 表 | login_streak.php |
| `is_first_today` | 今日是否首次登入 | API 回傳 | LoginStreakManager.cs |

### 登入獎勵機制

- 每 15 天為一個周期循環
- UI 顯示 "X/15"
- 連續登入可獲得額外獎勵（具體獎勵待確認）

### 建議新增欄位

```sql
ALTER TABLE user ADD COLUMN total_login_days INT DEFAULT 0 COMMENT '總登入天數';
ALTER TABLE user ADD COLUMN total_play_time_seconds INT DEFAULT 0 COMMENT '總遊玩時間（秒）';
ALTER TABLE user ADD COLUMN max_consecutive_days INT DEFAULT 0 COMMENT '歷史最高連續登入';
ALTER TABLE user ADD COLUMN first_login_date DATE DEFAULT NULL COMMENT '首次登入日期';
```

### 建議新增：登入歷史表

用於詳細追蹤玩家活躍度：

```sql
CREATE TABLE login_history (
    id              BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id         VARCHAR(100) NOT NULL,
    login_date      DATE NOT NULL,
    login_time      TIME NOT NULL,
    device_info     VARCHAR(200) DEFAULT NULL,          -- 裝置資訊（可選）

    UNIQUE KEY idx_user_date (user_id, login_date),
    INDEX idx_date (login_date)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

---

## 六、設定偏好

### 目前已有（僅本地）

| 設定 | PlayerPrefs Key | 預設值 | 來源腳本 |
|-----|----------------|-------|---------|
| 主音量 | `MathGame_MasterVolume_v1` | 1.0 | VolumeManager.cs |
| 音樂選擇 | `SelectedMusic` | 0 | MusicManager.cs |

### 音樂選項

| 索引 | 名稱 |
|-----|------|
| 0 | 無音樂 |
| 1 | Starry Night |
| 2 | BGM 2 |
| 3 | Bouncy Tune |
| 4 | Happy Adventure |
| 5 | Magic Town |

### 建議：同步設定到後端（可選）

如果希望跨裝置保持設定一致：

```sql
ALTER TABLE user ADD COLUMN master_volume FLOAT DEFAULT 1.0 COMMENT '主音量（0-1）';
ALTER TABLE user ADD COLUMN selected_music TINYINT DEFAULT 0 COMMENT '音樂選擇（0-5）';
```

**同步時機：**
- 設定變更時上傳
- 登入時下載並覆蓋本地

**注意：** 這是可選功能，優先級較低。

---

## 七、魔數動物系統

### 目前狀態：待確認

根據程式碼探索，魔數動物系統的實作狀態需要進一步確認：

- 是否有角色選擇功能？
- 是否有多個可解鎖的動物？
- 動物是否有屬性差異？

### 建議資料結構（如有此系統）

```sql
-- 動物定義表
CREATE TABLE animals (
    animal_id       INT PRIMARY KEY,
    animal_name     VARCHAR(50) NOT NULL,
    display_name    VARCHAR(50) NOT NULL,               -- 顯示名稱
    unlock_cost     INT DEFAULT 0,                      -- 解鎖成本（金幣）
    description     TEXT DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 玩家擁有的動物
CREATE TABLE user_animals (
    id              BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id         VARCHAR(100) NOT NULL,
    animal_id       INT NOT NULL,
    unlocked_at     DATETIME DEFAULT CURRENT_TIMESTAMP,
    is_selected     TINYINT DEFAULT 0,                  -- 是否為當前選擇

    UNIQUE KEY idx_user_animal (user_id, animal_id),
    INDEX idx_user (user_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- user 表新增
ALTER TABLE user ADD COLUMN selected_animal_id INT DEFAULT 1 COMMENT '當前選擇的動物';
```

---

## 資料庫結構建議

### 現有表格整理

| 表名 | 用途 | 狀態 |
|-----|------|------|
| `user` | 玩家帳戶與基本資料 | ✅ 需擴充欄位 |
| `training_history` | 遊戲分數紀錄 | ✅ 完整 |
| `user_abilities` | 能力擁有與使用 | ✅ 需加累計欄位 |
| `questionnaire_responses` | 問卷回答 | ✅ 完整 |
| `af_test_results` | 算術流暢性測驗 | ✅ 完整 |

### 建議新增表格

| 表名 | 用途 | 優先級 |
|-----|------|-------|
| `daily_weighted_score` | 每日加權分數/星星 | 🔴 高 |
| `currency_transaction` | 金幣/結晶交易紀錄 | 🟡 中 |
| `login_history` | 登入歷史 | 🟡 中 |
| `user_animals` | 玩家動物（如有） | 🟢 低 |

### 完整 SQL 變更腳本

```sql
-- =====================================================
-- 遊戲資料系統 - 資料庫變更腳本
-- =====================================================

-- 1. user 表擴充
ALTER TABLE user ADD COLUMN grade TINYINT DEFAULT NULL COMMENT '年級（1-6）';
ALTER TABLE user ADD COLUMN birth_year INT DEFAULT NULL COMMENT '出生年份';
ALTER TABLE user ADD COLUMN school_code VARCHAR(20) DEFAULT NULL COMMENT '學校代碼';
ALTER TABLE user ADD COLUMN cohort VARCHAR(30) DEFAULT NULL COMMENT '研究批次';
ALTER TABLE user ADD COLUMN created_at DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT '帳號建立時間';
ALTER TABLE user ADD COLUMN total_login_days INT DEFAULT 0 COMMENT '總登入天數';
ALTER TABLE user ADD COLUMN total_play_time_seconds INT DEFAULT 0 COMMENT '總遊玩時間';
ALTER TABLE user ADD COLUMN max_consecutive_days INT DEFAULT 0 COMMENT '歷史最高連續登入';
ALTER TABLE user ADD COLUMN first_login_date DATE DEFAULT NULL COMMENT '首次登入日期';
ALTER TABLE user ADD COLUMN master_volume FLOAT DEFAULT 1.0 COMMENT '主音量';
ALTER TABLE user ADD COLUMN selected_music TINYINT DEFAULT 0 COMMENT '音樂選擇';
ALTER TABLE user ADD COLUMN selected_animal_id INT DEFAULT 1 COMMENT '選擇的動物';

-- 2. 能力表擴充
ALTER TABLE user_abilities ADD COLUMN total_used INT DEFAULT 0 COMMENT '累計使用次數';
ALTER TABLE user_abilities ADD COLUMN last_used_at DATETIME DEFAULT NULL COMMENT '最後使用時間';

-- 3. 每日加權分數表
CREATE TABLE IF NOT EXISTS daily_weighted_score (
    id              BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id         VARCHAR(100) NOT NULL,
    game_type       TINYINT NOT NULL,
    play_date       DATE NOT NULL,
    weighted_score  FLOAT NOT NULL,
    star_count      TINYINT NOT NULL,
    updated_at      DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    UNIQUE KEY idx_user_game_date (user_id, game_type, play_date),
    INDEX idx_user (user_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 4. 貨幣交易紀錄表
CREATE TABLE IF NOT EXISTS currency_transaction (
    id              BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id         VARCHAR(100) NOT NULL,
    currency_type   ENUM('coins', 'crystals') NOT NULL,
    amount          INT NOT NULL,
    balance_after   INT NOT NULL,
    source          VARCHAR(50) NOT NULL,
    reference_id    VARCHAR(100) DEFAULT NULL,
    created_at      DATETIME DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_user (user_id),
    INDEX idx_created (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 5. 登入歷史表
CREATE TABLE IF NOT EXISTS login_history (
    id              BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id         VARCHAR(100) NOT NULL,
    login_date      DATE NOT NULL,
    login_time      TIME NOT NULL,
    device_info     VARCHAR(200) DEFAULT NULL,

    UNIQUE KEY idx_user_date (user_id, login_date),
    INDEX idx_date (login_date)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

---

## 實作優先級

### 🔴 高優先級（立即修復）

| 項目 | 說明 | 影響 |
|-----|------|------|
| **加權計分上傳** | WeightedScoreTracker 同步到後端 | 目前最精確的表現數據只存本地 |
| **年級欄位** | user 表新增 grade | 研究分析必需 |
| **birth_year 欄位** | 從 birthday 推算並獨立存 | 方便查詢 |

### 🟡 中優先級（月內完成）

| 項目 | 說明 | 影響 |
|-----|------|------|
| **累計統計欄位** | total_login_days, total_play_time 等 | 長期玩家分析 |
| **能力累計使用** | total_used, last_used_at | 能力效益分析 |
| **貨幣交易紀錄** | currency_transaction 表 | 經濟系統追蹤 |

### 🟢 低優先級（之後再做）

| 項目 | 說明 | 影響 |
|-----|------|------|
| **設定同步** | 音量、音樂跨裝置同步 | 體驗優化 |
| **登入歷史** | 詳細登入紀錄 | 進階分析 |
| **魔數動物系統** | 待確認是否需要 | 遊戲功能 |

---

## 與研究數據系統的關係

本文件（遊戲資料）與 `research-data-system-plan.md`（研究數據）的分工：

| 資料類型 | 負責文件 | 主要用途 |
|---------|---------|---------|
| 玩家基本資料 | 本文件 | 遊戲功能、帳戶管理 |
| 經濟系統 | 本文件 | 遊戲功能 |
| 能力擁有/等級 | 本文件 | 遊戲功能 |
| 設定偏好 | 本文件 | 遊戲功能 |
| **逐題 trial_log** | research-data-system-plan.md | 研究分析 |
| **會話 game_session** | research-data-system-plan.md | 研究分析 |
| **能力使用詳情** | research-data-system-plan.md | 研究分析 |

共用欄位（兩邊都會用到）：
- `user_id` - 關聯玩家
- `game_type` - 遊戲類型
- `grade` / `cohort` - 研究分群

---

## 待確認事項

1. **魔數動物系統**：是否已實作？如有，需要記錄哪些資料？
2. **登入獎勵詳情**：連續登入的具體獎勵內容？
3. **成就系統**：是否有計畫實作成就/徽章系統？
4. **排行榜**：是否需要跨玩家的排行榜功能？

---

## 附錄：相關程式檔案

| 檔案 | 用途 |
|-----|------|
| `DataManager.cs` | 主要的資料上傳管理 |
| `CoinManager.cs` | 金幣系統 |
| `CrystalManager.cs` | 結晶系統 |
| `AbilityManager.cs` | 能力系統 |
| `WeightedScoreTracker.cs` | 加權計分（需修改） |
| `LoginStreakManager.cs` | 連續登入 |
| `UserProfileController.cs` | 基本資料輸入 |
| `VolumeManager.cs` | 音量設定 |
| `MusicManager.cs` | 音樂選擇 |
