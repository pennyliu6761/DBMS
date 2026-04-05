# 實作九：進階查詢與多表關聯 (JOIN、子查詢與 CTE 加長實戰版)

**授課教師：劉鎮豪**
**課程：資料庫管理系統**
**學校：國立金門大學**

當系統越長越大，為了符合「正規化」的要求，資料會被拆散到不同的資料表中。例如：顧客資料在一張表、商品資料在一張表、訂單又在另一張表。
本章節我們將學習資料庫設計中最具價值的核心技術：**多表合併查詢 (JOIN)**、**子查詢 (Subquery)**、**集合運算 (SET)** 與進階的 **CTE (通用資料表運算式)**。課程後半段，我們將利用 Streamlit 打造一個能跨越多張資料表的「VIP 會員消費查詢系統」！

---

## 🛠️ 課前準備：擴充「會員與訂單」關聯資料庫

為了進行多表查詢，我們需要建立全新的關聯表。請在 MySQL Workbench 開啟查詢視窗，執行以下語法建立「會員表」與「訂單明細表」：

  ```sql
  USE kinmen_shop;

  -- 1. 建立會員資料表
  CREATE TABLE customers (
      customer_id VARCHAR(10) PRIMARY KEY,
      customer_name VARCHAR(50) NOT NULL,
      city VARCHAR(20)
  );

  -- 2. 建立訂單明細表 (包含兩個外鍵，分別指向會員與商品)
  CREATE TABLE order_details (
      order_id INT AUTO_INCREMENT PRIMARY KEY,
      customer_id VARCHAR(10),
      product_id VARCHAR(10),
      qty INT,
      order_date DATE,
      FOREIGN KEY (customer_id) REFERENCES customers(customer_id),
      FOREIGN KEY (product_id) REFERENCES products(product_id)
  );

  -- 3. 匯入測試資料
  INSERT INTO customers VALUES 
  ('C001', '王大明', '金門縣'), ('C002', '李小華', '台北市'), 
  ('C003', '陳阿建', '台中市'), ('C004', '林幽靈', '高雄市'); -- 林幽靈目前沒有任何訂單

  INSERT INTO order_details (customer_id, product_id, qty, order_date) VALUES 
  ('C001', 'P001', 2, '2026-05-01'), ('C001', 'P002', 5, '2026-05-02'),
  ('C002', 'P006', 1, '2026-05-03'), ('C003', 'P003', 10, '2026-05-05');
  ```

---

## 📖 第一部分：理論與 SQL 實戰 (深度解析)

### 1. 多資料表合併查詢 (INNER JOIN / LEFT JOIN)


[Image of SQL JOIN types Venn diagram]

合併查詢是將兩張（或多張）正規化分割的資料表，透過「關聯鍵 (通常是 PK 與 FK)」重新縫合起來的技術。

  ```sql
  -- 【INNER JOIN 內部合併】只會撈出「兩邊都有對應到」的資料
  -- 查詢：找出有消費的會員姓名，以及他們買了什麼商品代號
  SELECT c.customer_name, o.product_id, o.qty 
  FROM customers c
  INNER JOIN order_details o ON c.customer_id = o.customer_id;

  -- 【LEFT JOIN 外部合併】以「左表」為主，即使右表沒資料也會顯示 (補 NULL)
  -- 查詢：列出「所有」會員，以及他們的消費紀錄 (沒消費的林幽靈也會出現)
  SELECT c.customer_name, o.product_id 
  FROM customers c
  LEFT JOIN order_details o ON c.customer_id = o.customer_id;
  ```

### 2. 子查詢 (Subquery)
子查詢就是「把一個 SELECT 的結果，當作另一個 SELECT 的條件」。當我們不知道確切數值，必須先讓資料庫算出來時，就會用到子查詢。

  ```sql
  -- 查詢：找出定價大於「全店平均價格」的精緻商品
  SELECT product_name, price 
  FROM products 
  WHERE price > (SELECT AVG(price) FROM products);

  -- 查詢：找出「從來沒有消費過」的會員姓名 (利用 NOT IN 搭配子查詢)
  SELECT customer_name 
  FROM customers 
  WHERE customer_id NOT IN (SELECT DISTINCT customer_id FROM order_details);
  ```

### 3. 集合運算 (UNION)
與 JOIN (橫向增加欄位) 不同，`UNION` 是「直向」將兩筆結構相似的查詢結果上下疊加在一起。

  ```sql
  -- 查詢：舉辦金門特產大展，我們需要將「商品名稱」與「會員姓名」全部列在同一張邀請名單上
  SELECT product_name AS '邀請名單', '商品' AS '身分' FROM products
  UNION
  SELECT customer_name, 'VIP會員' FROM customers;
  ```

### 4. CTE (通用資料表運算式) 與 NULL 處理
CTE 就像是進階版的暫存表，透過 `WITH` 關鍵字，我們可以先把複雜的查詢包裝成一個有名字的虛擬表，讓後續的 SQL 更加好讀。

  ```sql
  -- 利用 IFNULL 處理空值，並利用 CTE 模組化查詢
  -- 查詢：先計算出每個商品的真實售價 (若 price 是 NULL 則當作 0)，再從中挑出大於 500 的商品
  WITH CleanProducts AS (
      SELECT product_name, IFNULL(price, 0) AS actual_price
      FROM products
  )
  SELECT * FROM CleanProducts WHERE actual_price > 500;
  ```

---

## 🤖 第二部分：AI 輔助開發 (複雜多表關聯 UI)

當 SQL 語法變得複雜（包含 JOIN 三張表以上），我們可以請 AI 幫忙規劃 Python 的 UI 邏輯。

> [!TIP]
> **【給 AI 的 Prompt - 協助設計關聯查詢系統】**
>
> 「我正在使用 Python Streamlit 開發『會員消費明細系統』。我的資料庫有 `customers` (會員)、`order_details` (訂單)、`products` (商品) 三張表。
> 1. 請幫我寫一段 Streamlit 程式碼。首先，用 `st.selectbox` 建立一個下拉選單，讓使用者選擇會員姓名 (這裡先用預設陣列 ['王大明', '李小華'] 模擬)。
> 2. 下方建立一個按鈕『查詢該會員消費紀錄』。
> 3. **核心重點：** 當按下按鈕後，請幫我寫一段 Python + MySQL 查詢。你需要使用 `INNER JOIN` 串接這三張表，撈出該名會員買過的『商品名稱 (product_name)』、『數量 (qty)』與『單價 (price)』。
> 4. 請將查詢結果轉為 pandas DataFrame 並顯示在網頁上。」

> **🧠 Prompt 解析 (思維訓練)：**
> 我們刻意將資料庫的 Schema (表名、欄位名) 提供給 AI，這樣 AI 生成出來的 `JOIN` 語法就會直接是我們能用的，不需要我們再手動把 `table1`、`table2` 改掉。

---

## 💻 第三部分：Python (Streamlit) 三表關聯查詢拼裝

請在 Spyder 開啟 `app.py`，我們將把 `customers`, `order_details`, `products` 完美縫合在網頁中！

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
  st.title("🤝 VIP 會員消費明細查詢系統")
  st.markdown("展示 `INNER JOIN` 跨越三張資料表的強大威力。")
  st.divider()

  # --- 步驟 1：動態載入會員選單 ---
  st.header("1. 選擇要查詢的會員")

  # 為了防呆，我們不讓使用者打字，而是從資料庫把「真會員」撈出來變選單
  try:
      conn = get_connection()
      cursor = conn.cursor()
      cursor.execute("SELECT customer_name FROM customers")
      # 將結果轉換為一維陣列：['王大明', '李小華', ...]
      customer_list = [row[0] for row in cursor.fetchall()]
  except Exception as e:
      st.error(f"無法載入會員清單：{e}")
      customer_list = []

  # 建立下拉選單
  selected_customer = st.selectbox("請選擇一位會員：", customer_list)

  # --- 步驟 2：執行三表 JOIN 查詢 ---
  if st.button("查詢消費明細"):
      try:
          # 【逐行解析】這裡示範如何一口氣 JOIN 三張表：
          # 1. 以訂單明細 (o) 為核心
          # 2. 往左連上會員表 (c) 取得姓名
          # 3. 往右連上商品表 (p) 取得商品名稱與價格
          query = """
          SELECT 
              c.customer_name AS '會員姓名',
              p.product_name AS '購買商品',
              p.price AS '商品單價',
              o.qty AS '購買數量',
              (p.price * o.qty) AS '消費小計',
              o.order_date AS '訂單日期'
          FROM order_details o
          INNER JOIN customers c ON o.customer_id = c.customer_id
          INNER JOIN products p ON o.product_id = p.product_id
          WHERE c.customer_name = %s
          ORDER BY o.order_date DESC
          """
          
          cursor.execute(query, (selected_customer,))
          result = cursor.fetchall()
          
          if result:
              columns = [col[0] for col in cursor.description]
              df = pd.DataFrame(result, columns=columns)
              
              st.success(f"✅ 成功撈取 {selected_customer} 的消費紀錄！")
              st.dataframe(df, use_container_width=True)
              
              # 加碼：計算這位會員的總貢獻金額！
              total_spent = df['消費小計'].sum()
              st.info(f"💡 該名會員的歷史總消費金額為：NT$ {total_spent:,}")
          else:
              st.warning("這位會員目前沒有任何消費紀錄喔！(可能是一位潛水會員)")
              
      except Exception as e:
          st.error(f"❌ 查詢失敗：{e}")
      finally:
          if 'cursor' in locals(): cursor.close()
          if 'conn' in locals(): conn.close()
  ```

---

## ❓ 第四部分：常見問題與排解 (FAQ)

1. **Q: 為什麼 JOIN 之後，出現錯誤 `Column 'customer_id' in field list is ambiguous`？**
   * A: 這是 JOIN 最常犯的錯誤（Ambiguous 指的是「模稜兩可」）。因為 `customers` 表和 `order_details` 表都有 `customer_id` 這個欄位，當你 `SELECT customer_id` 時，資料庫不知道你要撈哪張表的！**解法：在欄位前加上表名或別名，例如 `SELECT c.customer_id`。**
2. **Q: 為什麼我的子查詢出現 `Subquery returns more than 1 row` 錯誤？**
   * A: 當你寫 `WHERE customer_id = (SELECT ...)` 時，等號右邊的子查詢「絕對只能吐出一筆資料」。如果你的子查詢吐出了多筆結果，請把等號 `=` 改成 `IN`，例如：`WHERE customer_id IN (SELECT ...)`。

---

## 📝 第五部分：實戰演練與魔王挑戰

### 🟢 基礎題 (JOIN 與子查詢)
1. **基礎關聯：** 請在 Workbench 寫一段 `INNER JOIN` SQL，將 `products` (商品) 與 `order_details` (訂單) 關聯起來，列出「商品名稱」與「被購買的數量」。
2. **單一子查詢：** 老闆想知道誰的消費數量比平均還要高？請先寫一個子查詢算出 `AVG(qty)`，再利用它當條件，找出 `qty > 平均值` 的訂單紀錄。

### 🟡 進階題 (孤兒資料搜尋)
3. **抓出潛水客 (`LEFT JOIN` 應用)：** 在行銷實務上，找出「註冊了卻沒消費」的會員很重要。請寫一段 `LEFT JOIN`，結合 `WHERE ... IS NULL`，找出完全沒有出現在 `order_details` 表中的會員姓名。

### 😈 魔王挑戰題 (CTE + Streamlit 綜合開發)
4. **【金門特產排行榜系統】** * **挑戰要求**：我們要在網頁上做一個「全店商品總銷量排行榜」。
   * **複雜 SQL 要求**：請利用 `WITH` (CTE) 語法。第一步，先在 CTE 內將 `order_details` 的每個 `product_id` 進行 `GROUP BY` 與 `SUM(qty)` 計算出總銷量。第二步，將這個 CTE 虛擬表與 `products` 真實表進行 `INNER JOIN`，抓出「商品名稱」與「總銷量」。
   * **UI 要求**：在 Streamlit 建立一個新區塊，加上一個「顯示前三名熱銷榜」按鈕，按下後將剛剛那段複雜的 CTE 語法執行結果畫成柱狀圖 (`st.bar_chart`)！
