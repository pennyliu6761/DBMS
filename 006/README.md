# 第六章：SQL 基礎查詢與 Python (Streamlit) 全端實作 (深度擴充版)

**授課教師：劉鎮豪**
**課程：資料庫管理系統**
**學校：國立金門大學**

在了解了資料庫的建置與正規化後，我們進入資料庫操作的核心：**查詢 (Query)**。
本章節我們將以「金門特產進銷存系統」為例，深入學習如何使用 `SELECT` 語法從資料庫中精準、高效地撈出需要的資訊。課程後半段，我們將結合 AI 輔助，並使用 Python (Streamlit) 打造出具備多重過濾條件的前端互動網頁。

---

## 🛠️ 課前準備：建立測試資料庫 (Workbench 操作)

在開始練習查詢之前，請先在 MySQL Workbench 開啟新的查詢視窗，建立「金門特產資料表 (`products`)」並匯入測試資料：

  ```sql
  CREATE DATABASE IF NOT EXISTS kinmen_shop;
  USE kinmen_shop;

  CREATE TABLE products (
      product_id VARCHAR(10) PRIMARY KEY,
      product_name VARCHAR(50) NOT NULL,
      category VARCHAR(20),
      price INT,
      stock INT
  );

  INSERT INTO products VALUES 
  ('P001', '特級高粱酒58度', '酒類', 850, 150),
  ('P002', '原味貢糖', '伴手禮', 180, 500),
  ('P003', '花生貢糖', '伴手禮', 190, 320),
  ('P004', '高粱牛肉乾(微辣)', '肉品', 250, 80),
  ('P005', '高粱牛肉乾(原味)', '肉品', 250, 0),
  ('P006', '金門砲彈鋼刀', '工藝品', 1500, 15),
  ('P007', '一條根貼布', '保健品', 120, 1000),
  ('P008', '隱藏版高粱特調', '酒類', NULL, 5); -- 價格未定
  ```

---

## 📖 第一部分：理論與 SQL 實戰 (深度解析)

### 1. 基礎查詢與運算 (SELECT, AS)

除了直接撈取欄位，我們可以在 `SELECT` 階段就讓資料庫幫我們把數學算好，並利用 `AS` 賦予易讀的別名。

  ```sql
  -- 計算每項商品的庫存總價值，並套用 8 折優惠預估
  SELECT 
      product_name AS '商品名稱', 
      price AS '原單價', 
      stock AS '庫存量',
      (price * stock) AS '原庫存總值',
      (price * stock * 0.8) AS '打折後總值'
  FROM products;
  ```
> **💼 業界實務補充：**
> 在業界，強烈建議**不要**使用 `SELECT *`。當資料表有數十個欄位、包含大量圖片路徑或長篇描述時，`SELECT *` 會嚴重消耗網路頻寬。養成「要什麼欄位，就 SELECT 什麼欄位」的習慣，是資深工程師的基本素養。

### 2. 多重條件過濾與陷阱 (WHERE, AND, OR)

  ```sql
  -- 找出屬於「伴手禮」且庫存低於 400 的商品
  SELECT * FROM products WHERE category = '伴手禮' AND stock < 400;
  ```

> **💣 易錯提醒：AND 與 OR 的優先順序陷阱**
> 假設老闆說：「幫我找出**伴手禮或肉品**，而且**庫存大於 100** 的商品」。
> 如果你寫成：`WHERE category = '伴手禮' OR category = '肉品' AND stock > 100`，結果會大錯特錯！因為 SQL 會先執行 `AND` 再執行 `OR`。
> **正確寫法 (必須加括號)：**
> ```sql
> SELECT * FROM products WHERE (category = '伴手禮' OR category = '肉品') AND stock > 100;
> ```

### 3. 進階範圍過濾 (IN, BETWEEN, IS NULL)

為了簡化落落長的 `OR` 條件，SQL 提供了更優雅的寫法：

  ```sql
  -- 使用 IN：等同於 category = '酒類' OR category = '肉品'
  SELECT * FROM products WHERE category IN ('酒類', '肉品');

  -- 使用 BETWEEN：找出價格介於 150 到 300 之間的商品 (包含 150 與 300)
  SELECT * FROM products WHERE price BETWEEN 150 AND 300;

  -- 尋找空值：找出價格尚未定價 (NULL) 的商品
  SELECT * FROM products WHERE price IS NULL;
  ```

### 4. 模糊搜尋與筆數限制 (LIKE, ORDER BY, LIMIT)

  ```sql
  -- 模糊搜尋：% 代表任意長度的字元
  SELECT * FROM products WHERE product_name LIKE '%牛肉乾%';

  -- 排序與限制：找出全店「最貴的 3 樣」商品
  -- ORDER BY DESC (由大到小)，LIMIT 3 (只抓前三筆)
  SELECT product_name, price FROM products ORDER BY price DESC LIMIT 3;
  ```

---

## 🤖 第二部分：AI 輔助開發 (多重條件 UI 設計)

全端開發中，我們常需要複雜的過濾介面。這次我們不只要「搜尋框」，還要加上「價格滑桿」。來看看怎麼請 AI 幫忙生出這個架構！

> [!TIP]
> **【給 AI 的 Prompt - 複合式條件前端介面】**
>
> 「你是一位資深的網頁前端工程師。我正在設計『金門特產進銷存系統』。
> 1. 請幫我寫一個單一檔案的 HTML (包含 CSS/JS)。
> 2. **核心 UI：** 畫面上方需要有一個『控制面板』，裡面包含兩個元件：
>    - 一個文字輸入框 (用於輸入商品名稱)。
>    - 一個範圍滑桿 Range Slider (用於選擇最低與最高價格，範圍 0~2000)。
> 3. **核心功能：** 下方有一個商品表格。當我輸入文字或拉動滑桿時，請用 JavaScript 實作『即時過濾』功能，隱藏不符合名稱或不在價格區間內的商品。
> 4. 請加上精美的 CSS 樣式。」

> **🧠 Prompt 解析 (思維訓練)：**
> 在這個提示詞中，我們明確定義了「兩個元件 (文字 + 滑桿)」以及它們必須「同時連動 (即時過濾)」。把需求拆解得越細，AI 產生的 UI 結構就會越符合我們後續串接 Python 的邏輯。

---

## 💻 第三部分：Python (Streamlit) 全端系統拼裝

我們要把剛剛學到的 SQL `WHERE`, `LIKE`, `BETWEEN` 搬到 Python 網頁中。請在 Spyder 開啟 `app.py`。

### 步驟 1：基礎骨架與連線

  ```python
  import streamlit as st
  import pymysql
  import pandas as pd

  # --- 共用連線設定 ---
  def get_connection():
      return pymysql.connect(
          host="127.0.0.1", user="root", password="1234", 
          database="kinmen_shop", charset="utf8mb4"
      )

  st.title("🛒 金門在地特產進銷存系統")
  st.markdown("結合 SQL `WHERE` 條件與 Streamlit 互動元件。")
  st.divider()
  ```

### 步驟 2：類別下拉選單 (單一條件過濾)

  ```python
  st.header("1. 依商品類別過濾")

  # 建立下拉選單防呆
  selected_category = st.selectbox(
      "請選擇要查看的商品類別：", 
      ["酒類", "伴手禮", "肉品", "工藝品", "保健品"]
  )

  if st.button("查詢該類別商品"):
      try:
          conn = get_connection()
          cursor = conn.cursor()
          
          # 【安全防護】使用 %s 參數化查詢，防止 SQL Injection
          query = "SELECT * FROM products WHERE category = %s"
          cursor.execute(query, (selected_category,))
          result = cursor.fetchall()
          
          if result:
              columns = [col[0] for col in cursor.description]
              df = pd.DataFrame(result, columns=columns)
              st.success(f"✅ 找到 {len(df)} 筆【{selected_category}】商品！")
              st.dataframe(df, use_container_width=True)
          else:
              st.warning("該類別目前沒有商品。")
              
      except Exception as e:
          st.error(f"❌ 查詢失敗：{e}")
      finally:
          if 'cursor' in locals(): cursor.close()
          if 'conn' in locals(): conn.close()
  ```

### 步驟 3：複合條件進階搜尋 (LIKE + BETWEEN + 雙向滑桿)

這是一個進階的 UI 設計，結合了文字輸入與 Streamlit 超酷的雙向滑桿 (`st.slider`)！

  ```python
  st.divider()
  st.header("2. 進階商品搜尋 (關鍵字 + 價格區間)")

  # 建立兩欄式排版
  col1, col2 = st.columns(2)

  with col1:
      # 文字輸入框
      search_keyword = st.text_input("請輸入商品關鍵字：", placeholder="例如: 貢糖")

  with col2:
      # 雙向價格滑桿 (回傳一個包含最小值與最大值的 Tuple)
      min_price, max_price = st.slider(
          "請選擇價格區間：", 
          min_value=0, max_value=2000, value=(0, 1000), step=50
      )

  if st.button("執行進階搜尋"):
      try:
          conn = get_connection()
          cursor = conn.cursor()
          
          # 【重點語法】結合 LIKE 與 BETWEEN 的複合查詢
          # IFNULL(price, 0) 是為了防止 NULL 價格在 BETWEEN 比對時產生錯誤
          query = """
          SELECT * FROM products 
          WHERE product_name LIKE %s 
          AND IFNULL(price, 0) BETWEEN %s AND %s
          ORDER BY price ASC
          """
          
          # 依序塞入三個 %s 變數：關鍵字、最低價、最高價
          cursor.execute(query, (f"%{search_keyword}%", min_price, max_price))
          result = cursor.fetchall()
          
          if result:
              columns = [col[0] for col in cursor.description]
              df = pd.DataFrame(result, columns=columns)
              st.balloons() # 滿滿成就感的特效
              st.success(f"✅ 成功找到 {len(df)} 筆符合條件的商品！")
              st.dataframe(df, use_container_width=True)
          else:
              st.warning("找不到符合所有條件的商品，請放寬搜尋範圍。")
              
      except Exception as e:
          st.error(f"❌ 搜尋失敗：{e}")
      finally:
          if 'cursor' in locals(): cursor.close()
          if 'conn' in locals(): conn.close()
  ```

---

## ❓ 第四部分：常見問題與排解 (FAQ)

1. **Q: `LIKE '%貢糖'` 和 `LIKE '%貢糖%'` 有什麼差別？**
   * A: `%` 代表任意字元。`%貢糖` 只能找到以貢糖**結尾**的字（如：原味貢糖），找不到「貢糖冰淇淋」。`%貢糖%` 則是只要中間有包含這兩個字就會被撈出來，實務上最常用。
2. **Q: 為什麼我的網頁畫面一直轉圈圈，或者按了按鈕沒反應？**
   * A: 通常是因為你前一次執行時發生了錯誤，導致 `conn.close()` 沒有被正確執行，資料庫連線卡死了。請重啟 Anaconda Prompt，或者確認程式碼的 `finally` 區塊有正確縮排。
3. **Q: 為什麼我用 `WHERE price = NULL` 找不到第八筆資料？**
   * A: 記住！在關聯式資料庫中，判定空值**只能用 `IS NULL`**，絕對不可以使用等號。

---

## 📝 第五部分：實戰演練與魔王挑戰

### 🟢 基礎題 (熟悉語法)
1. 請在 Workbench 寫一段 SQL，撈出所有「單價大於 200」且「庫存大於 0」的商品。
2. 請寫一段 SQL，找出分類是 `('肉品', '保健品', '酒類')` 其中之一，且名稱中包含「高粱」的商品。（提示：請混用 `IN` 與 `LIKE`）

### 🟡 進階題 (網頁功能擴充)
3. **分頁與名次表**：請在 Streamlit 的 `app.py` 中新增一個區塊。這個區塊沒有輸入框，只有一個按鈕。按下後，請用 SQL 撈出「全店庫存量最少的前 3 名商品」(提示：使用 `ORDER BY ... ASC LIMIT 3`)，提醒店長準備進貨。

### 😈 魔王挑戰題 (開放式解題)
4. **【全動態過濾器】** 目前的「區塊 1 (下拉選單)」只能搜類別，「區塊 2 (滑桿)」只能搜價格與名稱。
   * **挑戰任務**：請在網頁的側邊欄 (`st.sidebar`) 設計一個「終極過濾器」，同時包含：文字框、類別下拉選單、價格滑桿。
   * **難點**：當使用者「沒有輸入關鍵字」或「類別選擇了『全部』」時，你的 Python 程式必須能動態判斷，不把這些條件加入 SQL 的 `WHERE` 子句中。（提示：你可以嘗試請 AI 教你如何用 Python 動態拼接 SQL 字串，或是傳入預設變數！）
