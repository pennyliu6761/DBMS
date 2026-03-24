# 第十章：新增、更新與刪除資料 (DML 操作與全端資料維護)

**授課教師：劉鎮豪**
**課程：資料庫管理系統**
**學校：國立金門大學**

在前面章節中，我們花了很大的篇幅學習如何「查詢 (SELECT)」資料。但在真實的商業系統中，金門特產店的商品會不斷進貨、調漲價格，甚至停售。
本章節我們將進入 **資料操作語言 (DML, Data Manipulation Language)** 的核心：**新增 (INSERT)**、**更新 (UPDATE)** 與 **刪除 (DELETE)**。課程後半段，我們將利用 Python (Streamlit) 打造一個完整的「後台商品維護系統」，讓店長能輕鬆管理庫存！

---

## 🛠️ 課前準備：建立測試資料庫與備份觀念

由於本章節的指令具備「破壞性」，為了避免同學不小心把前面的辛苦建立的資料毀掉，我們將學習課本中的一個實用密技：**使用 SELECT 查詢結果來建立新資料表 (資料表複製)**。

請在 MySQL Workbench 開啟查詢視窗，執行以下語法，我們將複製一份 `products` 表來進行本章的破壞性測試：

  ```sql
  USE kinmen_shop;

  -- 密技：快速複製一張結構與資料都一模一樣的備份表
  CREATE TABLE products_backup AS
  SELECT * FROM products;

  -- 確認備份表是否成功建立
  SELECT * FROM products_backup;
  ```

---

## 📖 第一部分：理論與 SQL 實戰 (深度解析)

### 1. 在 MySQL Workbench 中直接編輯資料
除了寫程式，Workbench 也提供了直覺的圖形化介面 (GUI) 讓我們修改資料。
* **操作步驟**：在左側 `SCHEMAS` 面板找到 `products_backup` 資料表 ➔ 點擊資料表名稱右側的「表格圖示 (Table Inspector)」或直接執行 `SELECT *` ➔ 在下方的 Result Grid 中，你可以像用 Excel 一樣直接點擊儲存格修改內容 ➔ 修改完畢後，**務必點擊右下角的 `Apply` 按鈕**，Workbench 才會將你的操作轉換為 `UPDATE` 語法並寫入資料庫！

### 2. 新增記錄 (INSERT)
我們已經很熟悉基本的單筆新增，但實務上常常需要「把 A 表的資料，整批匯入 B 表」。

  ```sql
  -- 基本新增：指定欄位並給予對應的值
  INSERT INTO products_backup (product_id, product_name, category, price, stock) 
  VALUES ('P009', '金門高粱醋', '伴手禮', 350, 100);

  -- 進階批次新增 (INSERT INTO ... SELECT)
  -- 假設我們有一張「新進貨商品表(new_items)」，我們可以直接把它的資料整批倒進備份表中：
  -- INSERT INTO products_backup (product_id, product_name, category, price, stock)
  -- SELECT id, name, type, cost, qty FROM new_items WHERE type = '酒類';
  ```

### 3. 更新記錄 (UPDATE)
當商品價格調漲或庫存變動時，我們必須使用 `UPDATE` 指令。

  ```sql
  -- 單筆更新：將「金門高粱醋」的價格調降為 300 元
  UPDATE products_backup 
  SET price = 300 
  WHERE product_id = 'P009';

  -- 多欄位與批次更新：將所有「肉品」的價格調漲 10%，且庫存統一加 50
  UPDATE products_backup 
  SET price = price * 1.1, stock = stock + 50 
  WHERE category = '肉品';
  ```

> **💣 業界核彈級地雷：忘記加 WHERE 條件**
> 如果你執行 `UPDATE products_backup SET price = 0;` (沒有 WHERE)，**全店所有的商品都會變成 0 元！** 這是無數工程師的惡夢。為了防止這種慘劇，MySQL Workbench 預設開啟了 **Safe Updates (安全更新模式)**。在安全模式下，如果你的 `WHERE` 條件不是使用 Primary Key (主鍵)，系統會直接擋下你的 `UPDATE` 與 `DELETE` 指令！

### 4. 刪除記錄 (DELETE) 與清空資料表 (TRUNCATE)
當商品停售時，我們將其從資料庫中移除。

  ```sql
  -- 基本刪除：刪除代號為 P009 的商品
  DELETE FROM products_backup WHERE product_id = 'P009';

  -- 課本進階技巧：子查詢與合併刪除 (DELETE with JOIN)
  -- 刪除所有「沒有出現在訂單明細中」的滯銷商品
  DELETE FROM products_backup 
  WHERE product_id NOT IN (SELECT DISTINCT product_id FROM order_details);
  ```

> **💡 觀念釐清：`DELETE` vs `TRUNCATE TABLE`**
> * `DELETE FROM 表格;`：一筆一筆刪除資料，會記錄在交易日誌中（速度較慢，但有機會還原）。
> * `TRUNCATE TABLE 表格;`：直接把整張表的資料「瞬間清空」並重置流水號，速度極快，但不記錄單筆日誌（一旦執行，神仙也難救！）。
> ```sql
> -- 瞬間清空整張備份表
> TRUNCATE TABLE products_backup;
> ```

---

## 🤖 第二部分：AI 輔助開發 (後台管理介面規劃)

我們將請 AI 幫忙規劃一個「後台商品管理系統」，讓店長可以在網頁上直接修改價格與刪除商品。

> [!TIP]
> **【給 AI 的 Prompt - 協助設計 Streamlit 資料維護介面】**
>
> 「我正在使用 Python Streamlit 開發『金門特產店後台系統』。我需要實作『更新價格』與『刪除商品』的功能。
> 1. 請幫我利用 `st.form` 或 `st.expander` 規劃兩個獨立的區塊介面。
> 2. **區塊一 (更新價格)：** 包含一個商品代號輸入框 (`product_id`)、一個新的價格數字輸入框 (`new_price`)，以及一個『確認修改』按鈕。
> 3. **區塊二 (刪除商品)：** 包含一個商品代號輸入框 (`del_product_id`)，以及一個紅色的『確認刪除』按鈕 (提示：可以使用 `type="primary"` 讓按鈕變色)。
> 4. 在按鈕按下的判斷邏輯處，請留白或加上註解即可，我會自己補上 pymysql 的 SQL 語法。只給我 UI 排版的程式碼。」

> **🧠 Prompt 解析 (思維訓練)：**
> 破壞性的操作（修改、刪除）在 UI 設計上必須「各自獨立」，絕對不能混在一起，以免使用者點錯按鈕造成資料遺失。請 AI 幫我們切分區塊，能讓後續填寫 SQL 邏輯時思路更清晰。

---

## 💻 第三部分：Python (Streamlit) 全端資料維護實戰

請在 Spyder 開啟 `app.py`。我們將把 `UPDATE` 與 `DELETE` 語法注入到網頁按鈕中，並見證 `conn.commit()` 的重要性！

### 步驟 1：建立連線與資料總覽

  ```python
  import streamlit as st
  import pymysql
  import pandas as pd

  def get_connection():
      return pymysql.connect(
          host="127.0.0.1", user="root", password="1234", 
          database="kinmen_shop", charset="utf8mb4"
      )

  st.title("⚙️ 金門特產店 - 後台商品維護系統")
  st.markdown("在此進行商品價格的 **UPDATE (修改)** 與 **DELETE (刪除)** 操作。")
  st.divider()

  # --- 顯示目前最新的商品清單 ---
  st.subheader("📦 目前商品庫存清單")
  try:
      conn = get_connection()
      df = pd.read_sql("SELECT * FROM products", conn)
      st.dataframe(df, use_container_width=True)
  except Exception as e:
      st.error(f"讀取失敗：{e}")
  finally:
      if 'conn' in locals(): conn.close()
  ```

### 步驟 2：實作 UPDATE 更新價格功能

  ```python
  st.divider()
  st.header("1. 📝 更新商品價格")

  with st.form("update_form"):
      col1, col2 = st.columns(2)
      with col1:
          update_id = st.text_input("請輸入要修改的商品代號 (例如: P001)")
      with col2:
          new_price = st.number_input("請輸入新價格", min_value=0, step=10)
          
      update_btn = st.form_submit_button("確認修改價格")
      
      if update_btn:
          if update_id:
              try:
                  conn = get_connection()
                  cursor = conn.cursor()
                  
                  # 執行 UPDATE 語法
                  query = "UPDATE products SET price = %s WHERE product_id = %s"
                  cursor.execute(query, (new_price, update_id))
                  
                  # 💡 核心關鍵：執行 UPDATE/INSERT/DELETE 後，絕對不能忘記 commit！
                  conn.commit()
                  
                  # 利用 rowcount 判斷是否有真正修改到資料
                  if cursor.rowcount > 0:
                      st.success(f"✅ 成功將 {update_id} 的價格更新為 {new_price} 元！請重新整理網頁查看。")
                  else:
                      st.warning("⚠️ 找不到該商品代號，或者新價格與原價格相同，資料未變動。")
                      
              except Exception as e:
                  st.error(f"❌ 修改失敗：{e}")
              finally:
                  if 'cursor' in locals(): cursor.close()
                  if 'conn' in locals(): conn.close()
          else:
              st.warning("請輸入商品代號！")
  ```

### 步驟 3：實作 DELETE 刪除商品功能

  ```python
  st.divider()
  st.header("2. 🗑️ 刪除下架商品")

  with st.form("delete_form"):
      del_id = st.text_input("請輸入要刪除的商品代號 (⚠️ 操作無法復原)")
      
      # 刪除按鈕通常會使用醒目的顏色
      delete_btn = st.form_submit_button("確認刪除此商品", type="primary")
      
      if delete_btn:
          if del_id:
              try:
                  conn = get_connection()
                  cursor = conn.cursor()
                  
                  query = "DELETE FROM products WHERE product_id = %s"
                  cursor.execute(query, (del_id,))
                  conn.commit()
                  
                  if cursor.rowcount > 0:
                      st.success(f"✅ 已成功將 {del_id} 永久刪除！請重新整理網頁查看。")
                  else:
                      st.warning("⚠️ 找不到該商品代號，無資料被刪除。")
                      
              except pymysql.err.IntegrityError as e:
                  # 防呆捕捉：如果該商品已經有顧客購買(存在於訂單明細表 FK)，資料庫會拒絕刪除！
                  st.error("❌ 刪除失敗：該商品已有交易紀錄！為保持帳務完整，請勿直接刪除，建議將庫存歸零。")
              except Exception as e:
                  st.error(f"❌ 系統錯誤：{e}")
              finally:
                  if 'cursor' in locals(): cursor.close()
                  if 'conn' in locals(): conn.close()
          else:
              st.warning("請輸入商品代號！")
  ```

---

## ❓ 第四部分：常見問題與排解 (FAQ)

1. **Q: 在 Workbench 執行 `UPDATE` 時出現 `Error Code: 1175. You are using safe update mode...` 怎麼辦？**
   * A: 這是因為你的 `WHERE` 條件沒有使用 Primary Key（例如你寫 `WHERE category = '酒類'`）。
     * **解法一（單次解除）**：在執行語法前，先執行 `SET SQL_SAFE_UPDATES = 0;`。
     * **解法二（永久解除）**：到 Workbench 上方選單 `Edit > Preferences > SQL Editor`，將底下的 `Safe Updates` 取消勾選，然後重啟 Workbench 連線。
2. **Q: 為什麼 Python 程式跑完了沒報錯，但資料庫的價格完全沒變？**
   * A: 請大聲唸三次：「我有沒有寫 `conn.commit()`？」在 pymysql 中，DML 操作預設開啟了交易控制 (Transaction)，如果沒有 `commit()` 提交，連線一關閉，所有異動都會自動還原 (Rollback)。

---

## 📝 第五部分：實戰演練與魔王挑戰

### 🟢 基礎題 (DML 語法熟悉)
1. **建立備份表：** 請利用 `CREATE TABLE ... SELECT` 語法，將 `customers` (會員表) 完整複製一份，命名為 `customers_backup`。
2. **條件更新：** 請在 Workbench 寫一段 `UPDATE`，將 `products` 表中「所有庫存小於 20」的商品，價格全面調漲 50 元。

### 🟡 進階題 (合併刪除與清空)
3. **清理幽靈帳號：** 在 `customers_backup` 表中，請利用 `DELETE` 搭配 `NOT IN` 子查詢，將「從來沒有出現在 `order_details` (訂單明細) 中」的會員刪除。
4. **毀滅指令測試：** 請執行 `TRUNCATE TABLE customers_backup;`，然後再 `SELECT` 看看這張表發生了什麼事？

### 😈 魔王挑戰題 (Streamlit 批次調薪系統)
5. **【員工全面加薪功能】** * **挑戰要求**：老闆決定給員工加薪！請在 Streamlit 網頁中建立一個新區塊。
   * **UI 需求**：放置一個下拉選單讓使用者選擇「加薪幅度」(例如：3%, 5%, 10%)，以及一個「一鍵全體加薪」的按鈕。
   * **功能需求**：按下按鈕後，請使用 `UPDATE` 語法，將 `employees` (員工表) 中「所有員工」的 `salary` 乘上對應的加薪幅度（需處理小數點四捨五入）。
   * **過關提示**：因為要更新「全表」，你必須在 SQL 語法中特別注意條件的下法，或者直接在 Python 的 Try 區塊內進行完美的 `commit()`，並利用 `st.balloons()` 慶祝加薪成功！
