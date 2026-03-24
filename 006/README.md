# 第六章：SQL 基礎查詢與 Python (Streamlit) 全端實作

**授課教師：劉鎮豪**
**課程：資料庫管理系統**
**學校：國立金門大學**

在了解了資料庫的建置與正規化後，我們進入資料庫操作的核心：**查詢 (Query)**。
本章節我們將以「金門特產進銷存系統」為例，學習如何使用 `SELECT` 語法從資料庫中精準地撈出需要的資訊。課程後半段，我們將結合 AI 輔助，並使用 Python (Streamlit) 打造出具備互動介面的前端網頁。

---

## 🛠️ 課前準備：建立測試資料庫 (Workbench 操作)

在開始練習查詢之前，請先在 MySQL Workbench 開啟新的查詢視窗，執行以下 SQL 語法，建立本次課程專用的「金門特產資料表 (`products`)」並匯入測試資料：

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

## 📖 第一部分：理論與 SQL 實戰 (請在 Workbench 測試)

要從資料庫拿資料，最基本的句型就是 `SELECT (要拿什麼欄位) FROM (從哪張表拿)`。實務上資料量很大，我們會搭配不同子句來過濾與排序。

### 1. 基礎查詢與別名 (SELECT, AS)

除了撈取原始欄位，我們可以直接在 `SELECT` 階段進行數學運算，並用 `AS` 取一個好讀的別名。

    ```sql
    -- 查詢特定欄位，並計算庫存總價值
    SELECT 
        product_name AS '商品名稱', 
        price AS '單價', 
        stock AS '庫存量',
        (price * stock) AS '庫存總值' 
    FROM products;
    ```

### 2. 條件過濾 (WHERE, AND, OR)

使用 `WHERE` 子句加上條件，告訴資料庫我們「只要哪些資料」。

    ```sql
    -- 找出屬於「伴手禮」且庫存小於 400 的商品
    SELECT * FROM products WHERE category = '伴手禮' AND stock < 400;
    ```

### 3. 進階過濾技巧 (LIKE, BETWEEN, IS NULL)

* **`BETWEEN`**：尋找某個範圍內的值。
* **`LIKE` (模糊搜尋)**：搭配 `%` (代表任意長度的字元) 使用。
* **`IS NULL`**：尋找沒有資料（空值）的欄位。

    ```sql
    -- 找出商品名稱包含「牛肉乾」的所有商品 (模糊搜尋)
    SELECT * FROM products WHERE product_name LIKE '%牛肉乾%';

    -- 找出價格尚未定價 (NULL) 的商品
    SELECT * FROM products WHERE price IS NULL;
    ```

### 4. 資料排序與去重 (ORDER BY, DISTINCT)

* **`ORDER BY`**：預設為 `ASC` (由小到大)，加上 `DESC` 代表由大到小。
* **`DISTINCT`**：消除重複的值，用於查詢資料表裡有哪些「種類」。

    ```sql
    -- 查詢店內有哪些商品類別 (去除重複)
    SELECT DISTINCT category FROM products;

    -- 列出所有商品，並依照價格「由高到低」排序
    SELECT product_name, price FROM products ORDER BY price DESC;
    ```

---

## 🤖 第二部分：AI 輔助開發 (生成前端 JS 互動元件)

在全端開發中，前端網頁通常會搭配 JavaScript (JS) 來實現「不需重新整理網頁，即可即時搜尋」的互動功能。現在，我們不需要從頭手刻複雜的語法，請嘗試將以下 Prompt (提示詞) 複製貼給 ChatGPT 或 Gemini，體驗 AI 輔助開發的威力：

> [!TIP]
> **請將以下指令複製貼給 AI 助理：**
>
> 「你現在是一位資深的網頁前端工程師。我正在設計一個『金門特產進銷存系統』。
> 1. 請幫我寫一個單一檔案的 HTML (包含 CSS 與 JavaScript)。
> 2. 畫面上要有表格，預設顯示幾筆假資料（例如：高粱酒、貢糖、牛肉乾，包含名稱、類別、價格）。
> 3. **核心功能：** 請用 JavaScript 實作一個『即時搜尋框』。當我在搜尋框輸入關鍵字（例如：貢糖）時，下方的表格要能『即時隱藏』不相關的商品，只留下符合的列。
> 4. 請加上精美的 CSS 樣式，讓它看起來像現代化的後台管理系統。」

同學可以將 AI 生成的程式碼存成 `index.html`，並用瀏覽器打開，親自體驗前端 JS 即時過濾的效果！

---

## 💻 第三部分：Python (Streamlit) 全端系統實作

了解 SQL 理論後，我們要將這些語法搬到 Python 程式中。請在 Spyder 中開啟 `app.py`，我們將逐步打造具備互動介面的進銷存查詢系統。

> [!IMPORTANT]
> **執行提醒：** 每次修改 `app.py` 並存檔後，請在 Anaconda Prompt 中執行 `streamlit run app.py`。後續只要在瀏覽器按重新整理即可看到更新。

### 步驟 1：建立基礎骨架與「資料總覽」功能

請複製以下程式碼貼入 `app.py`。這段程式碼包含了連線設定，以及對應 `SELECT *` 的資料表預覽功能。

    ```python
    import streamlit as st
    import pymysql
    import pandas as pd

    # --- 共用連線設定 ---
    def get_connection():
        return pymysql.connect(
            host="127.0.0.1",
            user="root",            # ⚠️ 請填入你的 MySQL 帳號
            password="1234",        # ⚠️ 請填入你的 MySQL 密碼
            database="kinmen_shop", # 連線到金門特產資料庫
            charset="utf8mb4"
        )

    st.title("🛒 金門在地特產進銷存系統")
    st.markdown("本系統展示如何結合 SQL 查詢語法與 Streamlit 網頁元件。")
    st.divider()

    # --- 區塊一：基礎查詢預覽 (SELECT *) ---
    st.header("1. 商品資料表總覽")

    if st.button("載入所有商品資料"):
        try:
            conn = get_connection()
            cursor = conn.cursor()
            
            cursor.execute("SELECT * FROM products")
            result = cursor.fetchall()
            
            if result:
                columns = [col[0] for col in cursor.description]
                df = pd.DataFrame(result, columns=columns)
                st.dataframe(df, use_container_width=True)
            else:
                st.info("目前沒有任何商品資料。")
                
        except Exception as e:
            st.error(f"❌ 查詢失敗：{e}")
        finally:
            if 'cursor' in locals(): cursor.close()
            if 'conn' in locals(): conn.close()
    ```

### 步驟 2：加入「依類別過濾 (WHERE)」功能

將下拉選單與 SQL 的 `WHERE` 條件結合。請將以下程式碼接續貼在 `app.py` 的最下方：

    ```python
    st.divider()

    # --- 區塊二：商品類別過濾 (WHERE 條件) ---
    st.header("2. 依商品類別過濾")

    # 建立下拉選單防呆
    selected_category = st.selectbox(
        "請選擇要查看的商品類別：", 
        ["酒類", "伴手禮", "肉品", "工藝品", "保健品"]
    )

    if st.button("查詢該類別商品"):
        try:
            conn = get_connection()
            cursor = conn.cursor()
            
            # 使用 %s 參數化查詢，防止 SQL Injection
            query = "SELECT * FROM products WHERE category = %s"
            cursor.execute(query, (selected_category,))
            result = cursor.fetchall()
            
            if result:
                columns = [col[0] for col in cursor.description]
                df = pd.DataFrame(result, columns=columns)
                st.success(f"✅ 找到 {len(df)} 筆屬於【{selected_category}】的商品！")
                st.dataframe(df, use_container_width=True)
            else:
                st.warning("該類別目前沒有商品。")
                
        except Exception as e:
            st.error(f"❌ 查詢失敗：{e}")
        finally:
            if 'cursor' in locals(): cursor.close()
            if 'conn' in locals(): conn.close()
    ```

### 步驟 3：加入「關鍵字搜尋與排序 (LIKE + ORDER BY)」功能

最後，我們加入模糊搜尋與價格排序功能。請將這段程式碼接續貼在最下方，存檔後重新整理網頁即可看到完整系統！

    ```python
    st.divider()

    # --- 區塊三：關鍵字搜尋與自動排序 (LIKE + ORDER BY) ---
    st.header("3. 商品關鍵字搜尋 (價格由低到高)")

    search_keyword = st.text_input("請輸入商品名稱關鍵字：", placeholder="例如: 貢糖")

    if st.button("搜尋商品"):
        if search_keyword:
            try:
                conn = get_connection()
                cursor = conn.cursor()
                
                # 結合模糊搜尋與排序
                query = "SELECT * FROM products WHERE product_name LIKE %s ORDER BY price ASC"
                cursor.execute(query, (f"%{search_keyword}%",))
                result = cursor.fetchall()
                
                if result:
                    columns = [col[0] for col in cursor.description]
                    df = pd.DataFrame(result, columns=columns)
                    st.success("✅ 成功找到符合的商品！")
                    st.dataframe(df, use_container_width=True)
                else:
                    st.warning("找不到符合關鍵字的商品，請嘗試其他關鍵字。")
                    
            except Exception as e:
                st.error(f"❌ 搜尋失敗：{e}")
            finally:
                if 'cursor' in locals(): cursor.close()
                if 'conn' in locals(): conn.close()
        else:
            st.warning("請先輸入關鍵字再點擊搜尋！")
    ```

---

## 📝 課堂隨堂練習 (Exercises)

請同學根據剛才建立的 `kinmen_shop` 資料庫，在 **MySQL Workbench** 中撰寫出對應的 SQL 查詢語法：

1. **基礎查詢：** 請列出所有「酒類」商品的名稱與庫存量。
2. **條件組合：** 由於庫存吃緊，請找出「庫存量低於 50」**或者**「價格大於 1000」的商品，提醒採購人員進貨。
3. **模糊搜尋與排序：** 請找出所有名稱中帶有「貢糖」的商品，並且依照價格**由便宜到昂貴**進行排序。
4. **進階挑戰：** 請問店內所有「非酒類」且「價格不是空值 (NULL)」的商品中，單價最貴的商品價格是多少？（提示：需結合 `WHERE`, `ORDER BY`, `LIMIT 1`）

> [!NOTE]
> 練習題的解答將於下週課程開頭公布，請同學務必親自在 Workbench 上敲擊語法測試！
