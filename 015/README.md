# 實作十五：資料指標、參數化查詢與交易處理

> **課程**：資料庫管理系統 ｜ **授課教師**：劉鎮豪 ｜ **學校**：國立金門大學

---

## 📌 情境導入

> 一個顧客在「金門特產店」網頁下單，系統已扣了他的會員點數，  
> 結果在扣減庫存時，伺服器突然斷電了！  
> **點數沒了，貨卻沒出——這就是「資料不一致」。**

本單元學習資料庫最高安全機制：**交易處理 (Transaction)** 與 **鎖定 (Locking)**，  
並透過 ACID 特性保護資料，同時實作 **Streamlit 點數轉贈系統**。

---

## 目錄

- [第一部分：資料表建置（前置作業）](#-第零部分資料表建置前置作業)
- [第二部分：理論與 SQL 實戰](#-第一部分理論與-sql-實戰)
  - [資料指標與參數化查詢](#1-資料指標-cursors-與參數化查詢)
  - [ACID 特性](#2-交易處理的-acid-特性)
  - [COMMIT 與 ROLLBACK](#3-交易指令commit-與-rollback)
  - [資料鎖定與並行控制](#4-資料鎖定與並行控制-locking)
- [第三部分：AI 輔助開發](#-第二部分ai-輔助開發)
- [第四部分：Streamlit 全端系統](#-第三部分python-streamlit-全端點數轉贈系統)
- [第五部分：常見問題 FAQ](#-第四部分常見問題與排解-faq)
- [第六部分：實戰演練](#-第五部分實戰演練與魔王挑戰)

---

## 🗄️ 第零部分：資料表建置（前置作業）

> 本單元需要 `vip_members` 資料表，請先執行以下 SQL 建立並填入測試資料。

### 建立 vip_members 資料表

```sql
USE kinmen_shop;

CREATE TABLE IF NOT EXISTS vip_members (
    member_id     INT          PRIMARY KEY AUTO_INCREMENT,
    customer_id   CHAR(4)      NOT NULL,
    phone_number  VARCHAR(20)  NOT NULL,
    level         VARCHAR(10)  DEFAULT 'Bronze',   -- Bronze / Silver / Gold
    point_balance INT          DEFAULT 0,

    -- ✅ 防止點數變負數
    CONSTRAINT chk_points CHECK (point_balance >= 0),
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);
```

### 插入測試資料

```sql
INSERT INTO vip_members (customer_id, phone_number, level, point_balance) VALUES
('C001', '0912-111-111', 'Gold',   1500),
('C002', '0923-222-222', 'Silver',  800),
('C003', '0934-333-333', 'Bronze',  200),
('C004', '0945-444-444', 'Bronze',   50);
```

<img width="373" height="95" alt="image" src="https://github.com/user-attachments/assets/735d27d4-4d42-42e8-b4a4-41fdbaad438a" />

### 補充 products 成本欄位

```sql
ALTER TABLE products ADD COLUMN IF NOT EXISTS cost INT;

SET SQL_SAFE_UPDATES = 0;
UPDATE products SET cost = price * 0.5 WHERE product_id IS NOT NULL;
SET SQL_SAFE_UPDATES = 1;
```

### 驗證資料

```sql
DESC vip_members;

SELECT v.member_id, c.customer_name, v.phone_number, v.level, v.point_balance
FROM vip_members v
JOIN customers c ON v.customer_id = c.customer_id;
```

<img width="397" height="98" alt="image" src="https://github.com/user-attachments/assets/b79eea6f-d54f-4cce-abb3-ae4c108a806a" />

### 本單元資料表清單

| 資料表 | 狀態 | 說明 |
|--------|:----:|------|
| `customers` | ✅ 已存在 | 顧客基本資料 |
| `products` | ✅ 需補 `cost` 欄位 | 商品資料 |
| `order_details` | ✅ 已存在 | 訂單明細（退貨題使用） |
| `vip_members` | ⚠️ **需新建** | 本單元交易處理主角 |

---

## 📖 第一部分：理論與 SQL 實戰

### 1. 資料指標 (Cursors) 與參數化查詢

SQL 指令通常處理「一整群」資料；若需「一筆一筆」分開處理，則需要 **資料指標 (Cursor)**——它像一個書籤，指向目前處理到哪一列。

**參數化查詢**可提高效能並防止 SQL Injection：

```sql
-- 準備查詢語句（? 為參數佔位符）
PREPARE stmt FROM 'SELECT * FROM products WHERE product_id = ?';

-- 設定參數並執行
SET @id = 'P001';
EXECUTE stmt USING @id;

-- 釋放預備語句
DEALLOCATE PREPARE stmt;
```

<img width="532" height="380" alt="image" src="https://github.com/user-attachments/assets/fc50584b-77eb-4f8a-ba97-4201747131be" />

> **為什麼參數化查詢能防範 SQL Injection？**  
> 資料庫會先「編譯」好指令框架，使用者輸入的參數只會被當作純字串，不會被誤認為 SQL 指令的一部分。

---

### 2. 交易處理的 ACID 特性

交易 (Transaction) 是一組**不可分割**的 SQL 指令，必須具備以下四個特性：

| 特性 | 英文 | 說明 |
|------|------|------|
| **原子性** | Atomicity | 全部成功，或全部失敗，沒有中間狀態 |
| **一致性** | Consistency | 交易前後資料必須符合預設規則（如：點數不能為負） |
| **隔離性** | Isolation | 多個交易同時進行時，彼此不互相干擾 |
| **持久性** | Durability | 一旦 COMMIT，異動永久儲存於硬碟 |

---

### 3. 交易指令：COMMIT 與 ROLLBACK

```sql
USE kinmen_shop;

START TRANSACTION;  -- ① 開始交易

-- 動作一：扣除會員 A 的點數
UPDATE vip_members SET point_balance = point_balance - 100 WHERE member_id = 1;

-- 動作二：增加會員 B 的點數
UPDATE vip_members SET point_balance = point_balance + 100 WHERE member_id = 2;

-- ② 情況 A：一切正常 → 正式提交
COMMIT;

-- ② 情況 B：發生錯誤 → 撤銷所有動作
-- ROLLBACK;
```

<img width="387" height="98" alt="image" src="https://github.com/user-attachments/assets/133c9d3c-447f-46d8-a289-80ebad6456c8" />

> **💡 小知識**  
> 沒有執行 `COMMIT` 前，所有異動只暫存於記憶體。  
> 連線中斷或網頁重整時，資料庫會自動視為交易失敗並回滾。

---

### 4. 資料鎖定與並行控制 (Locking)

為避免兩人同時修改同一筆資料造成衝突，MySQL 使用「鎖定」機制：

| 鎖定類型 | 符號 | 說明 |
|----------|:----:|------|
| **共用鎖定** (Shared Lock) | S | 允許他人讀取，但不准修改 |
| **獨佔鎖定** (Exclusive Lock) | X | 不准任何人讀取或修改 |

**⚠️ 死結 (Deadlock)**：兩個交易互相等待對方釋放鎖定，造成系統卡住。

**預防死結的原則：**
1. 盡量縮短交易的執行時間
2. 所有交易以**相同順序**存取資料表（例如：永遠先動 `vip_members`，再動 `products`）

---

## 🤖 第二部分：AI 輔助開發

設計跨資料表的連動交易時，可使用以下 Prompt 請 AI 協助規劃 Python 交易邏輯：

> **【Prompt 範本】**
>
> 「你是一位資料庫資安專家。我正在使用 Python Streamlit 開發『會員點數轉贈功能』。
> 1. 請幫我寫一個 Python 函式，傳入 `sender_id`, `receiver_id`, `amount`。
> 2. **核心要求**：必須使用 `conn.begin()` 開啟交易。
> 3. 執行兩個 UPDATE：先扣除 A 的點數，再增加 B 的點數。
> 4. **防呆與回滾**：請使用 `try...except`。若過程中發生任何錯誤（點數不足或連線中斷），務必執行 `conn.rollback()` 並顯示錯誤訊息；成功則執行 `conn.commit()`。」

---

## 💻 第三部分：Python (Streamlit) 全端點數轉贈系統

> 請在 Spyder 開啟 `app.py`，模擬一次「失敗的轉帳」，觀察 ROLLBACK 如何保護資料。

```python
import streamlit as st
import pymysql

def get_connection():
    """建立資料庫連線（關閉 autocommit 以支援交易）"""
    return pymysql.connect(
        host="127.0.0.1",
        user="root",
        password="1234",
        database="kinmen_shop",
        charset="utf8mb4",
        autocommit=False  # ✅ 必須關閉，才能手動控制 COMMIT / ROLLBACK
    )


# ── 頁面標題 ──────────────────────────────────────────────
st.title("🛡️ 金門特產店 - 會員點數轉贈系統")
st.markdown("展示資料庫『交易處理 (Transaction)』如何確保資料絕對安全。")
st.divider()


# ── 顯示目前會員點數 ───────────────────────────────────────
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


# ── 轉贈功能區 ─────────────────────────────────────────────
st.header("💸 點數轉贈操作")

with st.form("transfer_form"):
    from_id = st.number_input("轉出會員 ID", min_value=1, step=1)
    to_id   = st.number_input("接收會員 ID", min_value=1, step=1)
    amount  = st.number_input("轉贈點數金額", min_value=1, step=10)

    if st.form_submit_button("立即執行轉贈"):
        conn = get_connection()
        try:
            # ① 開啟交易
            conn.begin()
            cursor = conn.cursor()

            # ② 動作一：扣除點數（若餘額不足，CHECK 條件會觸發錯誤）
            sql_deduct = "UPDATE vip_members SET point_balance = point_balance - %s WHERE member_id = %s"
            cursor.execute(sql_deduct, (amount, from_id))

            # ③ 動作二：增加點數
            sql_add = "UPDATE vip_members SET point_balance = point_balance + %s WHERE member_id = %s"
            cursor.execute(sql_add, (amount, to_id))

            # ④ 全部成功 → 提交
            conn.commit()
            st.success(f"🎉 轉贈成功！已將 {amount} 點從 ID:{from_id} 轉給 ID:{to_id}")
            st.balloons()

        except Exception as e:
            # ⑤ 發生錯誤 → 全數回滾
            conn.rollback()
            st.error(f"❌ 轉贈失敗！資料已全數回滾 (Rollback)。\n\n錯誤原因：{e}")

        finally:
            conn.close()
```

<img width="734" height="834" alt="image" src="https://github.com/user-attachments/assets/6e158046-d60c-4e48-92d6-43e14b5acf79" />
<img width="724" height="443" alt="image" src="https://github.com/user-attachments/assets/dbad9a01-82dd-476a-be0d-7e1211f3f971" />
<img width="721" height="438" alt="image" src="https://github.com/user-attachments/assets/53d52fb5-be85-4b73-bb6f-06f3e741fdb4" />

---

## ❓ 第四部分：常見問題與排解 (FAQ)

**Q1：為什麼沒寫 `COMMIT`，重整網頁資料就復原了？**

> 交易的威力！`COMMIT` 前，所有異動只暫存於記憶體。連線中斷或網頁重整時，資料庫自動視為失敗並回滾。

**Q2：什麼是「死結 (Deadlock)」？遇到時該怎麼辦？**

> 交易 A 鎖住產品表等會員表，交易 B 鎖住會員表等產品表，兩者互相卡死。  
> **解法**：① 縮短交易時間；② 所有交易以相同順序存取資料表。

**Q3：參數化查詢為什麼能防範 SQL Injection？**

> 資料庫先「編譯」好指令框架，使用者輸入的參數只被當成純字串，不會被誤認為 SQL 指令。

**Q4：`Error Code: 1175` Safe Update Mode 怎麼處理？**

```sql
-- 方法一：暫時關閉安全模式（推薦）
SET SQL_SAFE_UPDATES = 0;
UPDATE products SET cost = price * 0.5;
SET SQL_SAFE_UPDATES = 1;

-- 方法二：WHERE 條件使用 KEY 欄位
UPDATE products SET cost = price * 0.5 WHERE product_id IS NOT NULL;
```

---

## 📝 第五部分：實戰演練與魔王挑戰

### 🟢 基礎題

**題目一：練習 ROLLBACK**

1. 執行 `START TRANSACTION;`
2. 隨機刪除一筆資料，用 `SELECT` 確認資料消失
3. 執行 `ROLLBACK;`，再 `SELECT` 一次——資料是否神奇地回來了？

**題目二：參數化查詢練習**

```sql
-- 查詢特定類別的商品
PREPARE stmt FROM 'SELECT * FROM products WHERE category = ?';
SET @cat = '高粱酒';
EXECUTE stmt USING @cat;
DEALLOCATE PREPARE stmt;
```

---

### 🟡 進階題

**題目三：體驗「鎖定」（需兩人一組）**

| 同學 A | 同學 B |
|--------|--------|
| `START TRANSACTION;` | — |
| `UPDATE products SET price = price + 10 WHERE product_id = 'P001';`（不 COMMIT）| — |
| — | 嘗試 `UPDATE` 同一筆商品 |
| — | 觀察畫面是否卡住（Waiting for lock）|

**題目四：意圖鎖定 (Intent Lock)**

> 請查閱課本第 15-5-2 節，解釋什麼是「意圖鎖定」，它如何提升鎖定效能？

---

### 😈 魔王挑戰題：完美退貨自動化程序

**需求說明**

店長希望有一個「一鍵退貨」功能，需在一個交易中完成：

1. 從 `order_details` 刪除該筆訂單
2. 將 `products` 的庫存加回去
3. 將 `vip_members` 的消費贈點扣回來

**過關標準**

- Streamlit 介面：輸入訂單 ID 即可執行退貨
- 若退貨導致會員點數變負數（點數已被使用），資料庫必須觸發錯誤並執行 `ROLLBACK`
- **絕對不允許會員點數變為負值**

**參考架構**

```python
def process_refund(conn, order_id):
    try:
        conn.begin()
        cursor = conn.cursor(pymysql.cursors.DictCursor)

        # Step 1：查詢訂單資訊
        cursor.execute("SELECT * FROM order_details WHERE order_id = %s", (order_id,))
        order = cursor.fetchone()
        if not order:
            raise ValueError(f"訂單 {order_id} 不存在")

        # Step 2：庫存加回
        cursor.execute(
            "UPDATE products SET stock = stock + %s WHERE product_id = %s",
            (order['quantity'], order['product_id'])
        )

        # Step 3：點數扣回（若點數不足，CHECK 條件會自動觸發錯誤）
        cursor.execute(
            "UPDATE vip_members SET point_balance = point_balance - %s WHERE customer_id = %s",
            (order['points_earned'], order['customer_id'])
        )

        # Step 4：刪除訂單
        cursor.execute("DELETE FROM order_details WHERE order_id = %s", (order_id,))

        conn.commit()
        return True, "退貨成功，所有資料已同步更新"

    except Exception as e:
        conn.rollback()
        return False, f"退貨失敗，資料已全數回滾：{e}"
```

---

## 📚 延伸閱讀

- MySQL 官方文件：[ACID and Transactions](https://dev.mysql.com/doc/refman/8.0/en/mysql-acid.html)
- MySQL 官方文件：[InnoDB Locking](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)
- MySQL 官方文件：[PREPARE Statement](https://dev.mysql.com/doc/refman/8.0/en/prepare.html)

---

*國立金門大學 · 資料庫管理系統 · 實作十五*
