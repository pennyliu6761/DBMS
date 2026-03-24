# 第七章：建立資料表與完整性限制條件 (DDL 與 AI 輔助開發)

**授課教師：劉鎮豪**
**課程：資料庫管理系統**
**學校：國立金門大學**

本章節我們將深入學習 **資料定義語言 (DDL)**。我們將為「金門特產系統」建立具備 **「完整性限制條件 (Integrity Constraints)」** 的新資料表，讓資料庫自動幫我們阻擋不合法的錯誤資料。課程後半段，我們將學習如何運用 AI 助理，以「區塊拼裝」的方式，逐步建構出 Python (Streamlit) 前端驗證網頁。

---

## 📖 第一部分：理論與 SQL 實戰 (請在 Workbench 測試)

### 1. 建立資料表與完整性限制條件 (CREATE TABLE)

完整性限制條件是資料庫的「守門員」。常見的限制條件包含：
* **`PRIMARY KEY`**：主鍵，唯一且不可為空。
* **`NOT NULL`**：必填欄位，不允許留白。
* **`UNIQUE`**：唯一值，整張表中不可重複（例如：身分證字號、Email）。
* **`DEFAULT`**：預設值，若新增時未提供資料，會自動帶入此值。
* **`CHECK`**：檢查條件，必須符合特定邏輯才能寫入（例如：薪水必須大於基本工資）。

**💻 SQL 實戰：建立「特產店員工表」**

請在 MySQL Workbench 中執行以下語法，體會限制條件的威力：

  ```sql
  USE kinmen_shop;

  CREATE TABLE employees (
      emp_id VARCHAR(10) PRIMARY KEY,              
      emp_name VARCHAR(50) NOT NULL,               
      salary DECIMAL(10, 2) CHECK (salary >= 27470) -- 薪水不可低於基本工資
  );
  ```

### 2. 修改與刪除資料表 (ALTER & DROP)

實務上，系統上線後常會需要修改資料表結構。

  ```sql
  -- 新增欄位與唯一限制
  ALTER TABLE employees ADD COLUMN email VARCHAR(100) UNIQUE;

  -- 刪除資料表 (加上 IF EXISTS 避免報錯)
  DROP TABLE IF EXISTS old_backup_table;
  ```

---

## 🤖 第二部分：AI 輔助開發 (Streamlit 區塊拼裝心法)

在開發全端系統時，我們不該一開始就要求 AI 寫出整套系統，這樣很容易得到一堆看不懂的程式碼。正確的做法是：**「先準備共用骨架，再針對單一功能區塊請 AI 協助撰寫，最後將積木拼裝起來。」**

### 🧱 步驟零：建立共用骨架 (基底程式碼)

請在 Spyder 開啟 `app.py`，先貼上這段與資料庫連線的基底程式碼。這就像是蓋房子的地基。

  ```python
  import streamlit as st
  import pymysql

  # --- 共用連線設定 ---
  def get_connection():
      return pymysql.connect(
          host="127.0.0.1",
          user="root",            # ⚠️ 請填入你的 MySQL 帳號
          password="1234",        # ⚠️ 請填入你的 MySQL 密碼
          database="kinmen_shop", 
          charset="utf8mb4"
      )

  st.title("🛡️ 員工資料管理系統 (限制條件測試)")
  st.markdown("測試 MySQL 的 `UNIQUE` 與 `CHECK` 限制條件。")
  st.divider()

  # ==========================================
  # 區塊一：新增員工表單 (等待 AI 協助撰寫)
  # ==========================================
  ```

---

### 🧩 步驟一：呼叫 AI 協助設計 UI 表單

現在，我們需要一個表單來接收使用者輸入。請將以下 Prompt 複製給 AI 助理 (ChatGPT/Gemini)：

> [!TIP]
> **【給 AI 的 Prompt - 任務一：建立前端表單】**
>
> 「我正在使用 Python 的 Streamlit 框架開發。請幫我寫一段 Streamlit 程式碼，建立一個名為『新增員工』的表單 (使用 `st.form`)。
> 表單內需要有三個輸入元件：
> 1. `emp_id`: 員工編號 (字串)
> 2. `emp_name`: 員工姓名 (字串)
> 3. `salary`: 員工薪資 (數字輸入框，預設值為 25000)
> 最後加上一個送出按鈕。
> **注意：請只輸出這個表單區塊的程式碼就好，不需要給我完整的連線或 import 設定。**」

請將 AI 產生的程式碼，**複製並貼到 `app.py` 骨架的「區塊一」下方**。存檔並執行 `streamlit run app.py`，看看畫面是不是出現表單了！

---

### 🧩 步驟二：呼叫 AI 協助撰寫 SQL 寫入與錯誤捕捉

表單有了，接著我們要處理按下送出按鈕後的「寫入動作」，並且要捕捉剛剛在資料庫設定的 `CHECK` 和 `UNIQUE` 錯誤。

> [!TIP]
> **【給 AI 的 Prompt - 任務二：後端寫入與錯誤捕捉】**
>
> 「接續剛剛的 Streamlit 表單。當使用者按下送出按鈕後，我要把資料寫入 MySQL 資料庫。
> 1. 請幫我寫一段 Python 程式碼，呼叫 `get_connection()` 建立連線。
> 2. 撰寫 `INSERT INTO` 語法，把 `emp_id`, `emp_name`, `salary` 寫入 `employees` 資料表，記得使用 `%s` 參數化查詢防範隱碼攻擊。
> 3. **核心重點：** 請使用 `try...except` 區塊。除了捕捉一般的 `Exception` 外，請特別用 `pymysql.err.IntegrityError` 和 `pymysql.err.OperationalError` 來捕捉『違反主鍵/UNIQUE 重複』以及『違反 CHECK 限制條件』的錯誤，並用 `st.error` 顯示白話文的警告給使用者。」

請仔細閱讀 AI 提供的 `try...except` 結構，將它整合進你剛剛的表單按鈕邏輯中。

---

## 🚑 第三部分：終極防線 (如果你真的卡住了...)

如果你在區塊拼裝的過程中，遇到縮排錯誤或變數對不起來的問題，且多次嘗試依然失敗，請不要灰心！這時我們可以使用「完整範例重構」的 Prompt，請 AI 幫你檢查並產出整份正確的程式碼。

> [!WARNING]
> **【給 AI 的 Prompt - 終極救援】**
>
> 「我現在有一段 Streamlit 程式碼，但它好像有錯誤，無法成功將資料寫入 MySQL。
> 這是我的原始程式碼：
> `(請在此處貼上你目前 app.py 裡的所有程式碼)`
>
> 請幫我修正它。目標是：
> 1. 提供一個能輸入員工編號、姓名與薪資的表單。
> 2. 將資料安全地 INSERT 到 MySQL。
> 3. 能夠優雅地捕捉並提示 MySQL 的 CHECK 與 UNIQUE 錯誤。
> 請給我一份可以完整執行並取代原檔案的最終 Python 程式碼。」

> **💡 教師解答參考區：** 如果你想對照標準答案，以下是這隻程式成功拼裝後的完整樣貌。
> 
> <details>
> <summary><b>點擊展開完整參考程式碼</b></summary>
>
>   ```python
>   import streamlit as st
>   import pymysql
>
>   def get_connection():
>       return pymysql.connect(
>           host="127.0.0.1",
>           user="root",
>           password="1234",
>           database="kinmen_shop", 
>           charset="utf8mb4"
>       )
>
>   st.title("🛡️ 員工資料管理系統 (限制條件測試)")
>   st.divider()
>
>   with st.form("employee_form"):
>       emp_id = st.text_input("員工編號 (例如: E001)", max_chars=10)
>       emp_name = st.text_input("員工姓名 (必填)")
>       salary = st.number_input("員工薪資 (不得低於 27470)", value=25000, step=1000) 
>       
>       submitted = st.form_submit_button("寫入資料庫")
>       
>       if submitted:
>           if not emp_id or not emp_name:
>               st.warning("⚠️ 員工編號與姓名不可空白！")
>           else:
>               try:
>                   conn = get_connection()
>                   cursor = conn.cursor()
>                   
>                   query = "INSERT INTO employees (emp_id, emp_name, salary) VALUES (%s, %s, %s)"
>                   cursor.execute(query, (emp_id, emp_name, salary))
>                   conn.commit()
>                   
>                   st.success(f"✅ 成功新增員工：{emp_name}！")
>                   
>               except pymysql.err.IntegrityError as e:
>                   error_code = e.args[0]
>                   if error_code == 1062:
>                       st.error("❌ 新增失敗：這組【員工編號】已經存在！")
>                   else:
>                       st.error(f"❌ 違反資料庫完整性限制：{e}")
>                       
>               except pymysql.err.OperationalError as e:
>                   st.error("❌ 新增失敗：【薪資】低於法規限制的 27470 元，資料庫拒絕寫入！")
>                   
>               except Exception as e:
>                   st.error(f"❌ 發生系統錯誤：{e}")
>                   
>               finally:
>                   if 'cursor' in locals(): cursor.close()
>                   if 'conn' in locals(): conn.close()
>   ```
> </details>

---

## 📝 課堂隨堂練習 (Exercises) 

請同學在 **MySQL Workbench** 中，挑戰以下 DDL 語法。

### 🟢 基礎題：建立資料表 (CREATE TABLE)
1. **供應商資料表：** 金門特產店需要記錄供應商。請建立一張 `suppliers` 表，包含：
   - `supplier_id` (字串 10 碼，設為主鍵)
   - `supplier_name` (字串 50 碼，必填 `NOT NULL`)
   - `contact_phone` (字串 15 碼)
2. **預設值與限制：** 在上述的 `suppliers` 表中，加入欄位 `status` (字串 10 碼)，請設定其**預設值 (DEFAULT)** 為 `'合作中'`。

### 🟡 中階題：修改表結構 (ALTER TABLE)
3. **新增欄位與唯一限制：** 老闆希望能紀錄供應商的統一編號。請修改 `suppliers` 表：
   - 新增一個欄位 `tax_id` (統一編號，字串 8 碼)。
   - 由於統編不可重複，請將該欄位加上 `UNIQUE` 限制條件。

### 🔴 進階題：挑戰資料庫底線 (Testing Constraints)
4. **破壞測試：** 請嘗試寫入三筆有問題的資料到剛建立的 `employees` 表，並將 Workbench 顯示的紅色錯誤訊息記錄下來：
   - 測試 A：寫入兩筆 `emp_id` 同為 `'E001'` 的員工。
   - 測試 B：寫入一筆 `emp_name` 留白 (沒有給值) 的員工。
   - 測試 C：寫入一筆 `salary` 為 `20000` 的員工。
