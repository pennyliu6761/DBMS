# 第十一章：檢視表的建立與應用 (虛擬資料表與資料安全)

**授課教師：劉鎮豪**
**課程：資料庫管理系統**
**學校：國立金門大學**

隨著「金門特產進銷存系統」的業務擴大，我們聘請了許多工讀生來協助結帳。但問題來了：我們不希望工讀生看到商品的「進貨成本」，也不想讓他們面對複雜的多表 `JOIN` 查詢。這時候該怎麼辦？

本章節我們將學習強大的 **檢視表 (View)** 功能。檢視表就像是一扇「特製的窗戶」，它不實際儲存資料，而是將複雜或敏感的基底資料表 (Base Tables) 重新包裝，提供一個乾淨、安全且簡化的資料視角！

---

## 🛠️ 課前準備：為商品表加上「機密成本」

為了展示檢視表的「隱藏欄位」功能，請先在 MySQL Workbench 執行以下語法，為我們的 `products` 表加上一個代表進貨成本的欄位：

  ```sql
  USE kinmen_shop;

  -- 新增進貨成本欄位，並隨機更新一些假資料
  ALTER TABLE products ADD COLUMN cost INT;
  UPDATE products SET cost = price * 0.5; -- 假設成本是售價的一半
  ```

---

## 📖 第一部分：理論與 SQL 實戰 (深度解析)

### 1. 建立檢視表 (CREATE VIEW)
檢視表本身不佔用硬碟空間存資料，它儲存的其實是「一段 SQL 查詢語法」。

  ```sql
  -- 【實戰一：隱藏機密欄位】
  -- 建立一個只給工讀生看的「安全檢視表」，把 cost (成本) 欄位藏起來
  CREATE VIEW view_safe_products AS
  SELECT product_id, product_name, category, price, stock 
  FROM products;

  -- 以後工讀生只要查詢這個 View，就像在查一般的 Table 一樣！
  SELECT * FROM view_safe_products;
  ```

> **💼 業界實務補充 (與 AI/ML 結合)：**
> 在工業工程與資料科學領域，我們常需要乾淨的 Dataset 來訓練機器學習模型。資料工程師通常會預先寫好一個包含極度複雜 JOIN 與資料清洗的 `VIEW`，資料科學家只要下 `SELECT * FROM clean_dataset_view` 就能直接拿去跑 Python 建模，大幅降低跨部門的溝通成本！

  ```sql
  -- 【實戰二：簡化複雜查詢】
  -- 將跨越三張表 (會員、訂單、商品) 的複雜 JOIN 寫成一個 View
  CREATE VIEW view_customer_orders AS
  SELECT c.customer_name, p.product_name, o.qty, o.order_date
  FROM order_details o
  INNER JOIN customers c ON o.customer_id = c.customer_id
  INNER JOIN products p ON o.product_id = p.product_id;

  -- 查詢時瞬間變得超級簡單！
  SELECT * FROM view_customer_orders WHERE customer_name = '王大明';
  ```

### 2. 修改與刪除檢視表 (ALTER / DROP VIEW)
檢視表的修改與刪除非常輕量，因為它只是刪除或覆寫一段查詢邏輯，不會影響到底層的真實資料。

  ```sql
  -- 修改檢視表 (例如：工讀生也不需要看到 stock 庫存欄位了)
  ALTER VIEW view_safe_products AS
  SELECT product_id, product_name, category, price 
  FROM products;

  -- 刪除檢視表
  DROP VIEW IF EXISTS view_safe_products;
  ```

### 3. 編輯檢視表內容與防呆機制 (WITH CHECK OPTION)
你可以透過檢視表來 `INSERT` 或 `UPDATE` 資料（這些異動會穿透回原始的 Base Table）。但為了防止資料錯亂，我們可以使用 `WITH CHECK OPTION` 來嚴格限制寫入規則！

  ```sql
  -- 建立一個「平價商品檢視表」(價格必須小於等於 200)
  CREATE VIEW view_budget_items AS
  SELECT * FROM products WHERE price <= 200
  WITH CHECK OPTION; -- 加上這句，開啟終極防護！

  -- ⭕ 成功：透過 View 新增一個 150 元的商品
  INSERT INTO view_budget_items (product_id, product_name, price) 
  VALUES ('P010', '小包裝貢糖', 150);

  -- ❌ 失敗報錯：企圖透過此 View 新增一個 500 元的商品
  -- 因為違反了 WHERE price <= 200 的條件，會被 WITH CHECK OPTION 擋下！
  INSERT INTO view_budget_items (product_id, product_name, price) 
  VALUES ('P011', '豪華高粱', 500); 
  ```

---

## 🤖 第二部分：AI 輔助開發 (設計 View 的專屬 UI)

有了 View，我們前端 Python 的 SQL 語法就能大幅縮短。請 AI 幫我們設計一個「工讀生專用 POS 機」介面。

> [!TIP]
> **【給 AI 的 Prompt - 協助設計 Streamlit 安全存取介面】**
>
> 「我正在使用 Python Streamlit 開發『特產店工讀生結帳系統』。我的資料庫裡有一個叫做 `view_safe_products` 的檢視表 (包含 id, name, price)。
> 1. 請幫我規劃一個介面。左半邊顯示 `view_safe_products` 的所有商品資料表格 (作為商品型錄)。
> 2. 右半邊規劃一個『新增平價商品』的表單 (`st.form`)，包含：商品代號、商品名稱、商品價格 (數字輸入框)。
> 3. 當送出表單時，請幫我寫 Python 程式將資料 `INSERT` 到 `view_budget_items` 這個檢視表中。
> 4. **核心防呆：** 由於 `view_budget_items` 設有 `WITH CHECK OPTION`，如果價格大於 200 會拋出 `pymysql.err.OperationalError` (Error Code 1369)，請在 Except 區塊中精準捕捉這個錯誤，並顯示警告：『超出平價商品定價範圍，拒絕新增！』。」

> **🧠 Prompt 解析 (思維訓練)：**
> 告訴 AI 我們正在操作的是 View 而不是 Table，並明確指出會觸發的 Error Code (`1369 CHECK OPTION failed`)。這樣 AI 產出的 Exception 捕捉程式碼就會非常精準，省去我們大量除錯的時間。

---

## 💻 第三部分：Python (Streamlit) 全端檢視表應用

請在 Spyder 開啟 `app.py`。我們將驗證在 Python 裡呼叫 View 是多麼輕鬆的一件事，並體驗 `WITH CHECK OPTION` 的保護力。

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
  st.title("🛡️ 前台工讀生安全操作系統 (檢視表應用)")
  st.markdown("透過 MySQL `VIEW` 保護進貨成本，並展示 `WITH CHECK OPTION` 的防護威力。")
  st.divider()

  # 建立左右兩欄排版
  col_left, col_right = st.columns([0.6, 0.4])

  # --- 左側區塊：讀取安全檢視表 ---
  with col_left:
      st.subheader("📋 商品型錄 (隱藏成本)")
      try:
          conn = get_connection()
          # 💡 在 Python 中，讀取 View 跟讀取 Table 的語法一模一樣！
          # 但此時前端絕對拿不到 cost (成本) 的資料，資安滿分。
          query = "SELECT * FROM view_safe_products"
          df = pd.read_sql(query, conn)
          st.dataframe(df, use_container_width=True)
      except Exception as e:
          st.error(f"讀取型錄失敗：{e}")
      finally:
          if 'conn' in locals(): conn.close()

  # --- 右側區塊：寫入帶有限制條件的檢視表 ---
  with col_right:
      st.subheader("➕ 新增平價專區商品")
      st.info("💡 測試：試著輸入超過 200 元的價格看看！")
      
      with st.form("budget_form"):
          new_id = st.text_input("商品代號 (例如: P010)")
          new_name = st.text_input("商品名稱")
          new_price = st.number_input("商品售價", min_value=0, step=10, value=150)
          
          submit_btn = st.form_submit_button("新增至平價檢視表", type="primary")
          
          if submit_btn:
              if new_id and new_name:
                  try:
                      conn = get_connection()
                      cursor = conn.cursor()
                      
                      # 企圖寫入 view_budget_items (內含 price <= 200 的限制)
                      insert_query = "INSERT INTO view_budget_items (product_id, product_name, price) VALUES (%s, %s, %s)"
                      cursor.execute(insert_query, (new_id, new_name, new_price))
                      conn.commit()
                      
                      st.success(f"✅ 成功新增平價商品：{new_name}！請重新整理左側型錄。")
                      
                  except pymysql.err.OperationalError as e:
                      # 【精準錯誤捕捉】1369 代表違反了 View 的 WITH CHECK OPTION
                      if e.args[0] == 1369:
                          st.error("❌ 新增失敗：此商品定價超過 200 元，違反平價檢視表規則，資料庫拒絕寫入！")
                      else:
                          st.error(f"❌ 發生操作錯誤：{e}")
                  except Exception as e:
                      st.error(f"❌ 系統錯誤：{e}")
                  finally:
                      if 'cursor' in locals(): cursor.close()
                      if 'conn' in locals(): conn.close()
              else:
                  st.warning("請填寫完整商品資訊！")
  ```

---

## ❓ 第四部分：常見問題與排解 (FAQ)

1. **Q: `VIEW` 會不會佔用資料庫的硬碟容量？**
   * A: 不會！檢視表被稱為「虛擬表」，它只儲存 SQL 查詢指令。每次你 `SELECT` 檢視表時，MySQL 才會即時去底層的真實資料表 (Base Table) 撈取並運算資料。
2. **Q: 任何 `VIEW` 都可以被 `INSERT` 或 `UPDATE` 嗎？**
   * A: 不是的！如果你的 `VIEW` 裡面包含了 `GROUP BY` (群組)、`SUM/AVG` (聚合函數)、`DISTINCT` (去重) 或是複雜的 `UNION`，這個 View 就是**唯讀 (Read-Only)** 的。因為資料庫無法將「一筆修改」反向推導回原本的多筆原始資料。
3. **Q: `WITH CHECK OPTION` 可以不寫嗎？**
   * A: 可以。如果不寫，你其實可以透過 `view_budget_items` 成功新增一個 500 元的商品。但詭異的是，新增成功後，你從這個 View 卻**看不到**剛剛新增的商品（因為它被 WHERE price <= 200 過濾掉了）。這會造成系統邏輯混亂，因此強烈建議加上這個防呆機制！

---

## 📝 第五部分：實戰演練與魔王挑戰

### 🟢 基礎題 (建立與修改 View)
1. **建立高價品檢視表：** 請在 Workbench 中，建立一個名為 `view_premium_products` 的檢視表，專門用來顯示單價大於 800 元的商品，並只顯示「商品代號」與「商品名稱」兩個欄位。
2. **修改檢視表：** 請使用 `ALTER VIEW`，將剛才的高價品檢視表加上 `price` (單價) 欄位。

### 🟡 進階題 (複雜查詢模組化)
3. **部門業績視角：** 假設我們常常需要查「每個類別 (category) 的總庫存價值 (price * stock)」。每次都要寫 `GROUP BY` 很麻煩。請建立一個檢視表 `view_category_value`，直接包含 `category` 與 `total_value` 兩個欄位。
4. **驗證唯讀性：** 建立完成後，請試著執行 `UPDATE view_category_value SET total_value = 9999 WHERE category = '酒類';`，觀察 Workbench 報出的錯誤訊息，驗證帶有聚合運算的 View 是唯讀的。

### 😈 魔王挑戰題 (全端權限切換系統)
5. **【店長 / 工讀生 視角切換器】** * **挑戰要求**：在 Streamlit 網頁側邊欄 (`st.sidebar`) 加上一個單選按鈕 (`st.radio`)，讓使用者切換身分為「店長」或「工讀生」。
   * **核心邏輯**：
      - 當選擇「店長」時，Python 程式去 `SELECT * FROM products` (顯示最原始、包含 cost 成本欄位的機密大表)。
      - 當選擇「工讀生」時，Python 程式改去 `SELECT * FROM view_safe_products` (只顯示安全檢視表)。
   * **過關提示**：透過簡單的 Python `if-else` 判斷式，動態改變傳給 `pd.read_sql()` 的 `query` 變數字串，你就能完美實作出業界最真實的「RBAC (角色權限控管)」系統！
