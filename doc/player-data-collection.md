# 玩家數據收集清單 v1.0

分析來源：Unity 客戶端（C#）、PHP 後端 API、Player Data Viewer（Vue.js）
排除 .md 檔案，僅依原始碼推導。

---

## 一、學術用途數據

### 1. 遊戲題目作答記錄（`game_question_log`）

每一題都記錄，遊戲結束時由 `QuestionLogger.cs` 批次上傳至 `save_question_log.php`。

| 欄位 | 內容 |
|------|------|
| `session_id` | 對應 `training_history.save_id`，可跨表聯合分析 |
| `game_type` | 遊戲類型（1=數字比較, 3=房子計數, 21/22/23=選擇題, 100=AF 測驗） |
| `question_index` | 題目順序（0-based） |
| `question_text` | 題目文字（如 "12 + 7 = ?" 或 "左:3 右:5"） |
| `correct_answer` | 正確答案 |
| `user_answer` | 玩家答案（null = 超時未作答） |
| `is_correct` | 對/錯（0/1） |
| `response_time_ms` | 反應時間（毫秒）——認知測量核心指標；null = 超時 |
| `player_level` | 作答當下的遊戲難度等級 |
| `player_score` | 作答當下的累計分數 |
| `ability_used` | 使用的能力 ID（null = 未使用） |
| `excluded` | 資料排除標記（0/1），供 IRB 資料品質控制 |
| `excluded_reason` | 排除原因（見下表） |
| `platform` | 平台（webgl / android / ios / editor） |
| `created_at` | 作答時間戳 |

**`excluded_reason` 對應表：**

| 排除原因 | 觸發情境 |
|---------|---------|
| `modifier_effect` | Modifier 影響 |
| `cat_simplify` | 貓咪能力簡化題目 |
| `turtle_pause` | 烏龜能力暫停時間 |
| `dog_slow` | 狗狗能力減速 |
| `owl_reveal` | 貓頭鷹顯示提示 |
| `heal` | 補血操作 |
| `pause` | 暫停遊戲 |
| `skip` | 跳過題目 |
| `slow` | 減速效果 |
| `reveal` | 直接顯示答案 |

---

### 2. 算術流暢度測驗（AF Test）

時程設計：baseline → Day 42+ posttest → followup
API：`af_test_manager.php` → 資料庫：`af_test_results`、`af_test_state`

| 欄位 | 內容 |
|------|------|
| `test_type` | baseline / posttest / followup |
| `test_date` | 完成時間戳 |
| `total_attempted` | 嘗試題數 |
| `total_correct` | 答對題數 |
| `total_duration_ms` | 全程耗時（毫秒） |
| `questions_json` | 每題詳細：operand1、operand2、operator、answer |
| `platform` | 平台 |
| `login_count` | 首次登入起的累計天數（用於判斷 posttest 時機） |
| `baseline_status` / `posttest_status` | pending / skipped / completed（狀態機追蹤） |

---

### 3. 問卷

#### 3a. PPTQ-C 人格問卷

API：`save_questionnaire.php`（Type=`PPTQ_C`）
資料庫：`questionnaire_responses`、`user.personality_scores`

| 欄位 | 內容 |
|------|------|
| `answers_json` | 15 題答案陣列（每題含 score + 反應時間 ms） |
| `personality_scores` | Big Five 五維度分數：E（外向）、N（神經質）、O（開放）、C（盡責）、A（親和） |
| `animal_type` | 最高分維度對應的動物分類（E→lion, N→cat, O→owl, C→turtle, A→dog） |
| `pptqc_platform` | 作答平台 |
| `pptqc_completed_at` | 完成時間 |

#### 3b. 數學態度問卷（MathAttitude）

API：`save_questionnaire.php`（Type=`MathAttitude_Single`）
資料庫：`questionnaire_single_responses`、`user.math_attitude_scores`
設計：縱向追蹤，每次遊玩後各作答一題（輪替）

| 欄位 | 內容 |
|------|------|
| `question_index` | 0=usefulness, 1=importance, 2=enjoyment, 3=anxiety, 4=self_efficacy |
| `answer_value` | Likert 量表值 |
| `games_played` | 作答當下已遊玩局數 |
| `timestamp` | 作答時間 |
| `math_attitude_scores` | 各面向平均分（attitude_avg, enjoyment_avg, anxiety_avg, self_efficacy_avg） |

---

### 4. 遊戲績效

API：`save_data.php` → 資料庫：`training_history`、`daily_history`

| 欄位 | 內容 |
|------|------|
| `score` | 本局得分 |
| `training_time` | 遊玩時長（秒） |
| `game_type` | 遊戲類型 |
| `lv` | 遊戲難度等級 |
| `game1/2/3_starCount` | 每日各遊戲星數（0~5） |
| `game1/2/3_totalTime` | 每日各遊戲累計時長 |
| `WScore_game1/2/3` | 加權分數（依難度等級乘數計算，轉換為 1~5 星） |
| `game1/2/3_daily_die` | 每日死亡次數（難度分析） |

**加權分數乘數對照：**

| 遊戲 | 等級條件 | 乘數 |
|------|---------|------|
| Game 1 | 難度 5/4/3/2/1 | 3.0 / 2.5 / 2.0 / 1.5 / 1.0 |
| Game 2 | 速度等級 3/2/1/0 | 2.5 / 2.0 / 1.5 / 1.0 |
| Game 3 | 正確數 15+/11-15/7-10/3-6/0-2 | 3.0 / 2.5 / 2.0 / 1.5 / 1.0 |

**星數門檻：** 5星≥120分、4星≥80分、3星≥45分、2星≥20分、1星≥1分

---

### 5. 使用者裝置環境調查

API：`save_device_session.php` → 資料庫：`device_session_logs`
觸發時機：登入成功後立即回報，每 60 秒心跳更新

| 欄位 | 內容 |
|------|------|
| `platform` | 平台（webgl / android / ios / editor）——控制平台差異的共變數 |
| `screen_width` / `screen_height` | 螢幕解析度（像素） |
| `os_info` | 作業系統資訊（最多 100 字） |
| `device_model` | 裝置型號（最多 100 字） |
| `total_duration_sec` | 遊玩時長（伺服器計算，上限 7200 秒，避免 AFK 毒化數據） |
| `fg_bg_count` | 前後台切換次數（偵測分心行為） |
| `ip_masked` | 匿名化 IP（僅保留前 3 段，符合台灣個資法 §19） |

---

### 6. 人口統計

API：`save_user_profile.php` → 資料庫：`user`

| 欄位 | 內容 |
|------|------|
| `gender` | 性別（male / female / other） |
| `birthday` | 生日（YYYY-MM-DD） |

---

## 二、非學術用途數據（遊戲運營）

### 1. 帳號與驗證

資料庫：`user`、`sessions`

| 欄位 | 內容 |
|------|------|
| `name` | 帳號（登入用） |
| `password` | 密碼（加密儲存） |
| `session_token` | 會話令牌（7 天有效） |
| `ip_address` | 完整 IP（sessions 表，非匿名） |
| `user_agent` | 瀏覽器 / 客戶端資訊 |
| `expires_at` | 令牌過期時間 |

### 2. 玩家個人化

| 欄位 | 內容 |
|------|------|
| `nickname` | 玩家暱稱 |
| `magic_animal` | 選擇的魔數動物（lion / dog / turtle / cat / owl） |

### 3. 虛擬貨幣與道具

資料庫：`user`、`user_abilities_v2`、`daily_locked_ability`

| 欄位 | 內容 |
|------|------|
| `coins` / `crystals` | 金幣 / 結晶餘額 |
| `abilities_json` | 各能力的 level、daily_used、total_used、last_used_date |
| `equipped_ability_id` | 當前裝備的能力 |
| `locked_ability_id` | 每日使用鎖定的能力槽位 |

### 4. 遊戲進度（運營用）

| 欄位 | 內容 |
|------|------|
| `game1/2/3_daily_time` | 每日剩餘遊戲時間（秒） |
| `game1/2/3_total_star` / `best_star` | 累計與最佳星數（用於排行榜） |
| `ComboMax_1/2/3` | 各遊戲最高連擊數 |

### 5. 留存系統

資料庫：`login_streak`（`login_streak.php`）

| 欄位 | 內容 |
|------|------|
| `consecutive_days` | 連續登入天數（驅動 15 日獎勵） |
| `last_login_date` | 最後登入日期 |
| `is_first_today` | 今日首次登入旗標 |

### 6. 排行榜

API：`rank_data_json.php`，回傳各遊戲的分數排名，用於玩家間競爭功能。

---

## 三、總覽對照表

| 類型 | 數據 | API / 資料庫 | 用途說明 |
|------|------|------------|---------|
| **學術** | 題目作答（正確率、反應時間） | `save_question_log.php` / `game_question_log` | 認知能力測量 |
| **學術** | `excluded` 排除標記 | `game_question_log` | IRB 資料品質管控 |
| **學術** | AF 測驗（baseline/posttest） | `af_test_manager.php` / `af_test_results` | 學習成效評估 |
| **學術** | PPTQ-C（Big Five + 反應時間） | `save_questionnaire.php` / `user.personality_scores` | 人格與學習風格研究 |
| **學術** | 數學態度問卷（縱向追蹤） | `save_questionnaire.php` / `questionnaire_single_responses` | 情意因素變化研究 |
| **學術** | 遊戲績效（分數、時長、星數） | `save_data.php` / `training_history` | 學習表現追蹤 |
| **學術** | 裝置環境（OS、螢幕、裝置型號） | `save_device_session.php` / `device_session_logs` | 裝置環境共變數控制 |
| **學術** | 遊玩時長、前後台切換 | `save_device_session.php` / `device_session_logs` | 投入度與分心行為分析 |
| **學術** | 性別、生日 | `save_user_profile.php` / `user` | 人口統計共變數 |
| **非學術** | 帳號 / 密碼 / Session Token | `login_v2.php` / `sessions` | 身份驗證安全 |
| **非學術** | 金幣 / 結晶 / 道具能力 | `coin_manager.php` 等 / `user_abilities_v2` | 遊戲內經濟系統 |
| **非學術** | 連續登入天數 | `login_streak.php` | 留存獎勵機制 |
| **非學術** | 排行榜分數 | `rank_data_json.php` | 社交競爭功能 |
| **非學術** | 暱稱 / 動物選擇 | `save_user_profile.php` / `save_magic_animal.php` | 玩家個人化 |
| **非學術** | 每日剩餘時間 / 最高連擊數 | `game_time_limit.php` / `update_max_combo.php` | 遊戲進度管理 |
