# 實作十一：檢視表的建立與應用 (虛擬資料表與資料安全)

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

-- 假設成本是售價的一半
SET SQL_SAFE_UPDATES = 0; -- 暫時關閉安全模式
UPDATE products SET cost = price * 0.5;
SET SQL_SAFE_UPDATES = 1;  -- 執行完畢後再開回來
```

<img width="367" height="172" alt="image" src="https://github.com/user-attachments/assets/acfaae01-6cf6-48a1-8122-093abe06254b" />

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

<img width="191" height="240" alt="image" src="https://github.com/user-attachments/assets/085ae62f-a5b5-4e09-9c98-f09f3a9450a2" />
<img width="465" height="537" alt="image" src="https://github.com/user-attachments/assets/81c34400-e7bc-4a84-9ce9-0a8afb2a5141" />

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

<img width="198" height="243" alt="image" src="https://github.com/user-attachments/assets/191908c2-c239-44f8-a370-6a4b5550dabf" />
<img width="344" height="98" alt="image" src="https://github.com/user-attachments/assets/5b25a92b-967a-49bc-bd9a-36d179988a3f" />

### 2. 修改與刪除檢視表 (ALTER / DROP VIEW)
檢視表的修改與刪除非常輕量，因為它只是刪除或覆寫一段查詢邏輯，不會影響到底層的真實資料。

```sql
-- 修改檢視表 (例如：工讀生也不需要看到 stock 庫存欄位了)
ALTER VIEW view_safe_products AS
SELECT product_id, product_name, category, price 
FROM products;
```

<img width="351" height="158" alt="image" src="https://github.com/user-attachments/assets/56dccb2e-2591-44c3-bbaf-cab3153f60a6" />
<img width="308" height="162" alt="image" src="https://github.com/user-attachments/assets/868ba31d-1235-4618-bbc5-7d47db14a6a6" />

```sql
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
```

<img width="348" height="79" alt="image" src="https://github.com/user-attachments/assets/6b35a80a-da9a-488a-9e2d-a7916051f58d" />

```sql
-- ⭕ 成功：透過 View 新增一個 150 元的商品
INSERT INTO view_budget_items (product_id, product_name, price) 
VALUES ('P010', '小包裝貢糖', 150);
```

<img width="346" height="97" alt="image" src="https://github.com/user-attachments/assets/bcd5a826-ed23-4b4a-b588-f084bad38de2" />

```sql
-- ❌ 失敗報錯：企圖透過此 View 新增一個 500 元的商品
-- 因為違反了 WHERE price <= 200 的條件，會被 WITH CHECK OPTION 擋下！
INSERT INTO view_budget_items (product_id, product_name, price) 
VALUES ('P011', '豪華高粱', 500); 
```

<img width="501" height="20" alt="image" src="https://github.com/user-attachments/assets/9553f9fa-b511-41e9-8265-83e4c531ff82" />
<img width="390" height="22" alt="image" src="https://github.com/user-attachments/assets/aaa7ef64-bb7e-447b-a10e-0383c4af34a0" />

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

<img width="1785" height="663" alt="image" src="https://github.com/user-attachments/assets/27b18b4d-6c8b-4ef2-bd74-583b7cdf1a92" />
<img width="699" height="543" alt="image" src="https://github.com/user-attachments/assets/ef7a6bf8-7a8e-4bc0-b56c-fb2859f2293d" />
<img width="692" height="528" alt="image" src="https://github.com/user-attachments/assets/99b60132-2957-4969-b1e5-ebf140a51eb5" />
<img width="1059" height="476" alt="image" src="https://github.com/user-attachments/assets/67af01c6-fb2c-451b-b81f-34cf009c1e42" />

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

---

# 實作十一補充：檢視表對外開放與權限管理 (最小權限原則 × 系統介接實戰)

**授課教師：劉鎮豪**
**課程：資料庫管理系統**
**學校：國立金門大學**

在上一節課，我們建立了 `view_safe_products`，成功把「進貨成本」藏起來。但問題來了：檢視表建好了，要怎麼安全地把它「開放」給另一支程式或外部系統使用？

直接把 `root` 帳號密碼給外部系統？🚨 **這是資安大忌！**

本單元我們將學習真實業界的做法：在 MySQL 層建立**專屬的唯讀帳號**，精準授權它只能存取特定的 View，再用 Python 以這個受限帳號連線，完成整個「安全介接鏈」。

---

## 🧭 本單元學習地圖

```
建立 View (上節課) 
    ↓
建立專屬 DB 帳號 (本節 Part 1)
    ↓
GRANT 精準授權 (本節 Part 2)
    ↓
Python 以受限帳號連線 (本節 Part 3)
    ↓
驗證邊界與錯誤處理 (本節 Part 4)
```

---

## 📖 第一部分：為什麼不能用 root？

| 帳號 | 能存取的範圍 | 風險 |
|------|------------|------|
| `root` | 整個 MySQL 伺服器的所有資料庫 | 外洩即全損 |
| `readonly_pos` (本節課建立) | 僅 `kinmen_shop.view_safe_products` | 外洩也只能讀一張 View |

**最小權限原則 (Principle of Least Privilege)**：給程式或人員「恰好夠用」的權限，不多一分。這是業界資安的基本守則，ISO 27001、GDPR 合規都要求這一點。

---

## 🛠️ 第二部分：建立帳號與授權 (在 MySQL Workbench 以 root 執行)

### 步驟一：建立專屬帳號

```sql
-- 建立一個只能從本機登入的帳號，密碼設強一點
-- 'readonly_pos'@'localhost' 的意思：
--   帳號名稱 = readonly_pos
--   只允許從本機 (localhost) 連線，無法從外部 IP 登入
CREATE USER 'readonly_pos'@'localhost' IDENTIFIED BY 'SecurePass2024!';
```

> **💡 帳號後面為什麼要加 `@'localhost'`？**
> MySQL 的帳號是「帳號 + 來源 IP」的組合。
> - `'app'@'localhost'` — 只有本機可以用這個帳號登入
> - `'app'@'%'` — 任意 IP 都可以登入（風險較高，需搭配防火牆）
> - `'app'@'192.168.1.50'` — 只允許指定的應用程式伺服器 IP

### 步驟二：授予精準權限

```sql
-- 只給 SELECT 權限，且只限定在 kinmen_shop 資料庫的 view_safe_products
GRANT SELECT ON kinmen_shop.view_safe_products TO 'readonly_pos'@'localhost';

-- 若有多張 View 需要開放，逐一授權：
GRANT SELECT ON kinmen_shop.view_customer_orders TO 'readonly_pos'@'localhost';

-- 讓權限立即生效
FLUSH PRIVILEGES;
```

### 步驟三：確認授權結果

```sql
-- 查看這個帳號目前擁有哪些權限
SHOW GRANTS FOR 'readonly_pos'@'localhost';
```

執行後應該看到：
```
GRANT SELECT ON `kinmen_shop`.`view_safe_products` TO `readonly_pos`@`localhost`
GRANT SELECT ON `kinmen_shop`.`view_customer_orders` TO `readonly_pos`@`localhost`
```

### 步驟四：撤銷權限 (REVOKE)

```sql
-- 如果工讀生離職，或系統下線，立即撤銷
REVOKE SELECT ON kinmen_shop.view_safe_products FROM 'readonly_pos'@'localhost';

-- 如果要完全刪除帳號
DROP USER IF EXISTS 'readonly_pos'@'localhost';
FLUSH PRIVILEGES;
```

> **⚠️ 注意：** `REVOKE` 只撤銷權限，帳號還在。`DROP USER` 才是連帳號一起刪除。離職處理通常先 `REVOKE` 確認業務不受影響，再 `DROP USER`。

---

## 📖 第三部分：理解「授權 View」vs「授權 Table」的差異

這是本節課最重要的觀念：

```sql
-- ❌ 危險做法：直接授權原始資料表
GRANT SELECT ON kinmen_shop.products TO 'readonly_pos'@'localhost';
-- 結果：外部程式可以 SELECT * FROM products → 看到 cost 成本欄位！

-- ✅ 安全做法：只授權 View
GRANT SELECT ON kinmen_shop.view_safe_products TO 'readonly_pos'@'localhost';
-- 結果：外部程式只能看到 view_safe_products 定義的欄位，cost 永遠隱藏
```

**結論：View 是資料的「公開介面」，Base Table 是「私有實作」。** 這個設計概念和軟體工程的 API 設計思維完全一致。

---

## 💻 第四部分：Python 以受限帳號連線並讀取 View

### 4-1 驗證帳號確實受限 (先在 Workbench 測試)

用 `readonly_pos` 帳號連線後，試著存取它沒被授權的東西：

```sql
-- 用 readonly_pos 帳號執行以下，應該全部報錯
SELECT * FROM products;           -- ❌ 沒有這張表的權限
INSERT INTO view_safe_products ...; -- ❌ 只有 SELECT，沒有 INSERT
SELECT * FROM kinmen_shop.orders; -- ❌ 完全不認識這張表
```

確認邊界清楚後，再接 Python。

### 4-2 完整 Python 介接程式

```python
import streamlit as st
import pymysql
import pandas as pd

# ============================================================
# 連線設定 — 使用受限帳號，不再是 root！
# ============================================================
def get_readonly_connection():
    """以唯讀帳號建立連線，只能存取被授權的 View"""
    return pymysql.connect(
        host="127.0.0.1",
        user="readonly_pos",          # ← 受限帳號
        password="SecurePass2024!",   # ← 對應密碼
        database="kinmen_shop",
        charset="utf8mb4"
    )

# ============================================================
# Streamlit 前端
# ============================================================
st.set_page_config(layout="wide")
st.title("🔌 外部系統介接示範 (最小權限原則)")
st.markdown("""
連線帳號：`readonly_pos`　｜　授權範圍：僅 `view_safe_products`、`view_customer_orders`  
> 這模擬的是：一支外部的 POS 程式或合作夥伴系統，以受限帳號讀取我們開放的資料介面。
""")
st.divider()

tab1, tab2, tab3 = st.tabs(["📋 商品型錄", "📦 訂單紀錄", "🚧 越權存取測試"])

# --- Tab 1：讀取被授權的 View ---
with tab1:
    st.subheader("從 view_safe_products 讀取（有權限）")
    try:
        conn = get_readonly_connection()
        df = pd.read_sql("SELECT * FROM view_safe_products", conn)
        st.success(f"✅ 成功讀取 {len(df)} 筆商品資料")
        st.dataframe(df, use_container_width=True)
        
        # 明確顯示「看不到的欄位」
        st.info("ℹ️ 注意：此查詢結果中沒有 `cost`（進貨成本）欄位，該欄位對此帳號完全不可見。")
    except Exception as e:
        st.error(f"連線失敗：{e}")
    finally:
        if 'conn' in locals(): conn.close()

# --- Tab 2：讀取另一張被授權的 View ---
with tab2:
    st.subheader("從 view_customer_orders 讀取（有權限）")
    try:
        conn = get_readonly_connection()
        df = pd.read_sql("SELECT * FROM view_customer_orders", conn)
        st.success(f"✅ 成功讀取 {len(df)} 筆訂單資料")
        st.dataframe(df, use_container_width=True)
    except Exception as e:
        st.error(f"連線失敗：{e}")
    finally:
        if 'conn' in locals(): conn.close()

# --- Tab 3：越權存取測試（教學用） ---
with tab3:
    st.subheader("🚧 越權存取測試（觀察錯誤訊息）")
    st.warning("以下按鈕會故意存取沒有權限的資源，用來驗證帳號邊界是否真的有效。")
    
    col1, col2 = st.columns(2)
    
    with col1:
        if st.button("嘗試讀取 products 原始表（含成本）"):
            try:
                conn = get_readonly_connection()
                df = pd.read_sql("SELECT * FROM products", conn)
                # 如果走到這裡，代表授權設定有問題！
                st.error("⚠️ 警告：竟然讀到了！請檢查 GRANT 設定是否正確。")
                st.dataframe(df)
            except pymysql.err.OperationalError as e:
                if e.args[0] == 1142:  # SELECT command denied
                    st.success("✅ 權限邊界正常！資料庫拒絕存取原始表。")
                    st.code(f"MySQL Error 1142: {e.args[1]}", language="text")
                else:
                    st.error(f"其他錯誤：{e}")
            finally:
                if 'conn' in locals(): conn.close()
    
    with col2:
        if st.button("嘗試 INSERT 進 View（唯讀帳號）"):
            try:
                conn = get_readonly_connection()
                cursor = conn.cursor()
                cursor.execute(
                    "INSERT INTO view_safe_products (product_id, product_name, price) VALUES (%s, %s, %s)",
                    ('P099', '測試商品', 100)
                )
                conn.commit()
                st.error("⚠️ 警告：INSERT 竟然成功了！請確認 GRANT 只有 SELECT。")
            except pymysql.err.OperationalError as e:
                if e.args[0] == 1142:  # INSERT command denied
                    st.success("✅ 權限邊界正常！唯讀帳號無法寫入。")
                    st.code(f"MySQL Error 1142: {e.args[1]}", language="text")
                else:
                    st.error(f"其他錯誤：{e}")
            finally:
                if 'cursor' in locals(): cursor.close()
                if 'conn' in locals(): conn.close()
```

---

## ❓ 第五部分：常見問題與排解 (FAQ)

1. **Q: 授權給 View 的帳號，能不能「穿透」存取底層的 Base Table？**
   * A: 不行。MySQL 對 View 的授權是獨立計算的。`readonly_pos` 被授予 `SELECT ON view_safe_products`，但對 `products` 這張 Base Table 完全沒有權限。即使他知道 Base Table 的名稱，直接查詢也會被 Error 1142 擋下。這就是授權在 View 層的安全意義。

2. **Q: Python 程式的帳號密碼寫在程式碼裡，這樣安全嗎？**
   * A: 教學環境可以接受，但正式上線時應改用**環境變數**或**.env 設定檔**，並加入 `.gitignore`，絕對不能把密碼 commit 進 Git。Python 可用 `python-dotenv` 套件讀取 `.env`。這是業界強制要求的做法。

3. **Q: 可以同時授權給多個帳號嗎？**
   * A: 可以。例如同時建立 `pos_cashier`（只能讀商品 View）、`report_system`（可讀訂單統計 View）、`partner_api`（只能讀公開型錄 View）。每個系統或角色有自己的帳號，是真實業界的標準做法，稱為 **Service Account 管理**。

4. **Q: `FLUSH PRIVILEGES` 是什麼？每次都要下嗎？**
   * A: `GRANT` / `REVOKE` / `CREATE USER` 這些指令通常會自動生效，不一定需要 `FLUSH PRIVILEGES`。但若直接修改 `mysql.user` 系統資料表（進階情境），則必須執行。養成下的習慣是安全的。

---

## 📝 第六部分：實戰演練

### 🟢 基礎題

1. **建立報表帳號：** 建立一個名為 `report_viewer` 的帳號，授予他讀取 `view_category_value`（上節課進階題建立的類別庫存價值 View）的權限，並用 `SHOW GRANTS` 確認。

2. **測試邊界：** 用 `report_viewer` 帳號連線 Workbench，試著 `SELECT * FROM products` 與 `SELECT * FROM view_category_value`，截圖兩個不同的結果（一個失敗、一個成功）。

### 🟡 進階題

3. **Python 動態帳號切換：** 修改 Part 4 的程式，在 `st.sidebar` 加入帳號選擇（`root` / `readonly_pos` / `report_viewer`），根據選擇動態切換連線帳號，並顯示「當前帳號可存取的 View 清單」。

4. **撤銷與重授：** 對 `readonly_pos` 執行 `REVOKE`，用 Python 程式測試連線後會收到什麼錯誤（Error Code 是什麼？），再 `GRANT` 回來，驗證連線恢復正常。

### 😈 魔王挑戰題

5. **多租戶資料隔離：** 假設我們有兩家加盟店共用同一個資料庫，各自只能看自己的訂單資料。請設計：
   - 兩個 View：`view_store_A_orders` 與 `view_store_B_orders`（用 `WHERE store_id = 'A'` 篩選）
   - 兩個帳號：`app_store_a` 與 `app_store_b`，各自只被授予對應的 View
   - 用 Python 驗證：`app_store_a` 讀不到 Store B 的訂單，`app_store_b` 讀不到 Store A 的訂單

   這個架構是 SaaS 系統「多租戶隔離」的最基礎實作，也是面試常被問到的設計題！

---

## 🗺️ 知識延伸：完整的安全介接鏈全貌

```
[外部系統 / Python 程式]
        ↓ 以 readonly_pos 帳號連線 (只有 SELECT 權限)
[MySQL 帳號權限層]
        ↓ 只開放特定 View 的存取
[View 層 (虛擬資料表)]
        ↓ View 定義了哪些欄位、哪些列可以被看到
[Base Table 層 (原始資料)]
        ← cost、密碼、個資等機密欄位永遠不出這一層
```

每一層都是一道防線。真實的生產環境中，還會在最外層加上 API Gateway、JWT Token 驗證等應用層防護，但資料庫層的帳號權限管理永遠是最後一道、也是最可靠的一道防線。
