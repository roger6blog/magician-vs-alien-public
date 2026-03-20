# 後端研究數據分析系統設計計畫

## 研究基礎：Ng et al. (2022) 論文

> **Ng, C.-T., Chen, Y.-H., Wu, C.-J., & Chang, T.-T. (2022).** Evaluation of math anxiety and its remediation through a digital training program in mathematics for first and second graders. *Brain and Behavior*, 12, e2557. DOI: [10.1002/brb3.2557](https://doi.org/10.1002/brb3.2557)

本遊戲（Igo Invasion）的研究論文。以下整理論文中與數據系統設計直接相關的要點。

### 研究設計摘要

| 項目 | 內容 |
|------|------|
| 對象 | 小一小二（6-8 歲），N=77（高強度 n=40，低強度 n=37） |
| 訓練時長 | 每天 30 分鐘（每模組 10 分鐘），持續 4-6 週 |
| 強度分組 | 高強度 >6 小時（M=10.35h），低強度 <2 小時（M=0.35h） |
| 前後測 | 數學焦慮（CMAQ 8 題 5 點量表）、數學成就（BMCST）、工作記憶（WISC-IV DS） |
| 核心發現 | 6 週訓練降低數學焦慮（尤其高焦慮兒童 r=-.51）、提升數學成就和工作記憶 |

### 三個模組與遊戲的對應關係

| 論文模組 | 論文描述 | 對應遊戲 | 訓練的認知能力 |
|---------|---------|---------|--------------|
| Module 1 | Star cluster numerical comparison — 比較兩組星星的數量 | **Game1 / Game2** | 數量比較（subitizing → counting），ANS 精確度 |
| Module 2 | Basic arithmetic | **算術流暢性前後測**（`af_test_results`） | 算術流暢性——非遊戲模組，而是以加減運算問卷形式做前後測評量 |
| Module 3 | Aliens entering houses with transfer tracking — 外星人走進房子，部分轉移到另一間 | **Game3** | 工作記憶負荷下的數量追蹤（transformation tracking） |

### 論文的關鍵測量指標 → 數據系統設計對應

| 論文測量 | 論文做法 | 本系統對應 | 設計備註 |
|---------|---------|-----------|---------|
| **訓練強度（dosage）** | 總訓練時數分高/低兩組（>6h vs <2h） | `game_session.duration_seconds` 加總 | 可依據 SUM(duration_seconds) 動態分群，不需預設閾值 |
| **數學焦慮** | CMAQ 8 題 5 點量表，前後測比較 | `questionnaire_responses`（MathAttitude 量表已實作 5 題） | 目前量表只有 5 題，考慮是否需補齊到 CMAQ 的 8 題 |
| **數學成就 / 算術能力** | BMCST 標準化測驗 | `af_test_results`（加減運算問卷前後測） | 對應論文 Module 2 的算術能力測量。以問卷形式做前後測，非遊戲內訓練模組 |
| **工作記憶** | WISC-IV Digit Span | 目前未收集 | 紙筆測驗，不在遊戲內收集 |
| **數量比較能力** | Module 1 的訓練歷程 | `trial_log`（number_left, number_right, response_time_ms, is_correct） | **論文最大的數據缺口就在這裡——沒有報告逐題數據**。本系統彌補此缺口 |
| **焦慮特質-表現關聯** | 前測焦慮分組 × 遊戲逐題表現的交叉分析 | 焦慮量表 → `questionnaire_responses`，表現 → `trial_log` + `game_session` | 比論文更細緻：焦慮特質 × 不同難度/距離的反應時間（非即時焦慮測量） |

### 論文的數據缺口 → 本系統如何彌補

論文作為初步研究（pilot study），有幾個明顯的數據限制：

1. **無逐題行為數據**：論文只報告「總訓練時數」作為干預強度的指標，沒有分析模組內的逐題表現。本系統的 `trial_log` 可以追蹤每一題的數字配對、反應時間和正確性，支持 Weber 定律（距離效應）和 ANS 精確度的精細分析。

2. **強度分組過於粗略**：論文只分高/低兩組（>6h vs <2h），中間地帶（2-6h）的參與者被排除（n=12）。本系統記錄精確的 `duration_seconds`，可做連續變數分析而非二分法。

3. **無法追蹤學習曲線**：論文只有前後測，看不到訓練過程中的表現變化。本系統的 `trial_log` 按時間序列記錄，可繪製每日/每週的正確率和反應時間曲線。

4. **Game3 的工作記憶負荷未量化**：論文提到 Module 3 涉及轉移追蹤，但沒有分析不同轉移次數對表現的影響。本系統的 `transfer_count` 和 `transfer_detail` 欄位可以精確分析工作記憶負荷（0/1/2 次轉移）與正確率的關係。

5. **焦慮特質對遊戲表現的差異分析**：論文只測前後總焦慮，沒有分析不同焦慮水平的兒童在遊戲中的表現差異。本系統可以用前測焦慮分數將兒童分組，交叉分析高焦慮 vs 低焦慮兒童在不同難度、不同數字距離下的反應時間和正確率差異。注意：這是「焦慮特質 × 表現」的分析，非即時焦慮狀態測量（即時焦慮需要生理指標或遊戲中嵌入自陳量表，不在本系統範圍）。

### 論文發現對系統設計的驗證

| 論文發現 | 驗證方式 | 相關 SQL 查詢思路 |
|---------|---------|------------------|
| 高強度訓練（>6h）顯著降低數學焦慮 | 比較不同 `SUM(duration_seconds)` 區間的前後測焦慮分數差異 | `SELECT SUM(gs.duration_seconds)/3600 as hours, ... FROM game_session gs JOIN questionnaire_responses qr ...` |
| 高焦慮兒童焦慮降幅最大（r=-.51） | 焦慮初始分 × 焦慮變化量的相關分析 | 前測焦慮分數 JOIN 後測焦慮分數，計算差值 |
| 訓練提升數學成就（p<.05, d=0.39） | 比較不同訓練量組的 `af_test_results` 前後測差異 | `SELECT ... FROM af_test_results WHERE test_type IN ('pre','post') ...` |
| 訓練提升工作記憶 digit span（p<.05, d=0.45） | Game3 的 `transfer_count` 表現隨訓練時數增加的變化 | `SELECT ... FROM trial_log WHERE game_type=3 GROUP BY transfer_count ...` |

### 數學焦慮量表對照

論文使用 CMAQ（Children's Mathematics Anxiety Questionnaire，8 題）。目前系統中的 `MathAttitudeQuestions`（5 題）涵蓋：

| 目前量表題目 | 對應構念 | CMAQ 是否涵蓋 |
|-------------|---------|--------------|
| 數學是有用的科目 | 數學態度（價值） | ✓ 相關但不同構念 |
| 數學在生活中很重要 | 數學態度（價值） | ✓ 相關但不同構念 |
| 上數學課很快樂 | 數學享受 | ✓ CMAQ 第 7 題類似 |
| 考數學時覺得很緊張 | 數學焦慮 | ✓ CMAQ 核心測量 |
| 覺得自己數學很厲害 | 數學自我效能 | △ CMAQ 不直接測量 |

> **建議（供研究者討論）**：如果要與論文的 CMAQ 直接對比，可能需要補充 3 題焦慮相關題目。但這取決於研究團隊的選擇，不影響數據系統的結構設計。

---

## 背景：目前的數據收集現狀

### 目前已收集的數據

| 資料表 | 欄位 | 說明 |
|--------|------|------|
| `training_history` | user_id, game_type, score, lv, training_time, date | 每次存檔（每秒自動存一次）的摘要 |
| `user` | game1/2/3_daily_time, game1/2/3_total_star, game1/2/3_best_star, game1/2/3_daily_die | 每日統計 |
| `questionnaire_responses` | user_id, type, answers[] | PPTQ-C 人格問卷、數學態度量表 |
| `af_test_results` | user_id, test_type, total_correct, time_taken, answers[] | 算術流暢性前後測 |
| `user` | coins, crystals, consecutive_days, last_login_date | 遊戲經濟與登入紀錄 |

### 三個遊戲的數學認知設計

| 遊戲 | 核心腳本 | 數感能力 | 機制 |
|------|---------|---------|------|
| Game1 | `score_g4.cs` | 數量比較（subitizing → counting） | 兩組怪物掉落，射擊較多的一邊。等級擴展數值範圍 0-4→0-9→0-14，速度遞增 |
| Game2 | `score.cs` + 子關 | 數量比較（多元呈現） | 類似 Game1 但不同視覺元素（水/石/柱），三個子關卡 |
| Game3 | `Game3_MainController.cs` | 數量變換追蹤（transformation tracking） | 外星人進兩個倉庫，部分轉移，判斷哪邊多。intro/easy/hard 模式 |

### WeightedScoreTracker 加權計分（目前僅存 PlayerPrefs，未上傳後端）

```
最終分 = correctCount × difficultyMultiplier
星星門檻: 120→5★, 80→4★, 45→3★, 20→2★, 1→1★
```

### 三個遊戲的節奏差異（影響數據接收設計）

| 遊戲 | 每題平均耗時 | 10 分鐘約可作答 | 瓶頸 |
|------|------------|----------------|------|
| Game1 | ~5-8 秒 | 75-120 題 | 怪物落地速度隨等級加快 |
| Game2 | ~5-8 秒 | 75-120 題 | 與 Game1 類似 |
| Game3 | ~15-25 秒 | 24-40 題 | **外星人走入倉庫 + 轉移動畫** 佔大量時間 |

> Game3 每題耗時是 Game1/2 的 3-4 倍，因此「每 5 題批次上傳」在 Game3 中約每 1.5-2 分鐘才上傳一次（而 Game1/2 約 30-40 秒）。需考慮：
> - Game3 的批次大小可以降到 3 題，確保資料更頻繁上傳
> - 或者改用「每 60 秒至少上傳一次」的時間觸發機制作為補充

### 能力系統對統計數據的影響

| 能力 | 類型 | 效果 | 統計處理 |
|------|------|------|---------|
| HeartShield 護心盾牌 | 主動 | 恢復一條命 | **不排除** — 不影響題目呈現或作答 |
| TimeWand 時間魔法棒 | 主動 | Game1/2: 暫停 5 秒; Game3: 延長作答 5 秒 | **標記** — response_time 受影響但題目本身有效 |
| MagicGlasses 魔法眼鏡 | 主動 | 直接顯示正確答案 | **排除** — 完全汙染作答結果 |
| MeteorDodge 流星閃避 | 主動 | Game1/2: 清除怪物; Game3: 跳過題目 | **排除** — 無真實作答 |
| Hourglass 脈衝引擎 | 被動 | 加速消耗剩餘時間 | 記錄但不排除個別題目 |
| IceCrystal 緩時冰晶 | 被動 | 減緩時間流逝 | 記錄但不排除個別題目 |
| LuckyStar 幸運星星 | 被動 | 星星數加倍 | 記錄但不排除個別題目 |

> **設計決策**：使用魔數能力（除 HeartShield 外）的題目 **直接不寫入 `trial_log`**，從源頭保證 trial_log 只包含乾淨的研究數據。能力使用的追蹤由獨立的 `ability_usage_log` 表負責，遊戲開發人員仍可分析能力使用頻率和時機，但不會汙染逐題研究數據。
>
> 具體規則：
> - **HeartShield**（護心盾牌）：不影響題目呈現或作答 → **正常寫入** trial_log
> - **TimeWand**（時間魔法棒）：影響反應時間 → **不寫入** trial_log，記入 ability_usage_log
> - **MagicGlasses**（魔法眼鏡）：直接顯示正確答案 → **不寫入** trial_log，記入 ability_usage_log
> - **MeteorDodge**（流星閃避）：跳過題目/清除怪物 → **不寫入** trial_log，記入 ability_usage_log
> - 被動能力（Hourglass / IceCrystal / LuckyStar）：影響遊戲節奏但不影響個別題目 → **正常寫入** trial_log，被動能力的啟用狀態記入 `game_session`

---

## 問題分析：從兒童數感心理學角度看數據缺口

### 1. 完全缺失的「逐題數據」（最關鍵缺口）

目前只存「每秒摘要」（score, lv, training_time），**完全沒有逐題（per-trial）紀錄**。這表示：

- 無法知道每一題出了什麼數字配對（例如 3 vs 7 和 4 vs 5 的正確率差異）
- 無法知道每題的反應時間（RT）——這是數感研究最核心的測量指標
- 無法分析「距離效應」（distance effect）：數字差距大的題目是否比差距小的快且準
- 無法分析「大小效應」（size effect）：相同差距下，數字越大是否越慢越容易錯
- 無法追蹤 Game3 的轉移追蹤失敗模式（是數錯了還是追不到轉移？）

> **心理學意義**：Weber 定律在兒童數感發展中預測，比較 2 vs 8 應該比 5 vs 6 快很多。如果某個孩子在差距=1 的題目和差距=5 的題目表現一樣差，可能暗示 ANS（近似數量系統）精確度低，需要特別關注。

### 2. 難度適應歷程未記錄

三個遊戲都有動態難度調整（Game1 每 10 題升級、Game3 根據 playerAnswerRightCount 調整），但升降級的觸發時機和玩家在各難度的表現沒有結構化地被記錄。

> **研究價值**：ZPD（最近發展區）分析——孩子在哪個難度帶停留最久？突破某難度需要多少次嘗試？

### 3. 會話層級指標缺失

- 沒有「一次遊玩會話」的開始/結束紀錄（只有每秒存檔的 training_time）
- 不知道孩子是主動離開還是時間到了
- 不知道暫停了幾次、每次暫停多久
- 無法區分「專注遊玩 10 分鐘」和「玩 2 分鐘暫停 8 分鐘」

### 4. 能力/道具使用未追蹤

- 魔數能力（AbilityEffects 系統）的啟用時機、效果、使用頻率沒有記錄
- 無法分析道具是否真的幫助了表現提升

### 5. WeightedScoreTracker 資料未上傳

加權計分是目前最精確的表現指標，但只存在 PlayerPrefs（本地），伺服器完全沒有這些數據。

---

## 提案：新增的研究數據系統

### 各資料表欄位的研究依據

雖然 Ng et al. (2022) 論文本身沒有收集 trial-level 數據，但以下欄位設計在數值認知與教育心理學研究領域有堅實的理論基礎。

#### trial_log 欄位的研究依據

| 欄位 | 研究依據 | 說明 | 關鍵文獻 |
|-----|---------|------|---------|
| **response_time_ms** | 反應時間是認知心理學的基本測量指標 | RT 可反映數字處理的自動化程度與認知效率 | Moyer & Landauer (1967); Dehaene (1997) *The Number Sense* |
| **number_distance** | **Distance Effect（距離效應）** | 兩數距離越小，比較越難（RT↑、錯誤率↑）——數值認知最經典的效應之一 | Moyer & Landauer (1967); Sekuler & Mierkiewicz (1977) |
| **number_ratio** | **Weber's Law（韋伯定律）** | 比值（小數/大數）越接近 1，辨別越困難；可計算 Weber fraction 評估 ANS 精確度 | Halberda et al. (2008) *Nature*; Piazza et al. (2010) |
| **is_correct** | Speed-Accuracy Tradeoff | RT 與 accuracy 需一起分析才能完整理解表現；可計算 d'（敏感度） | Signal Detection Theory |
| **number_left/right** | **Size Effect（大小效應）** | 數字越大，處理越慢；可分析不同數量範圍的表現差異 | Parkman (1971); Verguts & Fias (2004) |
| **transfer_count** | 工作記憶負荷 | Game3 的轉移次數反映工作記憶需求 | Ng (2022) 第7頁：Module 3 涉及 working memory |

**可衍生的研究指標：**
- **Distance Effect 斜率** = RT 對 distance 的迴歸斜率 → 斜率越陡，數字處理越依賴 magnitude comparison
- **Weber Fraction (w)** = 從正確率 vs ratio 擬合心理物理曲線 → w 越小，ANS 精確度越高
- **Size Effect** = RT 隨數量大小增加的趨勢 → 反映 counting 策略 vs subitizing
- **Coefficient of Variation (CV)** = SD(RT) / Mean(RT) → 反應一致性，與注意力/執行功能相關

#### game_session 欄位的研究依據

| 欄位 | 研究依據 | 說明 | 關鍵文獻 |
|-----|---------|------|---------|
| **duration_seconds** | **Training Dosage（訓練劑量）** | Ng (2022) 第9頁直接用訓練時數分組（>6h vs <2h）——論文唯一明確收集的訓練過程指標 | Ng et al. (2022) Table 2 |
| **start_time / end_time** | **Spacing Effect（間隔效應）** | 分散練習優於集中練習；可分析練習間隔與規律性 | Ebbinghaus (1885); Cepeda et al. (2006) |
| **total_trials / correct_count** | **Exposure Dosage（暴露劑量）** | 暴露療法的核心概念——Ng (2022) 第3頁引用 Abramowitz et al. (2011) | Deliberate Practice: Ericsson (1993) |
| **accuracy_rate / avg_response_ms** | **Learning Curve Analysis** | 追蹤跨 session 的進步軌跡；Power Law of Practice | Anderson (1982); Käser et al. (2013) |
| **max_level / end_level** | **Adaptive Learning** | Ng (2022) 第7頁提到遊戲難度根據表現調整；反映技能精熟程度 | Zone of Proximal Development: Vygotsky; Flow Theory: Csikszentmihalyi (1990) |

**可衍生的研究指標：**
- **訓練頻率** = sessions / 總天數 → 練習規律性，與 compliance 相關
- **平均 session 長度** = total_duration / session_count → 單次專注力/投入程度
- **學習曲線斜率** = accuracy 或 level 對 session 的迴歸 → 學習速率，可預測最終成效
- **練習間隔變異** = SD(session 間隔) → Spacing effect 的操作化指標

#### ability_usage_log 欄位的研究依據

此表相較於 `trial_log` 和 `game_session`，**較少直接的數值認知研究支持**，但有 game-based learning 和教育心理學的理論基礎：

| 欄位 | 研究依據 | 說明 | 關鍵文獻 |
|-----|---------|------|---------|
| **ability_name** | **Scaffolding（鷹架理論）** | 技能可視為學習鷹架，降低認知負荷 | Wood et al. (1976); Plass et al. (2015) - Ng (2022) 第3頁引用 |
| **usage_count** | **Help-seeking Behavior（求助行為）** | 求助行為反映自我調節學習能力；理想學習歷程應逐漸減少對鷹架的依賴 | Karabenick & Knapp (1991); Aleven et al. (2003) |
| **effective_count** | **Strategic Help-seeking** | 區分「策略性求助」vs「逃避性求助」；偵測學生是否濫用系統機制 | Baker et al. (2004) Gaming the System; Flavell (1979) Metacognition |

**可衍生的研究指標：**
- **技能依賴度** = total_usage / total_trials → 高依賴可能表示任務過難或缺乏信心
- **技能效率** = effective_count / usage_count → 低效率可能表示亂用或不理解機制
- **依賴變化趨勢** = 使用率隨 session 的變化 → 理想情況：隨技能提升而減少使用
- **困難關卡指標** = 特定關卡的技能使用激增 → 識別需要調整難度的關卡

**與焦慮的潛在關聯（探索性假設）：**
- 高焦慮 → 高技能使用？（焦慮者可能更依賴「安全網」）
- 技能使用 → 焦慮降低？（有後援可能減少挫折感）
- 過度使用 → 學習遷移差？（過度依賴鷹架可能阻礙獨立能力發展）

#### 各資料表研究依據強度評估

| 資料表 | 數值認知研究依據 | 教育/遊戲研究依據 | 定位 |
|-------|----------------|------------------|------|
| **trial_log** | ⭐⭐⭐⭐⭐ 非常強 | ⭐⭐⭐ | 核心研究數據 |
| **game_session** | ⭐⭐⭐ 中等 | ⭐⭐⭐⭐ | 訓練劑量與學習歷程 |
| **ability_usage_log** | ⭐ 較弱 | ⭐⭐⭐ 中等 | 探索性/補充性數據 |

> **ability_usage_log 的定位說明**：此表更偏向遊戲設計分析（Game Analytics）、學習行為探索性研究、UX/玩家體驗研究。在研究報告中可定位為：「除了認知層面的表現數據，本研究亦收集遊戲內輔助機制的使用行為，以探索學習者的**自我調節策略**與**求助行為模式**，作為理解數學焦慮者學習歷程的補充指標。」

#### 建議引用的參考文獻

以下文獻可支撐 trial-level 數據收集的理論基礎：

1. **Halberda, J., Mazzocco, M. M., & Feigenson, L. (2008)**. Individual differences in non-verbal number acuity correlate with maths achievement. *Nature*, 455, 665-668. → Weber fraction 與數學成就
2. **Moyer, R. S., & Landauer, T. K. (1967)**. Time required for judgements of numerical inequality. *Nature*, 215, 1519-1520. → Distance effect 經典研究
3. **Dehaene, S. (2011)**. *The Number Sense* (Revised). Oxford University Press. → 數感理論綜述
4. **Piazza, M., Facoetti, A., et al. (2010)**. Developmental trajectory of number acuity reveals a severe impairment in developmental dyscalculia. *Cognition*, 116, 33-41. → ANS 與計算障礙
5. **Cepeda, N. J., et al. (2006)**. Distributed practice in verbal recall tasks: A review and quantitative synthesis. *Psychological Bulletin*, 132, 354-380. → Spacing effect
6. **Käser, T., et al. (2013)**. Modelling and optimizing mathematics learning in children. *International Journal of Artificial Intelligence in Education*, 23, 115-135. → 數學學習遊戲的學習曲線
7. **Aleven, V., et al. (2003)**. Help seeking and help design in interactive learning environments. *Review of Educational Research*, 73, 277-320. → 求助行為研究
8. **Plass, J. L., Homer, B. D., & Kinzer, C. K. (2015)**. Foundations of game-based learning. *Educational Psychologist*, 50, 258-283. → 遊戲機制與學習（Ng 2022 有引用）

---

### 層級一：逐題數據表 `trial_log`（最高優先級）

這是最關鍵的新增——記錄每一次作答的完整資訊。

#### 資料表結構

```sql
CREATE TABLE trial_log (
    trial_id        BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id         VARCHAR(100) NOT NULL,
    session_id      VARCHAR(50) NOT NULL,       -- 會話 ID（同一次遊玩的所有題目共享）
    game_type       TINYINT NOT NULL,            -- 1, 2, 3
    trial_index     INT NOT NULL,                -- 該會話中的第幾題（從 0 開始）

    -- 題目內容
    number_left     INT NOT NULL,                -- 左邊的數量
    number_right    INT NOT NULL,                -- 右邊的數量
    correct_side    VARCHAR(5) NOT NULL,         -- "left" / "right"
    number_distance INT NOT NULL,                -- abs(left - right)，方便查詢距離效應
    number_max      INT NOT NULL,                -- max(left, right)，方便查詢大小效應

    -- Game3 專用（可 NULL）
    transfer_count  TINYINT DEFAULT NULL,        -- 轉移次數（0=intro, 1=easy, 2=hard）
    transfer_detail VARCHAR(200) DEFAULT NULL,   -- 轉移詳情 JSON，例如 [{"from":"left","count":2},{"from":"right","count":1}]

    -- 玩家作答
    player_answer   VARCHAR(10) NOT NULL,        -- "left" / "right" / "timeout"
    is_correct      TINYINT NOT NULL,            -- 0 或 1
    response_time_ms INT NOT NULL,               -- 從題目完全呈現到作答的毫秒數
    animation_duration_ms INT DEFAULT 0,         -- 題目動畫時長（Game3 外星人走路+轉移），研究可分析「純思考時間 = response - animation」

    -- 難度上下文
    -- 注意：使用魔數能力（除 HeartShield 外）的題目不會寫入此表，
    -- 因此 trial_log 中的所有資料都是乾淨的研究數據，不需要額外過濾。
    difficulty_level INT NOT NULL,               -- Game1: mylevel, Game2: mylevel, Game3: playerAnswerRightCount
    cumulative_correct INT NOT NULL,             -- 到這題為止的累計正確數
    cumulative_wrong INT NOT NULL,               -- 到這題為止的累計錯誤數
    weighted_score  FLOAT DEFAULT 0,             -- 該題的加權得分

    -- 時間戳
    created_at      DATETIME DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_user_game (user_id, game_type),
    INDEX idx_session (session_id),
    INDEX idx_distance (number_distance),
    INDEX idx_created (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

#### 資料純淨性保證

**trial_log 中的所有資料都是乾淨的研究數據**——使用魔數能力（除 HeartShield 外）的題目在 Unity 端就被攔截，不會寫入此表。研究人員無需額外過濾，直接查詢即可。

能力使用的頻率和時機由獨立的 `ability_usage_log` 表追蹤，供遊戲開發人員分析。

#### 研究分析用途

| 分析類型 | 查詢方式 | 心理學意義 |
|---------|---------|-----------|
| 距離效應 | GROUP BY number_distance, AVG(response_time_ms), AVG(is_correct) | Weber 定律驗證，ANS 精確度估計 |
| 大小效應 | GROUP BY number_max WHERE number_distance = N | 同距離下大數字是否更難 |
| 學習曲線 | 按 session 順序看 is_correct 和 response_time_ms 變化 | 練習效果、學習速率 |
| 速度-準確度權衡 | 散佈圖 response_time_ms vs is_correct | 衝動型 vs 謹慎型策略 |
| 轉移追蹤能力 | Game3: GROUP BY transfer_count, AVG(is_correct) | 工作記憶負荷對數感的影響 |
| 困難數字配對 | GROUP BY number_left, number_right | 找出特定困難配對（可能暗示概念理解問題）|
| 超時分析 | WHERE player_answer = "timeout" | 注意力或理解困難指標 |
| 訓練強度-焦慮關聯（Ng et al. 複製） | SUM(game_session.duration_seconds) × 焦慮量表前後差 | 驗證論文的核心發現：>6h 訓練降低焦慮 |
| ANS 精確度（Weber fraction）估計 | 距離效應斜率擬合 | 論文未報告的精細指標，可作為訓練效果的認知機制解釋 |

### 層級二：會話數據表 `game_session`

#### 資料表結構

```sql
CREATE TABLE game_session (
    session_id      VARCHAR(50) PRIMARY KEY,
    user_id         VARCHAR(100) NOT NULL,
    game_type       TINYINT NOT NULL,

    -- 時間
    start_time      DATETIME NOT NULL,
    end_time        DATETIME DEFAULT NULL,
    duration_seconds INT DEFAULT 0,              -- 實際遊玩秒數
    pause_count     INT DEFAULT 0,               -- 暫停次數
    total_pause_seconds INT DEFAULT 0,           -- 總暫停秒數

    -- 結束方式
    end_reason      VARCHAR(20) DEFAULT NULL,    -- "time_up" / "quit" / "interrupted" / "disconnect"
    -- time_up: 遊戲時間到（正常結束）
    -- quit: 玩家主動退出（按暫停→離開）
    -- interrupted: App 被關閉/切到背景超過 5 分鐘/下次啟動時補發
    -- disconnect: 網路斷線（保留）

    -- 會話摘要（所有數值均基於寫入 trial_log 的乾淨題目，能力汙染題目已在源頭排除）
    total_trials    INT DEFAULT 0,               -- 總題數（= trial_log 中該 session 的筆數）
    correct_count   INT DEFAULT 0,
    wrong_count     INT DEFAULT 0,
    timeout_count   INT DEFAULT 0,
    accuracy_rate   FLOAT DEFAULT 0,             -- 正確率 = correct_count / total_trials
    avg_response_ms INT DEFAULT 0,               -- 平均反應時間
    ability_skipped_trials INT DEFAULT 0,        -- 因能力使用而未記錄的題目數（供參考）

    -- 難度紀錄
    start_level     INT DEFAULT 0,
    end_level       INT DEFAULT 0,
    max_level       INT DEFAULT 0,               -- 最高到達的難度
    level_ups       INT DEFAULT 0,               -- 升級次數

    -- 計分
    weighted_score  FLOAT DEFAULT 0,             -- WeightedScoreTracker 的最終分
    star_count      INT DEFAULT 0,               -- 星星數
    crystals_earned INT DEFAULT 0,               -- 獲得的結晶數
    die_count       INT DEFAULT 0,               -- 死亡次數

    INDEX idx_user_game (user_id, game_type),
    INDEX idx_date (start_time)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 層級三：能力使用紀錄表 `ability_usage_log`

```sql
CREATE TABLE ability_usage_log (
    id              BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id         VARCHAR(100) NOT NULL,
    session_id      VARCHAR(50) NOT NULL,
    game_type       TINYINT NOT NULL,
    ability_name    VARCHAR(50) NOT NULL,         -- "shield" / "lucky_star" / "slow" / etc.
    used_at         DATETIME DEFAULT CURRENT_TIMESTAMP,
    trial_index     INT DEFAULT NULL,             -- 使用時是第幾題

    INDEX idx_session (session_id),
    INDEX idx_user (user_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

---

## 實作方案

### 新建檔案

#### 1. `Assets/Scripts/TrialLogger.cs` — 逐題數據收集器

```
核心設計：
- DontDestroyOnLoad 單例
- 每次開始遊玩時 StartSession(gameType) → 產生 session_id (GUID)
- 每題結束呼叫 LogTrial(trialData) → 暫存到記憶體 List
- 依遊戲類型自動調整批次大小（見下方）
- EndSession(endReason) → 上傳剩餘 trial + 寫入 game_session

API:
- StartSession(int gameType)
- LogTrial(TrialData data)
  - TrialData 結構體包含所有欄位
  - TrialData 包含 animationDurationMs（Game3 動畫時長）
  - 內部檢查：如果 currentAbilityUsed 不為空且不是 HeartShield，
    則 **跳過寫入 trial_log**，改為 abilitySkippedCount++，
    並記入 ability_usage_log
- MarkAbilityUsed(string abilityName)
  - 標記「目前正在作答的這一題」使用了某能力
  - 由 AbilityEffects.ExecuteAbility() 呼叫
  - LogTrial 時檢查此標記決定是否寫入 trial_log
- ClearAbilityMark()
  - 每題開始時重置能力標記
- EndSession(string endReason)
  - 計算 accuracy_rate = correct_count / total_trials
  - ability_skipped_trials 記錄因能力而跳過的題數
- PauseSession() / ResumeSession()

能力排除邏輯（在 LogTrial 內部）：
- 每題開始時 currentAbilityUsed = null
- 使用能力時 AbilityEffects 呼叫 MarkAbilityUsed("MagicGlasses") 等
- LogTrial 被呼叫時：
  - if (currentAbilityUsed == null || currentAbilityUsed == "HeartShield")
    → 正常寫入 trial_log
  - else
    → 不寫入 trial_log，abilitySkippedCount++
    → 寫入 ability_usage_log（記錄使用了什麼能力、第幾題、什麼時候）
- 被動能力（Hourglass/IceCrystal/LuckyStar）不觸發 MarkAbilityUsed，
  它們的啟用狀態記入 game_session 的備註欄位

批次上傳策略（考慮遊戲節奏差異）：
- Game1/2：每 5 題上傳一次（約 30-40 秒）
- Game3：每 3 題上傳一次（約 45-75 秒），因為每題動畫較長
- 補充機制：無論題數，每 60 秒至少上傳一次（Timer 觸發）
- 上傳失敗時保留在本地 queue，下次重試
- EndSession 時強制上傳所有剩餘資料

小孩中斷容錯機制（關鍵設計）：
小孩可能隨時中斷遊戲（關 app、切到其他 app、手機沒電、家長強制關閉等），
必須確保已收集的數據不會遺失。

1. OnApplicationPause(true) — App 進入背景
   - 立即觸發一次批次上傳（把記憶體中的 trial 全部送出）
   - 同時寫入本地 PlayerPrefs 備份（JSON 序列化未上傳的 trial list）
   - 記錄暫停時間戳（計算暫停時長）

2. OnApplicationPause(false) — App 回到前景
   - 恢復計時
   - 檢查離開時長：如果離開超過 5 分鐘，視為中斷，
     自動呼叫 EndSession("interrupted") 結束上一個會話

3. OnApplicationQuit — App 被完全關閉
   - 同步觸發最後一次上傳（Unity 的 OnApplicationQuit 允許短暫的同步操作）
   - 如果上傳失敗或來不及完成，確保 PlayerPrefs 備份已寫入

4. 下次啟動時恢復（Start / Awake）
   - 檢查 PlayerPrefs 中是否有未完成的會話資料
   - 如果有：
     a. 先把上次未上傳的 trial 資料補傳到後端
     b. 補發 EndSession("interrupted") 給後端，關閉上次的 game_session
     c. 清除本地備份
   - 這確保即使 app 被強制殺掉，資料也不會永久遺失

5. 本地備份格式（PlayerPrefs）
   - Key: "TrialLogger_PendingSession" → session_id + game_type + start_time
   - Key: "TrialLogger_PendingTrials" → JSON 陣列，每個元素是一筆 TrialData
   - Key: "TrialLogger_PendingAbilityLogs" → JSON 陣列
   - 每次成功上傳後清除對應的本地備份
   - 每次 LogTrial 後更新本地備份（追加到 JSON 陣列）

6. 後端 game_session 的 end_reason 值
   - "time_up": 遊戲時間到（正常結束）
   - "quit": 玩家主動退出（按暫停→離開）
   - "interrupted": App 被關閉 / 切到背景超過 5 分鐘 / 下次啟動時補發
   - "disconnect": 網路斷線導致無法繼續（保留，目前不一定用到）
```

#### 2. `mathgame-php-data-server/trial_log.php` — 逐題數據 API

```
端點：
- POST action=batch_insert: 接收 JSON 陣列，批次寫入 trial_log
- POST action=end_session: 寫入/更新 game_session 記錄
- GET action=get_trials: 依 user_id + 日期範圍查詢（給 viewer 用）
- GET action=get_session_summary: 取得會話摘要
```

#### 3. `mathgame-php-data-server/research_export.php` — 研究數據匯出 API

```
端點：
- GET action=export_trials: 匯出指定條件的 trial_log（CSV/JSON）
  - 篩選：user_id, game_type, date_range, difficulty_level
- GET action=export_sessions: 匯出 game_session 摘要
- GET action=export_distance_analysis: 預計算距離效應分析結果
- GET action=export_learning_curve: 預計算學習曲線數據

加上基本的 API key 驗證，避免公開存取
```

### 修改檔案

#### 4. `Assets/Scripts/score_g4.cs`（Game1）— 加入逐題紀錄

修改位置：玩家點擊按鈕判定答案正確/錯誤的程式碼段

```
需捕捉的數據：
- number_left, number_right：兩邊怪物數量（目前用 random1, random2 產生）
- response_time_ms：從題目呈現（怪物落地）到玩家點擊的時間差
- player_answer："left" / "right"
- is_correct：目前已有判斷邏輯
- difficulty_level：mylevel

新增計時變數：questionPresentedTime（float），在怪物完成落地時記錄 Time.time
作答時計算：responseTime = (Time.time - questionPresentedTime) * 1000
```

#### 5. `Assets/Scripts/Game2/score.cs`（及 score2_2.cs, score2_3.cs）— 同上

與 Game1 相同的修改模式，但需注意三個子腳本都要改。

#### 6. `Assets/Scripts/Game3/Game3_MainController.cs` — 加入逐題紀錄（含轉移細節 + 動畫時長）

修改位置：`RespondAnswer()` 方法 + `ShowQuestion()` 動畫序列

```
需捕捉的額外數據：
- transfer_count：transferTime（0/1/2）
- transfer_detail：JSON 紀錄每次轉移的方向和數量
  例如：[{"from":"left","to":"right","count":2}]
- response_time_ms：從動畫完畢（waitForAnswer 開始）到玩家按鈕的時間差
  目前 WaitForAnswer() 有 9 秒倒數計時器，可用它來反推
- animation_duration_ms：從題目開始（外星人開始走入倉庫）到動畫結束的毫秒數
  在 ShowQuestion() 動畫開始時記 animStartTime = Time.time
  在 WaitForAnswer() 開始時記 animEndTime = Time.time
  animation_duration_ms = (animEndTime - animStartTime) * 1000

  > 這讓研究人員可以分析「純思考時間 = response_time - animation_duration」
  > Game1/2 的 animation_duration 接近 0（怪物幾乎同時出現）
```

#### 7. `Assets/Scripts/AbilityEffects.cs` — 在能力觸發時標記 TrialLogger（觸發題目排除）

修改位置：`ExecuteAbility()` 方法

```csharp
bool ExecuteAbility(AbilityManager.AbilityType type) {
    // 標記當前題目使用了能力
    // TrialLogger 收到標記後，LogTrial 時會判斷：
    // - HeartShield → 正常寫入 trial_log（不影響作答）
    // - 其他主動能力 → 該題不寫入 trial_log，改記入 ability_usage_log
    TrialLogger.Instance?.MarkAbilityUsed(type.ToString());

    switch (type) { ... }
}
```

> HeartShield 也會被標記，但 TrialLogger 內部對 HeartShield 做了白名單處理，不會排除該題。
> 被動能力（Hourglass/IceCrystal/LuckyStar）的啟用由 game_session 記錄，不觸發逐題排除。

#### 8. `Assets/Scripts/PauseController.cs` — 通知 TrialLogger 暫停/恢復

```csharp
// 在 TogglePause() 中加入：
if (isPaused)
    TrialLogger.Instance?.PauseSession();
else
    TrialLogger.Instance?.ResumeSession();

// 在 ExitToMenu() 中加入：
TrialLogger.Instance?.EndSession("quit");
```

#### 9. `Assets/Scripts/open/GameTimeLimiter.cs` — 時間到時結束會話

```csharp
// 在 OpenTimesUpWindow() 中加入：
TrialLogger.Instance?.EndSession("time_up");
```

#### 10. `Assets/Scripts/open/WeightedScoreTracker.cs` — 上傳加權分數

在 `EndOfDayReset()` 或 `GetStarCount()` 被呼叫時，將分數寫入 game_session。
TrialLogger.EndSession() 會負責最終寫入。

### 玩家數據查看器增強（未來可做，本次不含）

以下列出研究人員需要的查看功能，但 **不在本次實作範圍**：

- 逐題數據瀏覽：表格顯示每一題的數字配對、作答、反應時間
- 距離效應圖表：X 軸=數字差距，Y 軸=平均正確率/反應時間
- 學習曲線圖：X 軸=累計題數，Y 軸=正確率（移動平均）
- 難度進程圖：X 軸=時間/題序，Y 軸=難度等級
- CSV 匯出按鈕

---

## 實作順序

### 第一步：資料庫建表
- 執行 SQL 建立 `trial_log`, `game_session`, `ability_usage_log` 三張表

### 第二步：PHP 後端 API
- 建立 `trial_log.php` — 批次寫入 + 會話結束
- 建立 `research_export.php` — 匯出端點

### 第三步：Unity 端 TrialLogger
- 建立 `TrialLogger.cs` 單例
- 實作 StartSession / LogTrial / EndSession / PauseSession / ResumeSession
- 實作批次上傳邏輯（每 5 題 + 結束時）

### 第四步：三個遊戲腳本改造
- `score_g4.cs`：加入反應時間計時 + LogTrial 呼叫
- `score.cs` / `score2_2.cs` / `score2_3.cs`：同上
- `Game3_MainController.cs`：加入反應時間計時 + 轉移細節 + LogTrial 呼叫

### 第五步：整合暫停/結束事件
- `PauseController.cs`：暫停通知 + 退出結算
- `GameTimeLimiter.cs`：時間到結束會話

### 第六步：驗證與測試
- 在 debug 模式下玩完一輪三個遊戲
- 檢查 trial_log 和 game_session 資料是否正確
- 用 research_export.php 匯出 CSV 驗證格式

---

## 關鍵檔案清單

| 檔案 | 操作 | 說明 |
|------|------|------|
| `Assets/Scripts/TrialLogger.cs` | **新建** | 逐題數據收集單例（含能力排除、中斷容錯、正確率計算） |
| `mathgame-php-data-server/trial_log.php` | **新建** | 逐題數據 + 會話 API |
| `mathgame-php-data-server/research_export.php` | **新建** | 研究數據匯出 API（支援年級篩選、能力排除） |
| `Assets/Scripts/score_g4.cs` | **修改** | Game1 逐題紀錄 |
| `Assets/Scripts/Game2/score.cs` | **修改** | Game2 逐題紀錄 |
| `Assets/Scripts/Game2/score2_2.cs` | **修改** | Game2 子關逐題紀錄 |
| `Assets/Scripts/Game2/score2_3.cs` | **修改** | Game2 子關逐題紀錄 |
| `Assets/Scripts/Game3/Game3_MainController.cs` | **修改** | Game3 逐題紀錄（含轉移細節 + 動畫時長） |
| `Assets/Scripts/AbilityEffects.cs` | **修改** | 能力觸發時標記 TrialLogger |
| `Assets/Scripts/PauseController.cs` | **修改** | 暫停/退出通知 |
| `Assets/Scripts/open/GameTimeLimiter.cs` | **修改** | 時間到結束會話 |
| 資料庫 | **修改** | 新建三張表 + `user` 表加 grade/birth_year/school_code/cohort 欄位 |

---

## 年級擴展預留設計

目前對象是小一小二，之後可能擴展到其他年齡的學齡兒童。以下設計預留擴展彈性：

### `user` 表新增欄位（本次一併加入）

```sql
ALTER TABLE user ADD COLUMN grade TINYINT DEFAULT NULL;         -- 年級（1=小一, 2=小二, ...6=小六）
ALTER TABLE user ADD COLUMN birth_year INT DEFAULT NULL;         -- 出生年份（可從 birthday 推算，但獨立存方便查詢）
ALTER TABLE user ADD COLUMN school_code VARCHAR(20) DEFAULT NULL; -- 學校代碼（未來跨校研究用）
ALTER TABLE user ADD COLUMN cohort VARCHAR(30) DEFAULT NULL;      -- 研究世代標籤（如 "2026S1" = 2026春季第一批）
```

### 研究匯出時的分群欄位

`trial_log` 和 `game_session` 不需要存年級——透過 `user_id` JOIN `user` 表即可。但匯出 API 支援按年級篩選：

```
GET research_export.php?action=export_trials&grade=1,2&cohort=2026S1
```

### 未來年級擴展時需要的調整

- **難度參數表**：不同年級的數字範圍、起始難度可能不同。目前硬編碼在遊戲腳本中，未來可抽成 config
- **常模（norm）數據**：不同年級的正確率/反應時間基準不同，需要分年級建立常模
- **問卷版本**：PPTQ-C 目前只有一個版本，可能需要年齡適配版本
- 這些都是**之後的事**，本次只需確保 schema 能容納年級欄位

## 不在本次範圍

- 玩家數據查看器的圖表增強（之後另做）
- 能力使用紀錄的實際追蹤（表先建好，實際埋點之後做）
- 跨遊戲的綜合分析 API
- 班級/群組層級的統計報告
- 不同年級的難度參數配置系統

---

## 討論要點

### 1. trial_log 的資料量估算

假設 100 個學生、每人每天玩 10 分鐘（約 60-100 題）、持續 6 週：
- 100 × 80 題 × 30 天 = **240,000 筆/月**
- 半年約 150 萬筆，需要考慮索引效能和定期歸檔策略

### 2. 上傳頻率 vs 資料完整性（已考慮 Game3 節奏 + 中斷場景）

- Game1/2 每題 ~6 秒，5 題批次 = ~30 秒上傳一次 ✓
- Game3 每題 ~20 秒，5 題批次 = ~100 秒才上傳一次 ✗ 風險太高
- **最終方案**：
  - Game1/2：每 5 題批次
  - Game3：每 3 題批次
  - 補充：每 60 秒無論題數強制上傳一次
  - OnApplicationPause(true) 時立即上傳 + 本地備份
  - OnApplicationQuit 時同步上傳 + 本地備份
  - 下次啟動時檢查並補傳上次未完成的會話資料

- **最壞情況分析**：小孩玩到一半家長直接關 app
  - 如果上一次批次上傳在 30 秒前 → 最多丟失 30 秒內的 5 題（Game1/2）或 1-2 題（Game3）
  - 但 OnApplicationPause/Quit 通常會被觸發，有機會搶救
  - 即使完全沒觸發（如 kill -9），下次啟動時從 PlayerPrefs 恢復
  - **真正遺失的條件**：app 被強制殺掉 + 之後從未再打開（極端情況，可接受）

### 3. response_time_ms 的精確度

- Unity 的 `Time.time` 精度取決於幀率（60fps ≈ 16ms 誤差）
- 對於兒童數感研究，16ms 誤差在可接受範圍（兒童 RT 通常 800-3000ms）
- 但需注意：動畫完成的精確時間點要仔細定義（怪物完全落地？倉庫門關上？）

### 4. 隱私與研究倫理

- trial_log 包含詳細的行為數據，需要確保符合研究倫理委員會（IRB）的要求
- user_id 應該是匿名化的（目前看起來已是系統生成的 ID）
- 考慮是否需要資料保留期限（例如研究結束後 N 年刪除）

### 5. 與 Ng et al. (2022) 論文的關係

- 本系統收集的數據遠比論文所使用的更精細：論文只用了「總訓練時數」做分組，本系統記錄到逐題層級
- 論文的核心發現（訓練降低焦慮、提升成就）可以用本系統的數據做更精確的複製與延伸
- **新增分析可能**：
  - 距離效應（distance effect）隨訓練時數的變化 → ANS 精確度是否因訓練提升
  - 不同焦慮特質兒童在 Game3（工作記憶負荷）的表現差異 → 高焦慮兒童是否在高認知負荷任務中表現更差（非即時焦慮測量，而是前測焦慮分組 × 遊戲表現的交叉分析）
  - 學習曲線的個體差異 → 是否存在「快速學習者」和「慢啟動者」等不同學習軌跡類型
  - 訓練強度的最佳閾值 → 論文用 6h 做二分法，本系統可做更細的劑量-反應分析
- **量表問題待討論**：目前 MathAttitude 量表只有 5 題，與論文使用的 CMAQ（8 題）不同。是否需要調整以確保可比較性，需研究團隊決定
