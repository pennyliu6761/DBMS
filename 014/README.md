# 第十四章：預存程序、函數與觸發程序 (資料庫邏輯自動化實戰)

**授課教師：劉鎮豪**
**課程：資料庫管理系統**
**學校：國立金門大學**

在之前的章節中，所有的邏輯都是在 Python (Streamlit) 端完成後才下指令給資料庫。但在處理海量資料或複雜交易時，頻繁的資料傳輸會造成效能瓶頸。
本章節我們將學習如何將邏輯直接封裝在資料庫中：**預存程序 (Stored Procedures)**、**自定義函數 (Functions)** 與 **觸發程序 (Triggers)**。透過這些工具，我們能讓資料庫具備「自動反應」的能力，打造一個會自動補貨、自動計算折扣的聰明系統！



---

## 📖 第一部分：理論與 SQL 實戰 (深度解析)

### 1. 預存程序 (Stored Procedures)
預存程序是一組預先編譯好的 SQL 指令集合。它最大的優點是「只需編譯一次，即可多次執行」，且能接受參數傳入。

  ```sql
  USE kinmen_shop;

  -- 變更結束符號 (因為程序內部有多個分號，必須暫時改變結束符號)
  DELIMITER //

  -- 【實戰：自動調薪程序】傳入員工編號與加薪金額
  CREATE PROCEDURE AdjustSalary(IN emp_id_in VARCHAR(10), IN bonus DECIMAL(10,2))
  BEGIN
      UPDATE employees SET salary = salary + bonus WHERE emp_id = emp_id_in;
  END //

  DELIMITER ;

  -- 呼叫執行程序
  CALL AdjustSalary('E001', 2000);
  ```

### 2. 自定義函數 (Functions)
函數與程序的差別在於：函數**必須**回傳一個值，且可以直接在 `SELECT` 語句中使用。

  ```sql
  DELIMITER //

  -- 【實戰：計算折扣後價格函數】
  CREATE FUNCTION GetDiscountPrice(original_price INT, discount_rate DECIMAL(3,2))
  RETURNS INT
  DETERMINISTIC
  BEGIN
      RETURN ROUND(original_price * discount_rate);
  END //

  DELIMITER ;

  -- 直接在查詢中使用函數
  SELECT product_name, GetDiscountPrice(price, 0.9) AS '九折價' FROM products;
  ```

### 3. 觸發程序 (Triggers)
觸發程序是「被動」執行的。當資料表發生 `INSERT`、`UPDATE` 或 `DELETE` 時，資料庫會自動觸發預設好的動作。



  ```sql
  -- 【實戰：自動庫存異動日誌】
  -- 當訂單明細有新增時，自動減少商品表的庫存量
  DELIMITER //

  CREATE TRIGGER update_stock_after_sale
  AFTER INSERT ON order_details
  FOR EACH ROW
  BEGIN
      -- NEW 關鍵字代表剛剛新增的那筆訂單資料
      UPDATE products 
      SET stock = stock - NEW.qty 
      WHERE product_id = NEW.product_id;
  END //

  DELIMITER ;
  ```

> **💼 業界實務補充：**
> 在金融系統或電商平台，觸發程序常用於「審計日誌 (Audit Log)」。例如：當有人修改了訂單價格，觸發程序會自動將「誰、在什麼時間、把價格從 A 改成 B」寫入一張秘密的 Log 表，這在資安查核時非常重要。

---

## 🤖 第二部分：AI 輔助開發 (撰寫複雜的程序邏輯)

預存程序的 `DELIMITER` 與 `BEGIN...END` 語法較為繁瑣，我們可以請 AI 幫忙生成正確的語法結構。

> [!TIP]
> **【給 AI 的 Prompt - 協助撰寫帶有邏輯判斷的程序】**
>
> 「你是一位資料庫架構師。請幫我寫一個 MySQL 預存程序，名稱為 `ProcessOrder`。
> 1. 參數包含：`in_cust_id` (會員ID)、`in_prod_id` (商品ID)、`in_qty` (數量)。
> 2. **程序邏輯：**
>    - 先檢查 `products` 表中的 `stock` 是否足夠。
>    - 如果足夠：新增一筆資料到 `order_details`，並回傳訊息『訂單成功』。
>    - 如果不足：不執行新增，並回傳訊息『庫存不足』。
> 3. 請正確使用 `DELIMITER`、`DECLARE` 宣告變數、以及 `IF...ELSE` 邏輯。」

---

## 💻 第三部分：Python (Streamlit) 呼叫預存程序實戰

在 Python 中，我們不再下長長的 SQL 語法，而是直接「呼叫」寫好的程序，這能讓 Python 程式碼變得非常乾淨。

請在 Spyder 開啟 `app.py`：

  ```python
  import streamlit as st
  import pymysql

  def get_connection():
      return pymysql.connect(
          host="127.0.0.1", user="root", password="1234", 
          database="kinmen_shop", charset="utf8mb4"
      )

  st.title("🤖 金門特產店 - 自動化管理後台")
  st.markdown("展示如何透過 Python 呼叫資料庫內部的預存程序 (Stored Procedures)。")
  st.divider()

  # --- 呼叫預存程序進行加薪 ---
  st.header("💰 快速調薪系統")

  with st.form("salary_form"):
      target_id = st.text_input("輸入員工編號")
      bonus_amount = st.number_input("加薪金額", min_value=0, step=500)
      
      if st.form_submit_button("執行調薪程序"):
          try:
              conn = get_connection()
              cursor = conn.cursor()
              
              # 💡 核心技巧：使用 callproc() 呼叫預存程序
              # 參數 1：程序名稱；參數 2：傳入的參數元組 (tuple)
              cursor.callproc('AdjustSalary', (target_id, bonus_amount))
              
              # 記得程序內部的 UPDATE 也需要 commit 才會生效
              conn.commit()
              st.success(f"✅ 程序執行成功！員工 {target_id} 已完成調薪。")
              
          except Exception as e:
              st.error(f"❌ 執行失敗：{e}")
          finally:
              if 'conn' in locals(): conn.close()

  # --- 觸發程序效果展示 ---
  st.divider()
  st.header("📦 庫存連動測試 (觸發程序)")
  st.info("💡 說明：當你在此新增一筆訂單，資料庫的 Trigger 會自動去扣除商品表的庫存，不需在 Python 額外寫 UPDATE。")

  # (此處可實作簡易的新增訂單表單，並在執行後查詢 products 表驗證庫存是否減少)
  ```

---

## ❓ 第四部分：常見問題與排解 (FAQ)

1. **Q: 為什麼我在 Workbench 執行預存程序語法一直報錯？**
   * A: 九成機率是因為你漏掉了 **`DELIMITER`**。MySQL 預設遇到分號 `;` 就會結束執行，但程序內部有很多分號。必須先用 `DELIMITER //` 把結束標誌改成斜線，寫完程序後再用 `DELIMITER ;` 改回來。
2. **Q: `IN`, `OUT`, `INOUT` 參數有什麼差？**
   * A: 
     - `IN` (最常用)：像投幣口，把資料傳進程序裡。
     - `OUT`：像出幣口，程序執行完把結果「帶出來」給 Python。
     - `INOUT`：同一個變數，傳進去修改後再傳出來。
3. **Q: 觸發程序 (Trigger) 越多越好嗎？**
   * A: 絕對不是！過多的觸發程序會讓資料庫寫入效能大幅下降，且會讓偵錯變得極度困難（因為你看不到誰在偷偷改資料）。建議只在維護資料一致性時使用。

---

## 📝 第五部分：實戰演練與魔王挑戰

### 🟢 基礎題 (預存程序建置)
1. **折扣計算程序：** 請建立一個預存程序 `ApplyCategoryDiscount`，傳入「商品類別」與「折扣率」，程序會自動更新該類別下所有商品的價格。
2. **參數傳遞練習：** 請建立一個帶有 `OUT` 參數的程序 `GetStockCount`，傳入商品編號，程序會將該商品的庫存量賦值給 `OUT` 參數。

### 🟡 進階題 (觸發程序實作)
3. **自動下架機制：** 請為 `products` 表建立一個 `BEFORE UPDATE` 觸發程序。如果更新後的 `stock` 小於 0，自動將 `stock` 強制設定為 0 (防止出現負數庫存)。
4. **刪除備份機制：** 請建立一張 `deleted_products_log` 表。並建立一個 `AFTER DELETE` 觸發程序，當有商品被刪除時，自動將被刪除的商品名稱與刪除時間記錄到 Log 表中。

### 😈 魔王挑戰題 (全端自動化結帳系統)
5. **【一鍵完成交易程序】** * **挑戰任務**：老闆希望開發一個「高效結帳程序 `CompleteTransaction`」。
   * **程序內部邏輯**：
     - 同時寫入 `order_details`。
     - 更新該會員的 `point_balance` (點數) 增加（消費額的 1%）。
     - 檢查如果庫存歸零，自動發出一則通知（可寫入另一張通知表）。
   * **全端整合**：在 Streamlit 製作一個簡易結帳畫面，只呼叫這個預存程序，並在完成後顯示「交易成功，點數已更新」。
   * **過關提示**：這個程序涉及多表更新，請務必使用 `START TRANSACTION` 與 `COMMIT` 來確保交易的原子性 (Atomicity)！
