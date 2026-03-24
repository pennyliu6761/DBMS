# 第十二章：規劃與建立索引 (資料庫效能調校與 EXPLAIN 分析)

**授課教師：劉鎮豪**
**課程：資料庫管理系統**
**學校：國立金門大學**

當「金門特產進銷存系統」營運了五年，訂單資料表累積了數百萬筆記錄時，你會發現原本瞬間跑出來的網頁，現在竟然要轉圈圈等上 10 秒鐘！
本章節我們將學習資料庫的效能救星：**索引 (Index)**。我們將探討索引的底層結構 (B-Tree)、如何建立與刪除索引，以及最重要的——如何使用 `EXPLAIN` 指令來剖析資料庫的查詢效能。課程後半段，我們將結合 Streamlit 打造一個專業的「SQL 執行計畫分析儀表板」！

---

## 🛠️ 課前準備：建立「海量訪客日誌表」

為了體會索引的威力，我們需要一張預期會有大量資料的表格。請在 MySQL Workbench 執行以下語法，建立一張特產店網站的「訪客搜尋日誌表」，並匯入測試資料：

  ```sql
  USE kinmen_shop;

  -- 建立訪客搜尋日誌表 (刻意先不加任何自訂索引)
  CREATE TABLE search_logs (
      log_id INT AUTO_INCREMENT PRIMARY KEY,
      user_ip VARCHAR(20) NOT NULL,
      search_keyword VARCHAR(50) NOT NULL,
      search_time DATETIME DEFAULT CURRENT_TIMESTAMP
  );

  -- 模擬匯入多筆搜尋紀錄
  INSERT INTO search_logs (user_ip, search_keyword) VALUES 
  ('192.168.1.1', '高粱酒'), ('10.0.0.5', '貢糖'), 
  ('192.168.1.2', '牛肉乾'), ('172.16.0.8', '高粱酒'),
  ('192.168.1.100', '砲彈鋼刀'), ('10.0.0.12', '麵線'),
  ('192.168.1.1', '貢糖'), ('172.16.0.9', '高粱酒');
  ```

---

## 📖 第一部分：理論與 SQL 實戰 (深度解析)

### 1. 什麼是索引 (Index)？
想像一本 1000 頁的實體字典，如果你要找「金」這個字，不可能是從第一頁逐頁翻找（這在資料庫稱為 **全表掃描 Full Table Scan**）。你會先翻到「目錄 (索引)」，找到「金」在第 500 頁，然後直接翻過去。資料庫的索引就是利用 **B-Tree (B樹)** 或 **Hash** 結構來達成這種極速查詢。

### 2. 建立與刪除索引 (CREATE / DROP INDEX)
MySQL 會自動為 `PRIMARY KEY` (主鍵) 和 `UNIQUE` (唯一值) 建立索引。但對於我們常放在 `WHERE` 條件的普通欄位，我們必須手動建立。

  ```sql
  -- 【建立一般索引】經常有客人搜尋「關鍵字」，我們為 search_keyword 加上索引
  CREATE INDEX idx_keyword ON search_logs(search_keyword);

  -- 【建立複合索引】如果我們常同時用 IP 和 關鍵字 查詢，可以建立複合索引
  CREATE INDEX idx_ip_keyword ON search_logs(user_ip, search_keyword);

  -- 【查詢資料表有哪些索引】
  SHOW INDEX FROM search_logs;

  -- 【刪除索引】發現索引建得不好，或者該欄位不常查了，記得刪除以節省空間
  DROP INDEX idx_ip_keyword ON search_logs;
  ```

> **💼 業界實務補充：什麼時候「不該」建索引？**
> 索引雖然能加速 `SELECT` (讀取)，但會嚴重拖慢 `INSERT`, `UPDATE`, `DELETE` (寫入) 的速度！因為每次新增資料，資料庫都要重新排一次 B-Tree 目錄。
> **不要建索引的時機**：1. 資料量太少的表。 2. 經常頻繁更新的欄位。 3. 重複性極高的欄位（例如：性別只有男/女，建索引完全沒幫助）。

### 3. 資深工程師的照妖鏡：EXPLAIN 執行計畫
當你下了一段查詢，你怎麼知道 MySQL 有沒有「乖乖使用」你建好的索引？只要在 `SELECT` 前面加上 `EXPLAIN` 就能看透一切！

  ```sql
  -- 測試一：用剛剛建好索引的 search_keyword 來查
  EXPLAIN SELECT * FROM search_logs WHERE search_keyword = '高粱酒';
  ```
> **🔍 EXPLAIN 關鍵欄位判讀：**
> * **`type`**：查詢的存取類型。看到 **`ALL`** 代表「全表掃描 (極慢)」；看到 **`ref`** 或 **`const`** 代表「成功使用索引 (極快)」。
> * **`possible_keys`**：MySQL 評估可以用哪些索引。
> * **`key`**：MySQL 最終**實際決定使用**的索引。
> * **`rows`**：MySQL 預估為了找到結果，需要掃描多少筆資料（越少越好）。

---

## 🤖 第二部分：AI 輔助開發 (生成海量測試資料)

只用 8 筆資料是測不出索引威力的。在實務上，我們會寫 Python 腳本或 SQL 預存程序來塞入海量假資料。這次我們直接請 AI 幫忙生出這支程式！

> [!TIP]
> **【給 AI 的 Prompt - 協助撰寫海量假資料生成腳本】**
>
> 「我正在測試 MySQL 的索引效能，需要大量的假資料。
> 請幫我寫一段 Python 程式碼 (使用 `pymysql` 和 `random` 套件)。
> 1. 連線到本地的 `kinmen_shop` 資料庫。
> 2. 使用 `for` 迴圈，隨機生成 10,000 筆資料，`INSERT` 到 `search_logs` 表中。
> 3. `user_ip` 請隨機生成類似 '192.168.1.X' 的字串。
> 4. `search_keyword` 請從 ['高粱酒', '貢糖', '牛肉乾', '麵線', '菜刀'] 這幾個字串中隨機挑選。
> 5. 為了效能，請使用 `cursor.executemany()` 進行批次寫入，最後記得 `conn.commit()`。」

> **🧠 Prompt 解析 (思維訓練)：**
> 在這段指令中，我們特別指定 AI 使用 `executemany()` 而不是在迴圈裡一直 `execute()`。在寫入巨量資料時，批次寫入的速度是單筆寫入的數十倍以上！

---

## 💻 第三部分：Python (Streamlit) DBA 效能分析儀表板

我們將把 `EXPLAIN` 的分析結果視覺化，讓學生在網頁上直接輸入 SQL 語法，並由系統自動判定這段語法是否及格！

請在 Spyder 開啟 `app.py`：

  ```python
  import streamlit as st
  import pymysql
  import pandas as pd

  def get_connection():
      return pymysql.connect(
          host="127.0.0.1", user="root", password="1234", 
          database="kinmen_shop", charset="utf8mb4"
      )

  st.set_page_config(layout="wide")
  st.title("⚡ DBA 效能調校分析儀表板")
  st.markdown("透過 `EXPLAIN` 指令，分析 SQL 語法是否有正確命中索引。")
  st.divider()

  # --- 步驟 1：SQL 語法輸入區 ---
  st.subheader("1. 輸入待測的 SELECT 查詢語法")
  
  # 提供一個預設的查詢語法讓學生測試
  default_query = "SELECT * FROM search_logs WHERE search_keyword = '高粱酒'"
  user_query = st.text_area("SQL 語法 (不需要加上 EXPLAIN，系統會自動加)：", value=default_query, height=100)

  # --- 步驟 2：執行 EXPLAIN 分析 ---
  if st.button("🚀 分析執行計畫", type="primary"):
      if user_query.strip().upper().startswith("SELECT"):
          try:
              conn = get_connection()
              cursor = conn.cursor()
              
              # 在使用者的查詢前加上 EXPLAIN
              explain_query = f"EXPLAIN {user_query}"
              cursor.execute(explain_query)
              result = cursor.fetchall()
              
              if result:
                  columns = [col[0] for col in cursor.description]
                  df = pd.DataFrame(result, columns=columns)
                  
                  st.subheader("2. 執行計畫分析結果")
                  st.dataframe(df, use_container_width=True)
                  
                  # --- 步驟 3：智能判讀 (全端防呆與提示) ---
                  # 抓取第一步(通常是 type)的存取類型
                  access_type = df.iloc[0]['type']
                  used_key = df.iloc[0]['key']
                  scan_rows = df.iloc[0]['rows']
                  
                  st.subheader("3. 系統診斷報告")
                  col1, col2, col3 = st.columns(3)
                  
                  with col1:
                      if access_type == 'ALL':
                          st.error(f"⚠️ 掃描類型：{access_type} (全表掃描)")
                          st.info("診斷：極度危險！系統正在從頭到尾掃描資料，建議建立索引。")
                      else:
                          st.success(f"✅ 掃描類型：{access_type} (索引查詢)")
                          st.info("診斷：效能優良，有正確避開全表掃描。")
                          
                  with col2:
                      if pd.isna(used_key):
                          st.warning("⚠️ 實際使用索引：無 (NULL)")
                      else:
                          st.success(f"✅ 實際使用索引：{used_key}")
                          
                  with col3:
                      st.metric(label="預估掃描列數", value=f"{scan_rows} 筆")
                      
          except Exception as e:
              st.error(f"❌ 語法執行失敗，請檢查 SQL：{e}")
          finally:
              if 'cursor' in locals(): cursor.close()
              if 'conn' in locals(): conn.close()
      else:
          st.warning("⚠️ 本儀表板僅支援 SELECT 查詢的效能分析喔！")
  ```

---

## ❓ 第四部分：常見問題與排解 (FAQ)

1. **Q: 為什麼我明明建了 `search_keyword` 的索引，但用 `LIKE '%貢糖'` 查詢時，EXPLAIN 卻顯示 `type: ALL` (全表掃描)？**
   * A: 這是業界俗稱的「索引失效」大陷阱！當你在 `LIKE` 的**最前面**加上 `%` 時，資料庫無法按字母順序從 B-Tree 找起，只能被迫整本字典從頭翻。**解法：盡量使用 `LIKE '貢糖%'` (字尾模糊)，索引就能正常運作！**
2. **Q: `type` 裡面的 `const`、`ref`、`range` 哪個最快？**
   * A: 速度排序是：`const` (主鍵單筆查詢，最快) > `ref` (非唯一索引查詢) > `range` (範圍查詢，如 `BETWEEN`, `>`) > `ALL` (全表掃描，最慢)。
3. **Q: 我可以幫表格所有的欄位都加上索引嗎？**
   * A: 絕對不行！這會導致這張表在執行 `INSERT` 時慢到讓你懷疑人生，且會吃掉伺服器大量的硬碟空間（因為每一種索引都要額外存一份 B-Tree）。

---

## 📝 第五部分：實戰演練與魔王挑戰

### 🟢 基礎題 (索引建置與分析)
1. **建立索引：** 請在 Workbench 為 `products` 表的 `category` (商品類別) 欄位建立一個名為 `idx_category` 的索引。
2. **效能驗證：** 開啟你剛剛寫好的 Python Streamlit「DBA 分析儀表板」，輸入 `SELECT * FROM products WHERE category = '伴手禮'`，觀察系統診斷報告是否顯示成功使用了 `idx_category`。

### 🟡 進階題 (索引失效地雷)
3. **運算與函數陷阱：** 請在儀表板中測試以下這段語法：
   `SELECT * FROM search_logs WHERE YEAR(search_time) = 2026;`
   你會發現它變成了「全表掃描」！因為**對欄位使用函數 (如 YEAR) 會導致索引直接失效**。
   * **思考題**：請修改上述的 SQL 語法，不要對 `search_time` 使用函數，改用 `>=` 和 `<` 的區間寫法，讓它能成功命中時間索引。

### 😈 魔王挑戰題 (複合索引的「最左前綴法則」)
4. **【深入 B-Tree 的秘密】** * **前置作業**：請在 Workbench 執行 `CREATE INDEX idx_ip_time ON search_logs(user_ip, search_time);`，建立一個結合 IP 與時間的「複合索引」。
   * **挑戰測試 A**：在儀表板輸入 `SELECT * FROM search_logs WHERE user_ip = '10.0.0.5' AND search_time > '2026-01-01'`。觀察是否有用到索引？
   * **挑戰測試 B**：在儀表板輸入 `SELECT * FROM search_logs WHERE search_time > '2026-01-01'` (只用時間查)。觀察是否變成全表掃描？
   * **挑戰解謎**：為什麼同一個複合索引，測試 A 成功，測試 B 卻失敗？請上網搜尋或詢問 AI 什麼是**「複合索引的最左前綴法則 (Leftmost Prefix Rule)」**，並向同學解釋你的發現！
