# 第八章：進階查詢與聚合分析 (從 SQL 統計到商業儀表板)

**授課教師：劉鎮豪**
**課程：資料庫管理系統**
**學校：國立金門大學**

當我們的「金門在地特產進銷存系統」上線一段時間後，資料庫裡會累積大量的交易紀錄。這時候，老闆想看的已經不是「某一筆資料」，而是「整體的營運狀況」。

本章節我們將進入資料分析的核心領域：**聚合函數 (Aggregate Functions)** 與 **群組查詢 (GROUP BY)**。課程後半段，我們將利用 AI 輔助，將這些冷冰冰的統計數字，透過 Python (Streamlit) 轉化為酷炫的商業儀表板 (Dashboard)！

---

## 🛠️ 課前準備：建立「特產店銷售紀錄表」

為了進行統計分析，請先在 MySQL Workbench 開啟查詢視窗，建立一張全新的銷售紀錄表 (`sales_records`)，並匯入近期的觀光季模擬營收：

  ```sql
  USE kinmen_shop;

  -- 建立銷售紀錄表
  CREATE TABLE sales_records (
      record_id INT AUTO_INCREMENT PRIMARY KEY,
      product_name VARCHAR(50) NOT NULL,
      category VARCHAR(20) NOT NULL,
      sale_price INT NOT NULL,
      qty INT NOT NULL,
      sale_date DATE NOT NULL
  );

  -- 匯入金門觀光季銷售數據
  INSERT INTO sales_records (product_name, category, sale_price, qty, sale_date) VALUES 
  ('特級高粱酒58度', '酒類', 850, 5, '2026-04-01'),
  ('原味貢糖', '伴手禮', 180, 20, '2026-04-01'),
  ('高粱牛肉乾(微辣)', '肉品', 250, 10, '2026-04-02'),
  ('特級高粱酒58度', '酒類', 850, 2, '2026-04-03'),
  ('金門砲彈鋼刀', '工藝品', 1500, 1, '2026-04-03'),
  ('花生貢糖', '伴手禮', 190, 15, '2026-04-04'),
  ('高粱牛肉乾(原味)', '肉品', 250, 8, '2026-04-05'),
  ('一條根貼布', '保健品', 120, 30, '2026-04-05');
  ```

---

## 📖 第一部分：理論與 SQL 實戰 (深度解析)

### 1. 聚合函數的摘要查詢 (SUM, AVG, COUNT, MAX, MIN)

聚合函數能將「多筆記錄」壓縮運算成「單一統計值」。

  ```sql
  -- 統計本月總共賣出幾「筆」訂單？(COUNT)
  SELECT COUNT(record_id) AS '總訂單筆數' FROM sales_records;

  -- 統計全店的「總營業額」？(SUM)
  SELECT SUM(sale_price * qty) AS '總營業額' FROM sales_records;

  -- 找出客單價「最高」與「最低」的商品售價？(MAX, MIN)
  SELECT MAX(sale_price) AS '最高單價', MIN(sale_price) AS '最低單價' FROM sales_records;
  ```

### 2. 群組查詢 (GROUP BY) 與小計 (WITH ROLLUP)

如果老闆問：「那『各個類別』的總銷量是多少？」我們就必須使用 `GROUP BY` 把資料分門別類，再進行聚合計算。

  ```sql
  -- 依照「商品類別」分組，計算各類別的總銷售數量
  SELECT category AS '商品類別', SUM(qty) AS '總銷售量'
  FROM sales_records
  GROUP BY category;
  ```

> **💼 業界實務補充：超強大的 `WITH ROLLUP`**
> 在業界產出財務報表時，我們不只要看「各類別的小計」，還要看最下面的「總計」。MySQL 提供了 `WITH ROLLUP` 關鍵字，能自動幫你在最後一行補上總和！
> ```sql
> SELECT category AS '商品類別', SUM(qty) AS '總銷售量'
> FROM sales_records
> GROUP BY category WITH ROLLUP;
> ```

### 3. 群組後的條件過濾 (HAVING) 與排序限制 (ORDER BY & LIMIT)

  ```sql
  -- 找出總銷售量「大於 10」的熱銷類別，並依照銷量由高到低排序
  SELECT category, SUM(qty) AS total_qty
  FROM sales_records
  GROUP BY category
  HAVING SUM(qty) > 10
  ORDER BY total_qty DESC;
  ```

> **💣 易錯提醒：`WHERE` 與 `HAVING` 到底差在哪？**
> 這絕對是期中考的必考題！
> * `WHERE`：在資料**分組前**進行過濾（剔除不要的原始資料）。
> * `HAVING`：在資料**分組並計算完（如 SUM 之後）**，對「群組的統計結果」進行過濾。
> * **絕對不能寫成**：`WHERE SUM(qty) > 10`，資料庫會直接報錯！

---

## 🤖 第二部分：AI 輔助開發 (商業儀表板 UI 設計)

要呈現統計數字，表格太單調了。我們來請 AI 幫忙規劃一個現代化的商業儀表板 (Dashboard) 介面！

> [!TIP]
> **【給 AI 的 Prompt - 協助設計 Streamlit 儀表板排版】**
>
> 「我正在使用 Python Streamlit 開發『金門特產銷售儀表板』。
> 1. 請幫我利用 `st.columns` 在網頁最上方建立三個『指標卡片 (Metric Cards)』，用來顯示：總營業額、總訂單數、最熱銷類別 (先給假資料即可，使用 `st.metric`)。
> 2. 在指標卡片下方，建立兩個區塊：左邊區塊預留放『長條圖』，右邊區塊預留放『詳細數據表格』。
> 3. 請加上分隔線與適當的標題。只給我排版架構的程式碼就好。」

> **🧠 Prompt 解析 (思維訓練)：**
> Streamlit 雖然寫起來像一般的 Python 腳本，但透過 `st.columns` 可以做出非常專業的排版。讓 AI 先幫我們把「空殼」架好，我們後續再把 SQL 撈出來的真資料填進去，能大幅降低開發的挫折感。

---

## 💻 第三部分：Python (Streamlit) 全端儀表板拼裝

請在 Spyder 開啟 `app.py`。我們要把 SQL 算出來的統計資料，變成網頁上的大字報與圖表！

### 步驟 1：建立連線與儀表板大標題

  ```python
  import streamlit as st
  import pymysql
  import pandas as pd

  def get_connection():
      return pymysql.connect(
          host="127.0.0.1", user="root", password="1234", 
          database="kinmen_shop", charset="utf8mb4"
      )

  # 設定網頁為寬版顯示 (讓圖表更大更清楚)
  st.set_page_config(layout="wide")
  st.title("📈 金門特產營運分析儀表板")
  st.divider()
  ```

### 步驟 2：聚合函數的應用 (指標卡片)

我們透過 `SUM()` 和 `COUNT()` 撈出全店總計，並顯示在頂部卡片上。

  ```python
  st.header("1. 營運總覽 (Aggregate Functions)")

  try:
      conn = get_connection()
      cursor = conn.cursor()
      
      # 同時計算總營業額與總訂單數
      query_total = "SELECT SUM(sale_price * qty), COUNT(record_id) FROM sales_records"
      cursor.execute(query_total)
      total_revenue, total_orders = cursor.fetchone() # fetchone 只抓第一筆(唯一一筆)結果
      
      # 建立三個排版直欄
      col1, col2, col3 = st.columns(3)
      
      with col1:
          st.metric(label="💰 本月總營業額", value=f"NT$ {total_revenue:,}")
      with col2:
          st.metric(label="📦 總訂單筆數", value=f"{total_orders} 筆")
      with col3:
          st.metric(label="🏆 顧客平均客單價", value=f"NT$ {round(total_revenue/total_orders):,}")
          
  except Exception as e:
      st.error(f"資料讀取失敗：{e}")
  finally:
      if 'cursor' in locals(): cursor.close()
      if 'conn' in locals(): conn.close()
  ```

### 步驟 3：群組查詢與資料視覺化 (GROUP BY)

利用 `GROUP BY` 抓出各類別銷量，並直接餵給 Streamlit 畫出長條圖！

  ```python
  st.divider()
  st.header("2. 各類別銷售分析 (GROUP BY & 圖表)")

  try:
      conn = get_connection()
      cursor = conn.cursor()
      
      # 撈取各類別的總營業額，並由大到小排序
      query_group = """
      SELECT category AS '商品類別', SUM(sale_price * qty) AS '類別總營業額'
      FROM sales_records
      GROUP BY category
      ORDER BY 類別總營業額 DESC
      """
      cursor.execute(query_group)
      result = cursor.fetchall()
      
      if result:
          # 將資料轉為 DataFrame
          columns = [col[0] for col in cursor.description]
          df = pd.DataFrame(result, columns=columns)
          
          # 建立左右兩欄排版 (圖表在左 70%，表格在右 30%)
          chart_col, table_col = st.columns([0.7, 0.3])
          
          with chart_col:
              st.subheader("📊 類別營收佔比")
              # 將 X 軸設為類別名稱，Streamlit 才能正確畫出長條圖
              df_chart = df.set_index('商品類別')
              st.bar_chart(df_chart)
              
          with table_col:
              st.subheader("📋 詳細數據")
              st.dataframe(df, use_container_width=True)
              
  except Exception as e:
      st.error(f"資料讀取失敗：{e}")
  finally:
      if 'cursor' in locals(): cursor.close()
      if 'conn' in locals(): conn.close()
  ```

---

## ❓ 第四部分：常見問題與排解 (FAQ)

1. **Q: 為什麼我用了 `GROUP BY`，但 `SELECT` 裡面放 `product_name` 會報錯？**
   * A: 這是群組查詢的鐵則！當你 `GROUP BY category` (依類別分組) 時，一個「肉品」類別裡可能包含「微辣牛肉乾」和「原味牛肉乾」。資料庫不知道該顯示哪一個名稱，所以會報錯。**`SELECT` 裡面只能放「分組的依據欄位」或是「聚合函數 (如 SUM, AVG)」！**
2. **Q: `fetchone()` 和 `fetchall()` 差別在哪？**
   * A: 當你使用沒有 `GROUP BY` 的聚合函數 (如 `SELECT SUM(...)`)，結果絕對只有一行，這時用 `fetchone()` 直接抓那一列最方便；如果是撈清單，就必須用 `fetchall()`。

---

## 📝 第五部分：實戰演練與魔王挑戰

### 🟢 基礎題 (聚合計算)
1. 請在 Workbench 寫一段 SQL，計算所有「肉品」的**平均售價 (AVG)**。
2. 請寫一段 SQL，找出 `sales_records` 中，哪一天的銷售營業額最高？(提示：使用 `GROUP BY sale_date` 搭配 `ORDER BY ... DESC LIMIT 1`)

### 🟡 進階題 (WITH ROLLUP 的威力)
3. 請在 Workbench 執行以下語法，並觀察最後一行的結果：
   ```sql
   SELECT category, SUM(qty) FROM sales_records GROUP BY category WITH ROLLUP;
   ```
   請嘗試將這段 SQL 替換到你的 Python 程式中，看看 Streamlit 的圖表和表格會發生什麼有趣的變化（或錯誤）？並思考在程式開發時，該如何處理這筆「多出來的總計列」。

### 😈 魔王挑戰題 (互動式 HAVING 過濾)
4. **【滯銷品揪察隊】** * **挑戰要求**：在 Streamlit 網頁側邊欄 (`st.sidebar`) 加上一個「數值輸入框 (`st.number_input`)」，標題為「設定熱銷門檻數量 (預設 10)」。
   * **核心功能**：請透過變數將這個數值動態傳入 SQL 的 `HAVING SUM(qty) > %s` 中。當使用者調整側邊欄的數字時，右邊的圖表與表格要即時更新，只顯示達到門檻的商品類別！
   * **過關提示**：結合 Streamlit UI 變數與 pymysql 的參數化查詢 `%s` 即可完美解題！
