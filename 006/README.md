# 第六章：SQL 基礎查詢與 Python (Streamlit) 全端實作 (加長實戰版)

**授課教師：劉鎮豪**
**課程：資料庫管理系統**
**學校：國立金門大學**

在了解了資料庫的建置與正規化後，我們進入資料庫操作的核心：**查詢 (Query)**。
本章節我們將以「金門特產進銷存系統」為例，學習如何使用 `SELECT` 語法從資料庫中精準地撈出需要的資訊。課程後半段，我們將結合 AI 輔助，並使用 Python (Streamlit) 打造出具備互動介面的前端網頁。

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

### 1. 基礎查詢與別名 (SELECT, AS)

  ```sql
  -- 查詢特定欄位，並計算庫存總價值
  SELECT 
      product_name AS '商品名稱', 
      price AS '單價', 
      stock AS '庫存量',
      (price * stock) AS '庫存總值' 
  FROM products;
  ```
> **💼 業界實務補充：為什麼資深工程師討厭 `SELECT *`？**
> 在課堂練習時 `SELECT *` 很方便，但在業界，資料表動輒幾十個欄位、數百萬筆資料。如果前端只需要「商品名稱」，卻用 `SELECT *` 把圖片路徑、詳細介紹等大欄位全撈出來，會嚴重拖垮網路頻寬與資料庫效能。**「要什麼，拿什麼」**才是好習慣！

### 2. 條件過濾與空值陷阱 (WHERE, IS NULL)

  ```sql
  -- 找出屬於「伴手禮」且庫存小於 400 的商品
  SELECT * FROM products WHERE category = '伴手禮' AND stock < 400;

  -- 找出價格尚未定價 (NULL) 的商品
  SELECT * FROM products WHERE price IS NULL;
  ```
> **💣 易錯提醒：空值 (NULL) 的致命陷阱**
> 很多初學者會寫成 `WHERE price = NULL`，這在 SQL 中是**絕對錯誤**的！`NULL` 代表「未知」，你不能說一個未知等於另一個未知。必須使用 `IS NULL` 或 `IS NOT NULL` 來判斷。

### 3. 進階過濾與排序 (LIKE, ORDER BY)

  ```sql
  -- 找出商品名稱包含「牛肉乾」的所有商品 (模糊搜尋)
  SELECT * FROM products WHERE product_name LIKE '%牛肉乾%';

  -- 列出所有商品，並依照價格「由高到低」排序
  SELECT product_name, price FROM products ORDER BY price DESC;
  ```

---

## 🤖 第二部分：AI 輔助開發 (溝通心法與 Prompt 解析)

在全端開發中，我們常需要寫 JavaScript (JS) 來實現「即時搜尋」。我們來看看如何精準地請 AI 幫忙。

> [!TIP]
> **【給 AI 的 Prompt - 前端 JS 搜尋框】**
>
> 「你現在是一位資深的網頁前端工程師。我正在設計一個『金門特產進銷存系統』。
> 1. 請幫我寫一個單一檔案的 HTML (包含 CSS 與 JavaScript)。
> 2. 畫面上要有表格，預設顯示幾筆假資料（例如：高粱酒、貢糖、牛肉乾，包含名稱、類別、價格）。
> 3. **核心功能：** 請用 JavaScript 實作一個『即時搜尋框』。當我在搜尋框輸入關鍵字（例如：貢糖）時，下方的表格要能『即時隱藏』不相關的商品，只留下符合的列。
> 4. 請加上精美的 CSS 樣式，讓它看起來像現代化的後台管理系統。」

> **🧠 Prompt 解析 (為什麼這樣寫？)**
> * **角色設定**：「你現在是一位資深前端工程師」，這能讓 AI 輸出的程式碼更符合業界規範。
> * **情境賦予**：「金門特產進銷存系統」，讓 AI 產生的假資料和變數命名更有代入感。
> * **明確限制**：「單一檔案的 HTML」，避免 AI 囉嗦地把 HTML, CSS, JS 拆成三個檔案，增加同學複製貼上的困難。

---

## 💻 第三部分：Python (Streamlit) 全端系統拼裝

我們要把 SQL 查詢搬到 Python 網頁中。請在 Spyder 開啟 `app.py` 進行拼裝。

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
  st.divider()
  ```

### 步驟 2：關鍵字搜尋與排序功能 (包含防呆與特效)

  ```python
  st.header("🔍 商品關鍵字搜尋")

  search_keyword = st.text_input("請輸入商品名稱關鍵字：", placeholder="例如: 貢糖")

  if st.button("搜尋商品"):
      if search_keyword:
          try:
              conn = get_connection()
              cursor = conn.cursor()
              
              # 【逐行解析 1】撰寫帶有 ORDER BY 的 SQL，並保留 %s 作為安全佔位符
              query = "SELECT * FROM products WHERE product_name LIKE %s ORDER BY price ASC"
              
              # 【逐行解析 2】執行查詢，並在關鍵字前後補上 % 以符合 LIKE 語法
              cursor.execute(query, (f"%{search_keyword}%",))
              result = cursor.fetchall()
              
              if result:
                  columns = [col[0] for col in cursor.description]
                  df = pd.DataFrame(result, columns=columns)
                  
                  # 【前端優化】成功時除了顯示表格，還會放氣球特效並彈出提示！
                  st.balloons()
                  st.success(f"✅ 成功找到 {len(df)} 筆商品！")
                  st.dataframe(df, use_container_width=True)
              else:
                  st.warning("找不到符合關鍵字的商品，請嘗試其他關鍵字。")
                  
          except Exception as e:
              st.error(f"❌ 搜尋失敗：{e}")
          finally:
              if 'cursor' in locals(): cursor.close()
              if 'conn' in locals(): conn.close()
      else:
          # 【防呆機制】如果使用者沒打字就按搜尋，跳出提示阻止執行 SQL
          st.toast('⚠️ 請先輸入關鍵字喔！', icon='🚨')
  ```

---

## ❓ 第四部分：常見問題與排解 (FAQ)

1. **Q: 終端機顯示 `Port 8501 is already in use` 怎麼辦？**
   * A: 這代表你之前啟動的 Streamlit 伺服器還沒關閉。請在 Anaconda Prompt 裡按下 `Ctrl + C` 中止之前的程序，再重新執行 `streamlit run app.py`。
2. **Q: 網頁出現 `pymysql.err.ProgrammingError: (1064, "You have an error in your SQL syntax...")`？**
   * A: 這是最常見的錯誤！代表你的 SQL 字串寫錯了。請檢查 `query = "..."` 裡面是不是少打了空格（例如 `SELECT*FROM` 黏在一起），或者引號沒有成對。

---

## 📝 第五部分：實戰演練與魔王挑戰

### 🟢 基礎題 (熟悉語法)
1. 請寫一段 SQL，撈出所有「單價超過 500 元」的「酒類」商品。
2. 請寫一段 SQL，撈出庫存量**不是 0** 的所有商品，並依照庫存量**由少到多**排序。

### 🟡 進階題 (全端修改)
3. 請修改你的 Streamlit `app.py`：新增一個「下拉選單 (`st.selectbox`)」，讓使用者可以選擇「價格由高到低」或「價格由低到高」，並根據使用者的選擇，動態改變 SQL 裡面的 `DESC` 或 `ASC`。

### 😈 魔王挑戰題 (開放式解題)
4. **【庫存警告系統】** 老闆希望在網頁的側邊欄 (`st.sidebar`) 顯示一個「危險庫存警報」。
   * **要求**：請結合 SQL 查詢與 Streamlit 語法，自動找出所有「庫存小於 20 且價格不是 NULL」的商品，並在側邊欄用紅色的字體 (`st.error`) 列出這些商品的名稱與庫存量。
   * **提示**：你可以請 AI 幫你了解 `st.sidebar` 的用法！

---
<br>
<br>

# 第七章：建立資料表與完整性限制條件 (加長實戰版)

**授課教師：劉鎮豪**
**課程：資料庫管理系統**
**學校：國立金門大學**

本章節我們將深入學習 **資料定義語言 (DDL)**。我們將為「金門特產系統」建立具備 **「完整性限制條件 (Integrity Constraints)」** 的新資料表，讓資料庫自動幫我們阻擋不合法的錯誤資料。課程後半段，我們將學習如何運用 AI 助理，以「區塊拼裝」的方式建構前端驗證網頁。

---

## 📖 第一部分：理論與 SQL 實戰 (深度解析)

### 1. 建立資料表與完整性限制條件 (CREATE TABLE)

完整性限制條件是資料庫的「守門員」。常見的限制條件包含：
* **`PRIMARY KEY`**：主鍵，唯一且不可為空。
* **`NOT NULL`**：必填欄位。
* **`UNIQUE`**：唯一值，整張表中不可重複。
* **`CHECK`**：檢查條件，必須符合特定邏輯才能寫入。

  ```sql
  USE kinmen_shop;

  CREATE TABLE employees (
      emp_id VARCHAR(10) PRIMARY KEY,              
      emp_name VARCHAR(50) NOT NULL,               
      salary DECIMAL(10, 2) CHECK (salary >= 27470) -- 薪水不可低於基本工資
  );
  ```
> **💼 業界實務補充：為什麼主鍵 (PK) 這麼重要？**
> 在業界，每一張資料表「絕對」要有主鍵。如果沒有主鍵，當你發現有兩筆名字同為「王小明」的資料，且其他內容也一模一樣時，你將永遠無法單獨刪除或修改其中一筆，這會造成資料庫的災難。

### 2. 修改與刪除資料表 (ALTER & DROP)

  ```sql
  -- 新增欄位與唯一限制
  ALTER TABLE employees ADD COLUMN email VARCHAR(100) UNIQUE;

  -- 刪除資料表 (加上 IF EXISTS 避免報錯)
  DROP TABLE IF EXISTS old_backup_table;
  ```
> **💣 易錯提醒：ALTER TABLE 的風險**
> 在上線的系統中執行 `ALTER TABLE` 是一件危險的事。如果資料表裡已經有 100 萬筆資料，你突然加上一個 `NOT NULL` 但沒有給預設值的欄位，整個資料庫可能會因為鎖定 (Lock) 而停機數分鐘。實務上都會在半夜進行！

---

## 🤖 第二部分：AI 輔助開發 (區塊拼裝心法)

在開發全端系統時，我們不該一開始就要求 AI 寫出整套系統，正確的做法是：**「先準備共用骨架，再針對單一功能區塊請 AI 協助撰寫。」**

> [!TIP]
> **【給 AI 的 Prompt - 協助撰寫寫入與防呆邏輯】**
>
> 「我正在使用 Python Streamlit 開發表單。當使用者按下送出按鈕後，我要把資料寫入 MySQL 資料庫。
> 1. 撰寫 `INSERT INTO` 語法，把 `emp_id`, `emp_name`, `salary` 寫入 `employees` 表。
> 2. **核心重點：** 請使用 `try...except` 區塊。請特別用 `pymysql.err.IntegrityError` 和 `pymysql.err.OperationalError` 來捕捉『違反主鍵/UNIQUE 重複』以及『違反 CHECK 限制條件』的錯誤，並用 `st.error` 顯示警告。」

> **🧠 Prompt 解析 (為什麼這樣寫？)**
> 如果我們不告訴 AI 要捕捉特定的 `IntegrityError`，AI 通常只會寫一個大的 `except Exception as e:`。這樣當違反主鍵規則時，使用者只會看到一長串可怕的英文錯誤代碼 (Error 1062)，而不是我們想要的「員工編號已重複」。

---

## 💻 第三部分：Python (Streamlit) 區塊拼裝與錯誤捕捉

請在 Spyder 開啟 `app.py`，將底下的骨架與 AI 產生的邏輯拼裝起來。

  ```python
  import streamlit as st
  import pymysql

  def get_connection():
      return pymysql.connect(
          host="127.0.0.1", user="root", password="1234", 
          database="kinmen_shop", charset="utf8mb4"
      )

  st.title("🛡️ 員工資料管理系統 (限制條件測試)")
  st.divider()

  with st.form("employee_form"):
      emp_id = st.text_input("員工編號 (例如: E001)", max_chars=10)
      emp_name = st.text_input("員工姓名 (必填)")
      salary = st.number_input("員工薪資 (不得低於 27470)", value=25000, step=1000) 
      
      submitted = st.form_submit_button("寫入資料庫")
      
      if submitted:
          if not emp_id or not emp_name:
              # 【前端防呆】避免根本沒填資料就白白浪費資料庫運算
              st.warning("⚠️ 員工編號與姓名不可空白！")
          else:
              try:
                  conn = get_connection()
                  cursor = conn.cursor()
                  
                  # 【逐行解析】執行寫入。如果這裡違反了 CHECK 或 UNIQUE 條件，程式會立刻跳到 except 區塊
                  query = "INSERT INTO employees (emp_id, emp_name, salary) VALUES (%s, %s, %s)"
                  cursor.execute(query, (emp_id, emp_name, salary))
                  conn.commit() # 記得 commit 才會真正寫入硬碟！
                  
                  st.success(f"✅ 成功新增員工：{emp_name}！")
                  
              except pymysql.err.IntegrityError as e:
                  # 【錯誤捕捉 1】專門處理主鍵重複 (1062) 的錯誤
                  if e.args[0] == 1062:
                      st.error("❌ 新增失敗：這組【員工編號】已經存在！")
                  else:
                      st.error(f"❌ 違反資料庫完整性限制：{e}")
                      
              except pymysql.err.OperationalError as e:
                  # 【錯誤捕捉 2】處理 CHECK 條件 (例如薪水太低) 被阻擋的錯誤
                  st.error("❌ 新增失敗：【薪資】不符合法規限制，資料庫拒絕寫入！")
                  
              except Exception as e:
                  st.error(f"❌ 發生系統錯誤：{e}")
                  
              finally:
                  if 'cursor' in locals(): cursor.close()
                  if 'conn' in locals(): conn.close()
  ```

---

## ❓ 第四部分：常見問題與排解 (FAQ)

1. **Q: 為什麼我執行 SQL 的 `INSERT` 沒報錯，但在 Workbench 卻沒看到資料？**
   * A: 在 Python 中執行新增、修改、刪除時，**絕對不能忘記寫 `conn.commit()`**！如果沒寫，資料只會停留在暫存區，連線一關閉就消失了。
2. **Q: `UNIQUE` 限制跟 `PRIMARY KEY` (主鍵) 到底差在哪？**
   * A: 一張表只能有一個主鍵，且主鍵絕對不能是 NULL。但一張表可以有很多個 `UNIQUE` 欄位（例如信箱、手機號碼），且 `UNIQUE` 欄位通常允許寫入 NULL。

---

## 📝 第五部分：實戰演練與魔王挑戰

### 🟢 基礎題 (DDL 操作)
1. **建立供應商資料表：** 請建立一張 `suppliers` 表，包含：
   - `supplier_id` (字串 10 碼，設為主鍵)
   - `supplier_name` (字串 50 碼，必填 `NOT NULL`)
2. **新增欄位：** 請使用 `ALTER TABLE` 在 `suppliers` 表中新增一個欄位 `tax_id` (統一編號，字串 8 碼，必須是 `UNIQUE`)。

### 🟡 進階題 (破壞測試)
3. **挑戰底線：** 請在 Workbench 中寫入一筆 `salary` 為 `20000` (低於基本工資) 的資料到 `employees` 表中。請將 Workbench 下方 Output 視窗彈出的**紅色 Error Message 與 Error Code** 複製下來，並解釋資料庫是透過哪個設定擋下這筆資料的。

### 😈 魔王挑戰題 (全端刪除功能)
4. **【員工離職系統】** * **要求**：請在 Streamlit 中設計一個新區塊。畫面上只要有一個輸入框 (`emp_id`) 與一個紅色的「執行刪除」按鈕。
   * **功能**：按下按鈕後，請到資料庫將該名員工刪除 (`DELETE FROM employees WHERE ...`)。
   * **防呆要求**：如果輸入的 `emp_id` 根本不存在於資料庫，不能顯示系統報錯，必須用白話文 (`st.warning`) 告訴使用者：「找不到該員工，請確認編號是否正確！」。
   * **提示**：你可以善用 `cursor.rowcount` 來判斷 SQL 到底有沒有成功刪除資料。
