# 實作七：建立資料表與完整性限制條件 (DDL 與全端防呆加長版)

**授課教師：劉鎮豪**
**課程：資料庫管理系統**
**學校：國立金門大學**

在前面章節我們學會了如何查詢資料，但資料庫界有一句名言：「垃圾進，垃圾出 (Garbage In, Garbage Out)」。如果一開始存入資料庫的資料就是錯的，後續查出來的報表絕對是一場災難。

本章節我們將深入學習 **資料定義語言 (DDL)**。我們將探討 MySQL 的各式資料類型，並為「金門特產系統」建立具備 **「完整性限制條件 (Integrity Constraints)」** 的新資料表，讓資料庫自動幫我們阻擋不合法的錯誤資料。課程後半段，我們將學習如何運用 AI 助理，以「區塊拼裝」的方式建構前端驗證網頁。

---

## 📖 第一部分：理論與 SQL 實戰 (深度解析)

### 1. 豐富的 MySQL 資料類型

在建立資料表時，為欄位選擇正確的「資料類型 (Data Type)」是節省硬碟空間與提升查詢效能的關鍵。

* **數值類型**：`INT` (整數), `DECIMAL(總位數, 小數位數)` (精確小數，非常適合用於金額，避免浮點數誤差)。
* **字串類型**：
  * `VARCHAR(長度)`：變動長度字串。若設為 VARCHAR(50) 但只存 3 個字，就只佔 3 個字的空間，適合存姓名、地址。
  * `CHAR(長度)`：固定長度字串。效能較佳，適合存長度固定的資料，如身分證字號或電話號碼。
* **日期時間**：`DATE` (僅日期 YYYY-MM-DD), `DATETIME` (日期與時間)。
* **JSON 類型**：用於儲存半結構化的資料。

### 2. 建立資料表與完整性限制條件 (CREATE TABLE)

完整性限制條件是資料庫的「終極守門員」。常見的限制條件包含：
* **`PRIMARY KEY`**：主鍵，唯一且不可為空。
* **`NOT NULL`**：必填欄位，不允許留白。
* **`UNIQUE`**：唯一值，整張表中不可重複（例如：Email、手機號碼）。
* **`DEFAULT`**：預設值，若新增時未提供資料，系統會自動帶入此值。
* **`CHECK`**：檢查條件，必須符合特定邏輯才能寫入（例如：價格不能是負數）。

**💻 SQL 實戰：建立「特產店員工表」**

請在 MySQL Workbench 中執行以下語法，體會限制條件的威力：

  ```sql
  USE kinmen_shop;

  -- 建立員工資料表 (結合多種限制條件)
  CREATE TABLE employees (
      emp_id VARCHAR(10) PRIMARY KEY,              -- 主鍵
      emp_name VARCHAR(50) NOT NULL,               -- 姓名必填
      email VARCHAR(100) UNIQUE,                   -- Email 不可與他人重複
      hire_date DATE DEFAULT (CURRENT_DATE),       -- 若未填寫，預設為今天日期
      salary DECIMAL(10, 2) CHECK (salary >= 27470) -- 薪水不可低於基本工資
  );
  ```
> **💼 業界實務補充：為什麼主鍵 (PK) 這麼重要？**
> 在業界，每一張資料表「絕對」要有主鍵。如果沒有主鍵，當你發現有兩筆名字同為「王小明」、其他內容也一模一樣的重複資料時，你將**永遠無法單獨刪除或修改其中一筆**，這會造成資料庫的災難。

### 3. 修改與刪除資料表 (ALTER & DROP)

實務上，系統上線後常會需要修改資料表結構。我們可以使用 `ALTER TABLE` 來進行微調。

  ```sql
  -- 1. 新增欄位 (ADD)：公司決定在員工表新增「職稱」欄位
  ALTER TABLE employees ADD COLUMN job_title VARCHAR(30);

  -- 2. 修改欄位屬性 (MODIFY)：發現 Email 長度不夠，加長到 150
  ALTER TABLE employees MODIFY COLUMN email VARCHAR(150);

  -- 3. 刪除欄位 (DROP)：因為隱私考量，決定移除員工的 Email 欄位
  ALTER TABLE employees DROP COLUMN email;

  -- 4. 刪除整張資料表 (加上 IF EXISTS 避免報錯)
  -- DROP TABLE IF EXISTS old_backup_table;
  ```
> **💣 易錯提醒：ALTER TABLE 的隱藏風險**
> 在上線的系統中執行 `ALTER TABLE` 是一件非常危險的事！如果資料表裡已經有數百萬筆資料，你突然加上一個 `NOT NULL` 的欄位卻沒有給預設值，整個資料庫可能會因為鎖定 (Table Lock) 而停機數分鐘。實務上，這種操作通常必須在半夜進行！

### 4. 暫存資料表的建立 (TEMPORARY TABLE)

如果我們只需要一個臨時的空間來處理資料（例如結算本月業績），可以建立「暫存資料表」。**當你關閉與資料庫的連線時，暫存資料表就會自動消失**，不會佔用永久空間。

  ```sql
  -- 建立暫存資料表
  CREATE TEMPORARY TABLE temp_promotions (
      promo_id INT PRIMARY KEY,
      promo_name VARCHAR(50)
  );
  ```

---

## 🤖 第二部分：AI 輔助開發 (溝通心法與區塊拼裝)

在開發全端系統時，我們不該一開始就要求 AI 寫出整套系統，正確的做法是：**「先準備共用骨架，再針對單一功能區塊請 AI 協助撰寫寫入邏輯。」**

> [!TIP]
> **【給 AI 的 Prompt - 協助撰寫寫入與防呆邏輯】**
>
> 「我正在使用 Python Streamlit 開發『新增員工』表單。當使用者按下送出按鈕後，我要把資料寫入 MySQL 資料庫。
> 1. 撰寫 `INSERT INTO` 語法，把 `emp_id`, `emp_name`, `salary` 寫入 `employees` 表。
> 2. **核心重點：** 請使用 `try...except` 區塊。請特別用 `pymysql.err.IntegrityError` 和 `pymysql.err.OperationalError` 來捕捉『違反主鍵/UNIQUE 重複』以及『違反 CHECK 限制條件』的錯誤，並用 `st.error` 顯示繁體中文警告。」

> **🧠 Prompt 解析 (為什麼這樣寫？)**
> 如果我們不告訴 AI 要捕捉特定的 `IntegrityError`，AI 通常只會寫一個大的 `except Exception as e:`。這樣當違反主鍵規則時，使用者只會看到一長串可怕的英文錯誤代碼 (Error 1062)，而不是我們想要的「員工編號已重複」。精準要求例外處理，是資深工程師的寫 code 日常。

---

## 💻 第三部分：Python (Streamlit) 區塊拼裝與錯誤捕捉

請在 Spyder 開啟 `app.py`，將底下的骨架與 AI 產生的邏輯拼裝起來。

  ```python
  import streamlit as st
  import pymysql

  # --- 共用連線設定 ---
  def get_connection():
      return pymysql.connect(
          host="127.0.0.1", user="root", password="1234", 
          database="kinmen_shop", charset="utf8mb4"
      )

  st.title("🛡️ 員工資料管理系統 (限制條件測試)")
  st.markdown("測試 MySQL 的防呆機制如何保護資料庫。")
  st.divider()

  # 使用 Streamlit 建立表單 UI
  with st.form("employee_form"):
      st.subheader("➕ 新增特產店員工")
      emp_id = st.text_input("員工編號 (例如: E001)", max_chars=10)
      emp_name = st.text_input("員工姓名 (必填)")
      
      # 故意將預設值設為低於基本工資，測試資料庫的 CHECK 條件
      salary = st.number_input("員工薪資 (不得低於 27470)", value=25000, step=1000) 
      
      submitted = st.form_submit_button("寫入資料庫")
      
      if submitted:
          if not emp_id or not emp_name:
              # 【前端防呆】避免根本沒填資料就白白浪費資料庫連線運算
              st.toast('⚠️ 員工編號與姓名不可空白！', icon='🚨')
              st.warning("請填寫完整資訊後再送出。")
          else:
              try:
                  conn = get_connection()
                  cursor = conn.cursor()
                  
                  # 【逐行解析】執行寫入。如果這裡違反了 CHECK 或 UNIQUE 條件，程式會立刻跳到 except 區塊
                  query = "INSERT INTO employees (emp_id, emp_name, salary) VALUES (%s, %s, %s)"
                  cursor.execute(query, (emp_id, emp_name, salary))
                  
                  # 💡 絕對不能忘記這行！把暫存區的異動正式寫入硬碟
                  conn.commit() 
                  
                  st.balloons()
                  st.success(f"✅ 成功新增員工：{emp_name}！")
                  
              except pymysql.err.IntegrityError as e:
                  # 【錯誤捕捉 1】專門處理主鍵重複 (Error 1062) 的錯誤
                  if e.args[0] == 1062:
                      st.error("❌ 新增失敗：這組【員工編號】已經存在，請勿重複新增！")
                  else:
                      st.error(f"❌ 違反資料庫完整性限制：{e}")
                      
              except pymysql.err.OperationalError as e:
                  # 【錯誤捕捉 2】處理 CHECK 條件 (例如薪水太低) 被阻擋的錯誤
                  # 在較新的 MySQL 版本中，違反 CHECK 會拋出 OperationalError (Error 3819)
                  st.error("❌ 新增失敗：【薪資】不符合法規限制，資料庫已嚴格拒絕寫入！")
                  
              except Exception as e:
                  st.error(f"❌ 發生系統錯誤：{e}")
                  
              finally:
                  if 'cursor' in locals(): cursor.close()
                  if 'conn' in locals(): conn.close()
  ```

---

## ❓ 第四部分：常見問題與排解 (FAQ)

1. **Q: 為什麼我執行 Python 的 `INSERT` 沒報錯，但在 Workbench 卻沒看到資料？**
   * A: 在 Python 中執行新增 (INSERT)、修改 (UPDATE)、刪除 (DELETE) 時，**絕對不能忘記寫 `conn.commit()`**！如果沒寫，資料只會停留在記憶體中，連線一關閉就消失了。
2. **Q: `UNIQUE` 限制跟 `PRIMARY KEY` (主鍵) 到底差在哪？**
   * A: 一張表只能有**一個**主鍵，且主鍵絕對不能是 NULL。但一張表可以有**很多個** `UNIQUE` 欄位（例如信箱、手機號碼），且 `UNIQUE` 欄位通常允許寫入 NULL 值（表示使用者尚未提供）。
3. **Q: 出現錯誤 `Table 'kinmen_shop.employees' doesn't exist`？**
   * A: 代表你忘記在 Workbench 執行本章節第一部分的 `CREATE TABLE` 語法，或是你在 Python 連接到了錯誤的資料庫，請檢查程式碼上方的連線設定。

---

## 📝 第五部分：實戰演練與魔王挑戰

### 🟢 基礎題 (DDL 操作)
1. **建立供應商資料表：** 請在 Workbench 建立一張 `suppliers` 表，包含：
   - `supplier_id` (字串 10 碼，設為主鍵)
   - `supplier_name` (字串 50 碼，必填 `NOT NULL`)
2. **新增欄位：** 建立後，請使用 `ALTER TABLE` 在 `suppliers` 表中新增一個欄位 `tax_id` (統一編號，字串 8 碼，必須是 `UNIQUE`)。

### 🟡 進階題 (破壞測試)
3. **挑戰底線：** 請在 Workbench 中寫入一筆 `salary` 為 `20000` (低於基本工資) 的資料到 `employees` 表中。請將 Workbench 下方 Output 視窗彈出的**紅色 Error Message 與 Error Code** 複製下來，並向隔壁同學解釋資料庫是透過哪個設定擋下這筆資料的。

### 😈 魔王挑戰題 (全端刪除功能)
4. **【員工離職系統】** * **挑戰要求**：請在 Streamlit 的 `app.py` 中設計一個新區塊。畫面上只要有一個文字輸入框 (`emp_id`) 與一個紅色的「執行刪除」按鈕。
   * **核心功能**：按下按鈕後，請連線到資料庫將該名員工刪除 (`DELETE FROM employees WHERE emp_id = %s`)。
   * **防呆要求**：如果使用者輸入的 `emp_id` 根本不存在於資料庫，系統不能無動於衷，必須用白話文 (`st.warning`) 告訴使用者：「找不到該名員工，請確認編號是否正確！」。
   * **過關提示**：你可以善用 Python 裡的 `cursor.rowcount` 屬性。它會告訴你剛才的 SQL 到底成功影響了幾筆資料！
