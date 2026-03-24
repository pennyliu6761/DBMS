# 第十三章：MySQL/MariaDB SQL 程式設計與流程控制 (變數、邏輯判斷與實戰)

**授課教師：劉鎮豪**
**課程：資料庫管理系統**
**學校：國立金門大學**

在過去的章節中，我們學會了如何查詢、新增、修改與刪除資料。但在真實的商業情境中，我們常遇到更複雜的需求，例如：「如果庫存低於 20 就標示為警告」、「如果今天是會員日就打 8 折」。

本章節我們將學習 MySQL/MariaDB 的 **SQL 程式設計** 功能。我們將在 SQL 中引入**變數 (Variables)** 以及 **流程控制 (Flow Control)**，讓資料庫不只是一個「存資料的倉庫」，而是具備邏輯判斷能力的「運算大腦」！

---

## 🛠️ 課前準備：製造測試情境與空值 (NULL)

為了讓稍後的邏輯判斷與空值處理更有感，我們繼續使用 `kinmen_shop` 資料庫中的 `products` (商品表)。請在 MySQL Workbench 執行以下語法，故意製造一些空值與極端的庫存狀態：

  ```sql
  USE kinmen_shop;

  -- 1. 將隱藏版特調的庫存設為 0，價格設為 NULL (測試 IFNULL)
  UPDATE products SET stock = 0, price = NULL WHERE product_id = 'P008';
  
  -- 2. 將砲彈鋼刀的庫存設為 15 (測試低庫存警告)
  UPDATE products SET stock = 15 WHERE product_id = 'P006';

  -- 3. 新增一個測試用的「會員評等」欄位 (測試 ELT 函數)
  ALTER TABLE customers ADD COLUMN rating INT DEFAULT 1;
  UPDATE customers SET rating = 3 WHERE customer_id = 'C001';
  UPDATE customers SET rating = 2 WHERE customer_id = 'C002';
  ```

---

## 📖 第一部分：理論與 SQL 實戰 (深度解析)

### 1. 使用變數 (Variables)
在 SQL 中，我們可以宣告變數並暫存資料。MySQL/MariaDB 中的「使用者定義變數」會以 `@` 開頭。這在做「全館統一折扣」這類情境時非常實用。

  ```sql
  -- 【實戰一：全館動態折扣設定】
  -- 宣告一個變數 @discount_rate 並賦值為 0.85 (85折)
  SET @discount_rate = 0.85;

  -- 在查詢中直接呼叫這個變數參與數學運算！
  SELECT 
      product_name AS '商品名稱', 
      price AS '原價', 
      ROUND(price * @discount_rate) AS '員工優惠價' 
  FROM products;
  ```

### 2. 空值處理函數 (IFNULL)
在程式開發中，任何數字與 `NULL` 進行數學運算，結果必定是 `NULL`。為了避免算不出總價的慘劇，必須使用 `IFNULL()`。

  ```sql
  -- 語法：IFNULL(可能為空的欄位或運算式, 替代值)
  
  -- 錯誤示範：遇到 P008 (價格 NULL)，算出來的總值會變成 NULL
  SELECT product_name, (price * stock) AS '錯誤總值' FROM products;

  -- 正確示範：如果 price 是 NULL，就把它當成 0 來計算！
  SELECT 
      product_name, 
      price,
      (IFNULL(price, 0) * stock) AS '安全計算總值' 
  FROM products;
  ```

### 3. 單一條件判斷 (IF 函數) 與 字串選擇 (ELT 函數)
就像 Python 裡的 `if...else`，MySQL 也提供了單行的 `IF()` 函數。而 `ELT()` 則是 MySQL/MariaDB 特有的超強字串對應函數。

  ```sql
  -- 【實戰二：IF 條件判斷】
  -- 語法：IF(條件運算式, 條件為真時的回傳值, 條件為假時的回傳值)
  SELECT 
      product_name, 
      stock, 
      IF(stock = 0, '🚨 嚴重缺貨', '✅ 供貨正常') AS '庫存狀態'
  FROM products;

  -- 【實戰三：ELT 字串選擇】
  -- 語法：ELT(索引數字, 字串1, 字串2, 字串3...)
  -- 我們利用剛才建立的 rating (1~3) 欄位，直接轉換成中文的會員等級！
  SELECT 
      customer_name, 
      rating,
      ELT(rating, '🥉 銅牌會員', '🥈 銀牌會員', '🥇 金牌會員') AS '會員等級'
  FROM customers;
  ```

### 4. 多重條件判斷 (CASE...WHEN...THEN)

當你的條件超過兩個（例如：庫存充足、低庫存、缺貨），重複使用 `IF()` 會像毛線一樣糾結。這時候業界最愛用的就是 `CASE` 語法！

  ```sql
  -- 【實戰四：精細的庫存紅綠燈系統】
  SELECT 
      product_name, 
      stock,
      CASE 
          WHEN stock > 100 THEN '🟢 庫存充足'
          WHEN stock BETWEEN 1 AND 100 THEN '🟡 庫存緊張(即將售罄)'
          ELSE '🔴 缺貨中'
      END AS '庫存紅綠燈'
  FROM products
  ORDER BY stock ASC;
  ```

> **💼 業界實務補充：為什麼要把邏輯寫在 SQL 裡？**
> 很多初學者會把所有資料 `SELECT *` 撈回 Python，再用 Pandas 慢慢寫 `if-else` 判斷。這在資料只有 10 筆時沒感覺，但當資料有 100 萬筆時，將百萬筆資料塞滿網路頻寬傳到 Python 是極度浪費效能的。將這類單純的列級 (Row-level) 邏輯判斷，交給底層由 C/C++ 撰寫的資料庫引擎 (如 `CASE`)，運算速度會快上數十倍！

---

## 🤖 第二部分：AI 輔助開發 (將複雜商業邏輯轉為 SQL)

在全端開發中，我們可以要求 AI 幫我們把落落長的商業規則，直接翻譯成優雅的 SQL `CASE` 語法。

> [!TIP]
> **【給 AI 的 Prompt - 協助撰寫複雜的 SQL 邏輯】**
>
> 「我正在開發『金門特產店』的資料庫系統。
> 我有一張 `products` 表 (包含 `price` 價格)。
> 為了做行銷活動，我需要在撈出資料的當下，自動幫商品貼上『價格級距標籤』。
> 請幫我寫一段 MySQL 查詢語法，使用 `CASE WHEN` 達成以下邏輯：
> 1. 如果價格大於等於 1000，標籤為『✨ 頂級尊榮』。
> 2. 如果價格介於 300 到 999 之間，標籤為『🎁 經典必買』。
> 3. 如果價格小於 300，標籤為『🍬 銅板小資』。
> 4. 如果價格是 NULL，標籤為『⏳ 敬請期待』。
> 請將這個標籤欄位命名為 `price_level`。」

---

## 💻 第三部分：Python (Streamlit) 動態邏輯儀表板

請在 Spyder 開啟 `app.py`。我們將打造一個結合「MySQL Session 變數 (`@var`)」與「`CASE` 邏輯判斷」的超強後台儀表板，完美展現前後端邏輯分工的藝術。

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
  st.title("🎛️ 特產店動態定價與庫存警報系統")
  st.markdown("展示如何在 Python 中傳遞 SQL 變數，並利用 `CASE` 將商業邏輯實作於資料庫層。")
  st.divider()

  # --- 步驟 1：UI 控制面板 (動態設定折扣) ---
  st.header("1. 🎯 員工購物節：全館動態折扣模擬")
  
  # 使用 Streamlit 滑桿讓店長選擇折扣數
  discount_input = st.slider("請設定今日全館折扣 (%)", min_value=10, max_value=100, value=85, step=5)
  # 將百分比轉為小數 (例如 85 -> 0.85)
  discount_rate = discount_input / 100.0 

  if st.button("套用折扣並預覽價格", type="primary"):
      try:
          conn = get_connection()
          cursor = conn.cursor()
          
          # 【進階技巧：Session 變數傳遞】
          # 先執行 SET 指令設定 MySQL 端的使用者變數
          cursor.execute("SET @py_discount = %s", (discount_rate,))
          
          # 執行 SELECT，直接在 SQL 內運算折扣，並利用 IFNULL 處理空值避免報錯
          query_discount = """
          SELECT 
              product_name AS '商品名稱',
              IFNULL(price, 0) AS '原價',
              ROUND(IFNULL(price, 0) * @py_discount) AS '今日活動價'
          FROM products
          """
          cursor.execute(query_discount)
          result = cursor.fetchall()
          
          if result:
              columns = [col[0] for col in cursor.description]
              df = pd.DataFrame(result, columns=columns)
              st.success(f"✅ 已成功套用 {discount_input} 折優惠！")
              st.dataframe(df, use_container_width=True)
              
      except Exception as e:
          st.error(f"執行失敗：{e}")
      finally:
          if 'cursor' in locals(): cursor.close()
          if 'conn' in locals(): conn.close()

  st.divider()

  # --- 步驟 2：利用 CASE 實作庫存紅綠燈 ---
  st.header("2. 🚦 智能庫存警報器 (SQL CASE 實作)")

  if st.button("掃描全店庫存狀態"):
      try:
          conn = get_connection()
          cursor = conn.cursor()
          
          # 【逐行解析】把複雜的分類邏輯寫在 SQL 的 CASE WHEN 裡面
          # Python 端完全不用寫 if-else，只要負責顯示結果就好！
          query_stock = """
          SELECT 
              product_name AS '商品',
              stock AS '目前庫存量',
              CASE 
                  WHEN stock = 0 THEN '🔴 缺貨中 (請立即叫貨)'
                  WHEN stock BETWEEN 1 AND 50 THEN '🟡 庫存緊張 (建議補貨)'
                  ELSE '🟢 庫存充足'
              END AS '系統建議動作'
          FROM products
          ORDER BY stock ASC
          """
          cursor.execute(query_stock)
          result = cursor.fetchall()
          
          if result:
              columns = [col[0] for col in cursor.description]
              df_stock = pd.DataFrame(result, columns=columns)
              st.dataframe(df_stock, use_container_width=True)
              
      except Exception as e:
          st.error(f"執行失敗：{e}")
      finally:
          if 'cursor' in locals(): cursor.close()
          if 'conn' in locals(): conn.close()
  ```

---

## ❓ 第四部分：常見問題與排解 (FAQ)

1. **Q: `SET @var = 10;` 設定的變數會永遠存在於資料庫嗎？**
   * A: 不會！MySQL 中帶有 `@` 的變數稱為「連線階段變數 (Session Variable)」。只要你關閉了與資料庫的連線（或者 Python 的 `conn.close()` 被執行），這個變數就會立刻像泡沫一樣消失，完全不會影響到其他使用者的查詢。
2. **Q: 為什麼我在 `CASE WHEN` 裡面少寫了 `ELSE`，有些資料會變成 `NULL`？**
   * A: 這是 `CASE` 語法的預設行為。如果一筆資料沒有符合任何一個 `WHEN` 條件，而且你又沒有提供 `ELSE` 做為「兜底方案」，資料庫就會直接回傳 `NULL`。**業界鐵則：寫 CASE 時永遠記得補上 ELSE！**
3. **Q: MySQL 和 MariaDB 在這些程式設計語法上會有差異嗎？**
   * A: 完全沒有！MariaDB 是 MySQL 原創始人建立的開源分支。在我們課程涵蓋的基礎語法、變數 (`@var`)、`IF()`、`ELT()` 與 `CASE` 流程控制中，兩者是 **100% 相容**的。您在 Workbench 寫的語法，可以直接無縫轉移到 MariaDB 執行！

---

## 📝 第五部分：實戰演練與魔王挑戰

為了確保大家精通這幾個超級實用的函數，請在 Workbench 完成以下挑戰！

### 🟢 基礎題 (變數與 IFNULL 練習)
1. **運費全免變數：** 請宣告一個變數 `@shipping_fee` 並設為 `0`。接著寫一段查詢，將所有商品的價格加上這個變數。
2. **空值強迫轉型：** 請利用 `IFNULL`，撈出所有商品的名稱與價格，如果價格是 `NULL`，請把它替換成字串 `'老闆還在想'`。

### 🟡 進階題 (IF 與 ELT 靈活運用)
3. **滿額免運判斷 (`IF`)：** 假設運費是 100 元。請使用 `IF()` 函數寫一段查詢：判斷商品的單價 (請先用 IFNULL 把空值當作 0)，如果單價大於等於 500，運費欄位顯示 `0`，否則顯示 `100`。
4. **星座元素轉換 (`ELT`)：** 假設我們有一個數字欄位代表象限 (1=火, 2=土, 3=風, 4=水)。請利用 `ELT()` 函數，輸入數字 `3`，看看是否能正確回傳字串 `'風象星座'`。

### 😈 魔王挑戰題 (跨表 CASE 動態會員價)
5. **【客製化 VIP 結帳系統】** * **挑戰要求**：我們希望不同的會員等級，買東西有不同的折扣！
   * **前置說明**：我們知道 `customers` 表有 `rating` (1~3)，`products` 表有 `price`。
   * **複雜 SQL 要求**：請在 Workbench 寫一段查詢，先將 `customers` 與 `products` 表（這裡先假設他們想買所有商品，直接 `CROSS JOIN` 或隨便找一筆訂單 `INNER JOIN`）。
   * **核心邏輯**：使用 `CASE` 判斷會員的 `rating`：
     - 若 `rating` 為 3 (金牌)，結帳價格為 `price * 0.7`。
     - 若 `rating` 為 2 (銀牌)，結帳價格為 `price * 0.9`。
     - 若 `rating` 為 1 (銅牌)，結帳價格為原價 `price`。
   * **過關提示**：你必須在 `SELECT` 裡面同時用到 `JOIN` 帶出來的欄位以及 `CASE` 運算式！這段 SQL 如果寫得出來，你已經具備業界後端工程師的基礎實力了！
