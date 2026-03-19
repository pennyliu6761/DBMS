# 實作二：網頁前台 UI 串接與資料庫查詢 (以 Sakila 為例)

**授課教師：劉鎮豪**
**課程：資料庫管理系統**

在確認純 Python 環境能成功連線後，我們將導入 `Streamlit` 網頁框架。本章節將使用 MySQL 內建的 `sakila` (影音出租店) 範例資料庫，帶領同學實作一個具備「下拉選單切換資料表」與「關鍵字搜尋」功能的現代化查詢網頁。

---

## 📖 課前觀念與 SQL 語法解析

在正式撰寫 Python 程式碼之前，我們先來了解本次實作會用到的核心資料庫觀念，以及前端 UI 是如何與後端 SQL 語法互相配合的。

### 1. 動態資料表預覽與 LIMIT 限制

第一個功能是讓使用者透過下拉選單選擇資料表，並在網頁上顯示內容。

- **對應 UI 元件**：下拉選單 (`st.selectbox`) 與 資料表格 (`st.dataframe`)
- **核心 SQL 語法**：
    ```sql
    SELECT * FROM film LIMIT 100;
    ```
    
> [!NOTE]
> **為什麼要加 `LIMIT`？**
> 在實務的網頁開發中，資料庫的資料量可能高達數百萬筆。如果前端直接執行 `SELECT *` 而不加限制，會導致伺服器記憶體耗盡或網頁卡死。因此，在做資料預覽時，強制加上 `LIMIT` (例如只撈前 100 筆) 是非常重要的效能保護機制。

### 2. 條件式關鍵字搜尋 (WHERE + LIKE)

第二個功能是讓使用者輸入關鍵字，找出名稱中包含該關鍵字的電影。

- **對應 UI 元件**：文字輸入框 (`st.text_input`)
- **核心 SQL 語法**：
    ```sql
    SELECT title, release_year FROM film WHERE title LIKE '%LOVE%';
    ```
- **語法解析**：`LIKE` 運算子搭配 `%` (百分比符號) 可以進行模糊搜尋。`%LOVE%` 代表不管 `LOVE` 前面或後面是什麼字元，只要中間包含這四個字母就會被撈出來。

### 3. 🛡️ 核心資安觀念：防範 SQL Injection (資料庫隱碼攻擊)

這是資料庫程式設計中**最重要的一課**！當我們接收使用者的輸入（例如關鍵字）並送往資料庫時，**絕對不可以**直接用字串拼接的方式組合 SQL 語法。

- ❌ **危險的寫法 (純字串拼接)**：
    ```python
    # 如果使用者故意在輸入框打上： ' OR 1=1 --
    user_input = "' OR 1=1 --"
    query = f"SELECT * FROM film WHERE title = '{user_input}'"
    # 這會讓語法變成 SELECT * FROM film WHERE title = '' OR 1=1 --'
    # 導致駭客可以繞過驗證，撈出整張資料表的所有機密！
    ```

- ✅ **安全的寫法 (參數化查詢 Parameterized Query)**：
    ```python
    query = "SELECT * FROM film WHERE title LIKE %s"
    # 我們在 SQL 語法中留下 %s 當作「佔位符」
    # 在執行時，才由 pymysql 套件安全地將變數塞進去
    cursor.execute(query, (f"%{user_input}%",))
    ```

> [!WARNING]
> **永遠不要相信使用者的輸入！** 養成使用 `%s` 參數化查詢的習慣，是成為合格資料庫管理員的第一步。

---

## 步驟一：安裝 Streamlit 並確認資料庫狀態

1. 請開啟 **Anaconda Prompt**，確認是否已安裝 Streamlit。若無，請執行以下指令：

    ```bash
    pip install streamlit
    ```

2. 請開啟 **MySQL Workbench**，確認左側清單中存在 `sakila` 資料庫。若該資料庫尚未啟用，請對著 `sakila` 按右鍵選擇 **Set as Default Schema**。

---

## 步驟二：撰寫 Streamlit 查詢系統程式碼

了解理論後，我們來實際將 UI 與 SQL 結合。

1. 在 Spyder 中建立一個新檔案，命名為 `app.py`（請務必存檔）。
2. 複製並貼上以下程式碼，並請將**密碼**修改為您自己的 MySQL 密碼：

    ```python
    import streamlit as st
    import pymysql
    import pandas as pd

    # 1. 建立資料庫連線的共用函式
    def get_connection():
        return pymysql.connect(
            host="127.0.0.1",
            user="root",          # ⚠️ 請填入你的帳號
            password="1234",      # ⚠️ 請填入你的密碼
            database="sakila",    # 使用內建的 sakila 資料庫
            charset="utf8mb4"
        )

    # 2. 網頁標題設計
    st.title("🎬 影音出租店 (Sakila) 查詢系統")
    st.markdown("本系統展示如何透過 Streamlit 與 pymysql 進行動態資料庫查詢。")

    # ==========================================
    # 區塊一：動態選擇資料表預覽 (使用 SELECT *)
    # ==========================================
    st.header("1. 資料表總覽")

    # 建立下拉選單，讓使用者選擇要看的資料表
    table_name = st.selectbox("請選擇要查看的資料表：", ["film", "actor", "customer", "category"])

    # 建立按鈕
    if st.button("載入資料表"):
        try:
            conn = get_connection()
            cursor = conn.cursor()
            
            # 執行查詢 (加上 LIMIT 100 避免資料庫過載)
            query = f"SELECT * FROM {table_name} LIMIT 100;"
            cursor.execute(query)
            result = cursor.fetchall()
            
            if result:
                columns = [col[0] for col in cursor.description]
                df = pd.DataFrame(result, columns=columns)
                st.dataframe(df, use_container_width=True)
            else:
                st.info("該資料表目前沒有資料。")
                
        except Exception as e:
            st.error(f"❌ 查詢失敗：{e}")
        finally:
            if 'cursor' in locals(): cursor.close()
            if 'conn' in locals(): conn.close()

    st.divider() # 畫一條分隔線

    # ==========================================
    # 區塊二：電影關鍵字搜尋 (使用 WHERE LIKE)
    # ==========================================
    st.header("2. 電影關鍵字搜尋")

    # 建立文字輸入框
    search_keyword = st.text_input("請輸入電影名稱 (Title) 關鍵字：", placeholder="例如: LOVE")

    if st.button("搜尋電影"):
        if search_keyword:
            try:
                conn = get_connection()
                cursor = conn.cursor()
                
                # 💡 實踐資安觀念：使用「參數化查詢 (%s)」
                query = "SELECT film_id, title, release_year, rental_rate FROM film WHERE title LIKE %s"
                
                # 在執行時才將變數塞入 %s 的位置，並加上 % 作為模糊搜尋
                cursor.execute(query, (f"%{search_keyword}%",))
                result = cursor.fetchall()
                
                if result:
                    columns = [col[0] for col in cursor.description]
                    df = pd.DataFrame(result, columns=columns)
                    st.success(f"✅ 成功找到 {len(df)} 筆符合的電影！")
                    st.dataframe(df, use_container_width=True)
                else:
                    st.warning("找不到符合關鍵字的電影，請嘗試其他關鍵字。")
                    
            except Exception as e:
                st.error(f"❌ 搜尋失敗：{e}")
            finally:
                if 'cursor' in locals(): cursor.close()
                if 'conn' in locals(): conn.close()
        else:
            st.warning("請先輸入關鍵字再點擊搜尋！")
    ```

---

## 步驟三：啟動網頁伺服器並進行測試

> [!WARNING]
> **注意：** Streamlit 程式**不可**在 Spyder 內直接點擊執行 (Run file)，必須透過終端機啟動。

1. 回到 **Anaconda Prompt**。
2. 使用 `cd` 指令切換到存放 `app.py` 的資料夾（例如桌面）。
    ```bash
    cd Desktop
    ```
3. 輸入以下指令啟動系統：
    ```bash
    streamlit run app.py
    ```
4. 成功後，瀏覽器會自動彈出 `http://localhost:8501`。
    <img width="743" height="633" alt="image" src="https://github.com/user-attachments/assets/ab1c56c7-1e09-4fe0-a5f0-dc50e2563edb" />

5. **測試任務：**
    - 在「區塊一」切換不同的資料表，觀察下方表格是否會自動重新渲染。
      <img width="721" height="604" alt="image" src="https://github.com/user-attachments/assets/5716fbc0-a389-4013-9284-d9049ae18ede" />
      <img width="723" height="609" alt="image" src="https://github.com/user-attachments/assets/19f9841e-764d-4d6e-9284-318d1a833ee0" />

    - 在「區塊二」輸入 `DINOSAUR`，測試模糊搜尋是否能正確從 `film` 資料表中找出包含該單字的電影。
      <img width="737" height="839" alt="image" src="https://github.com/user-attachments/assets/1e474320-db79-432c-b454-862e5473a885" />
