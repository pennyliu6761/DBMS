# 實作十五：資料指標、參數化查詢與交易處理 (資料一致性與安全實戰)

**授課教師：劉鎮豪**
**課程：資料庫管理系統**
**學校：國立金門大學**

想像一下：一個顧客在你的「金門特產店」網頁下單，系統已經扣了他的會員點數，結果在扣減庫存時，伺服器突然斷電了！這會發生什麼慘劇？點數沒了，貨卻沒出，這就是所謂的「資料不一致」。

本章節我們將學習資料庫的最高安全機制：**交易處理 (Transaction Processing)** 與 **鎖定 (Locking)**。我們將探討如何利用 ACID 特性保護資料，並學習如何使用 **資料指標 (Cursors)** 與 **參數化查詢**。後半段我們將在 Streamlit 實作一個「點數轉贈系統」，體會資料絕對安全的藝術！

---

## 📖 第一部分：理論與 SQL 實戰 (深度解析)

### 1. 資料指標 (Cursors) 與 參數化查詢
SQL 指令通常是處理「一整群」資料，但如果你想「一筆一筆」分開處理，就需要 **資料指標 (Cursor)**。它就像一個標籤，指著目前處理到哪一列。



  ```sql
  -- 【實戰：參數化查詢預備語法】
  -- 參數化查詢能提高效能並防止 SQL Injection，其語法如下：
  PREPARE stmt FROM 'SELECT * FROM products WHERE product_id = ?';
  SET @id = 'P001';
  EXECUTE stmt USING @id;
  DEALLOCATE PREPARE stmt;
  ```

### 2. 交易處理的 ACID 特性
交易 (Transaction) 是指一組不可分割的 SQL 指令。它必須具備以下四個特性：
* **原子性 (Atomicity)**：一組指令要嘛全部成功，要嘛全部失敗，沒有中間地帶。
* **一致性 (Consistency)**：交易前後，資料庫必須符合預設規則（如：點數不能為負值）。
* **隔離性 (Isolation)**：多個交易同時進行時，彼此不應干擾。
* **持久性 (Durability)**：一旦交易提交 (COMMIT)，異動就會永久儲存在硬碟中。



### 3. 交易指令：COMMIT 與 ROLLBACK

  ```sql
  USE kinmen_shop;

  START TRANSACTION; -- 開始交易

  -- 動作一：扣除會員 A 的點數
  UPDATE vip_members SET point_balance = point_balance - 100 WHERE member_id = 1;
  -- 動作二：增加會員 B 的點數
  UPDATE vip_members SET point_balance = point_balance + 100 WHERE member_id = 2;

  -- 情況 A：如果一切正常，正式提交
  -- COMMIT; 

  -- 情況 B：如果發現動作二失敗或點數不足，撤銷所有動作
  -- ROLLBACK; 
  ```

### 4. 資料鎖定與並行控制 (Locking)
為了避免兩個人同時修改同一筆資料造成衝突，MySQL 會使用「鎖定」機制。
* **共用鎖定 (Shared Lock, S)**：允許別人讀取，但不准修改。
* **獨佔鎖定 (Exclusive Lock, X)**：不准任何人讀取或修改。
* **死結 (Deadlock)**：兩個交易互相等待對方釋放鎖定，造成系統當機。



---

## 🤖 第二部分：AI 輔助開發 (設計穩健的交易邏輯)

當我們要處理跨資料表的連動動作時，寫法必須非常嚴謹。我們可以請 AI 幫忙規劃 Python 的交易捕捉結構。

> [!TIP]
> **【給 AI 的 Prompt - 協助設計點數轉帳交易邏輯】**
>
> 「你是一位資料庫資安專家。我正在使用 Python Streamlit 開發『會員點數轉贈功能』。
> 1. 請幫我寫一個 Python 函式，傳入 `sender_id`, `receiver_id`, `amount`。
> 2. **核心要求：** 必須使用 `conn.begin()` 開啟交易。
> 3. 執行兩個 UPDATE：先扣除 A 的點數，再增加 B 的點數。
> 4. **防呆與回滾：** 請使用 `try...except`。如果過程中發生任何錯誤（例如點數扣完變負值、或是連線中斷），請務必執行 `conn.rollback()` 撤銷所有異動，並顯示錯誤訊息；如果成功，則執行 `conn.commit()`。」

---

## 💻 第三部分：Python (Streamlit) 全端點數轉贈系統

請在 Spyder 開啟 `app.py`。我們將親手模擬一次「失敗的轉帳」，看看資料庫如何透過 `ROLLBACK` 保護我們的錢。

  ```python
  import streamlit as st
  import pymysql

  def get_connection():
      # 💡 為了測試交易，記得連線設定不要開啟 autocommit
      return pymysql.connect(
          host="127.0.0.1", user="root", password="1234", 
          database="kinmen_shop", charset="utf8mb4", autocommit=False
      )

  st.title("🛡️ 金門特產店 - 會員點數轉贈系統")
  st.markdown("展示資料庫『交易處理 (Transaction)』如何確保資料絕對安全。")
  st.divider()

  # 顯示目前會員點數狀況
  st.subheader("📋 目前會員點數概況")
  try:
      conn = get_connection()
      cursor = conn.cursor(pymysql.cursors.DictCursor)
      cursor.execute("SELECT member_id, phone_number, point_balance FROM vip_members")
      st.table(cursor.fetchall())
  except Exception as e:
      st.error(f"讀取失敗：{e}")
  finally:
      conn.close()

  # --- 轉贈功能區 ---
  st.header("💸 點數轉贈操作")
  with st.form("transfer_form"):
      from_id = st.number_input("轉出會員 ID", min_value=1, step=1)
      to_id = st.number_input("接收會員 ID", min_value=1, step=1)
      amount = st.number_input("轉贈點數金額", min_value=1, step=10)
      
      if st.form_submit_button("立即執行轉贈"):
          conn = get_connection()
          try:
              # 💡 1. 開啟交易
              conn.begin()
              cursor = conn.cursor()
              
              # 💡 2. 執行動作一：扣除點數
              # 特別檢查：SQL 會配合 CHECK 限制條件防止點數變負數
              update_sql1 = "UPDATE vip_members SET point_balance = point_balance - %s WHERE member_id = %s"
              cursor.execute(update_sql1, (amount, from_id))
              
              # 💡 3. 執行動作二：增加點數
              update_sql2 = "UPDATE vip_members SET point_balance = point_balance + %s WHERE member_id = %s"
              cursor.execute(update_sql2, (amount, to_id))
              
              # 💡 4. 若都成功，則提交異動
              conn.commit()
              st.success(f"🎉 轉贈成功！已將 {amount} 點從 ID:{from_id} 轉給 ID:{to_id}")
              st.balloons()
              
          except Exception as e:
              # 💡 5. 萬一出錯 (例如 ID 不存在或點數不足)，立刻撤銷
              conn.rollback()
              st.error(f"❌ 轉贈失敗！資料已全數回滾 (Rollback)。錯誤原因：{e}")
          finally:
              conn.close()
  ```

---

## ❓ 第四部分：常見問題與排解 (FAQ)

1. **Q: 為什麼我沒寫 `COMMIT`，重整網頁資料就復原了？**
   * A: 這就是交易的威力！在沒寫 `COMMIT` 前，資料庫只會把異動暫存在記憶體。如果連線中斷或網頁重整，資料庫會自動視為交易失敗並回滾。
2. **Q: 什麼是「死結 (Deadlock)」？遇到時該怎麼辦？**
   * A: 當 A 鎖住了產品表等會員表，而 B 鎖住了會員表等產品表，兩個人就會互相卡死。
     * **解法**：1. 盡量縮短交易時間。 2. 讓所有的交易都以「相同的順序」存取表格（例如：永遠先動會員表，再動產品表）。
3. **Q: 參數化查詢為什麼能防範 SQL Injection？**
   * A: 因為參數化查詢會先讓資料庫「編譯」好指令框架，參數（如使用者輸入的 ID）只會被當成純字串處理，不會被誤認為是 SQL 指令的一部分。

---

## 📝 第五部分：實戰演練與魔王挑戰

### 🟢 基礎題 (交易指令熟悉)
1. **練習 Rollback：** 請在 Workbench 執行 `START TRANSACTION;`，隨便刪除一筆資料，然後執行 `SELECT` 確認資料沒了。最後執行 `ROLLBACK;`，再 `SELECT` 一次，看看資料是否神奇地回來了？
2. **參數化查詢練習：** 請參考課本語法，利用 `PREPARE` 指令撰寫一段「查詢特定類別商品」的語法。

### 🟡 進階題 (鎖定模式與並行)
3. **體驗「鎖定」：** 請同學兩人一組。同學 A 執行 `START TRANSACTION;` 並更新一筆商品價格（但不 commit）；同學 B 同時嘗試更新同一筆商品。觀察同學 B 的畫面是否會卡住 (Waiting for lock)？
4. **意圖鎖定 (Intent Lock) 思考：** 請查閱課本第 15-5-2 節，解釋什麼是「意圖鎖定」，它在提升鎖定效能上扮演什麼角色？

### 😈 魔王挑戰題 (全端退貨與點數回補系統)
5. **【完美退貨自動化程序】** * **挑戰要求**：店長希望能有一個「一鍵退貨」功能。
   * **複雜交易邏輯**：
     - 從 `order_details` 刪除該筆訂單。
     - 將 `products` 內的庫存加回去。
     - 將 `vip_members` 內該次消費贈送的點數扣回來。
   * **UI 要求**：在 Streamlit 實作一個「輸入訂單 ID 退貨」的介面。
   * **過關標準**：如果退貨時導致會員點數變為負數（因為點數已被換掉），資料庫必須觸發錯誤並執行 `ROLLBACK`，絕對不能讓會員點數變負值！
