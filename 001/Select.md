# 實作二：網頁前台 UI 串接與資料庫查詢 (以 Sakila 為例)

**授課教師：劉鎮豪**
**課程：資料庫管理系統**

在確認純 Python 環境能成功連線後，我們將導入 `Streamlit` 網頁框架。本章節將使用 MySQL 內建的 `sakila` (影音出租店) 範例資料庫，帶領同學實作一個具備「下拉選單切換資料表」與「關鍵字搜尋」功能的現代化查詢網頁。

---

## 步驟一：安裝 Streamlit 並確認資料庫狀態

1. 請開啟 **Anaconda Prompt**，確認是否已安裝 Streamlit。若無，請執行以下指令：

    ```bash
    pip install streamlit
    ```

2. 請開啟 **MySQL Workbench**，確認左側清單中存在 `sakila` 資料庫。若該資料庫尚未啟用，請對著 `sakila` 按右鍵選擇 **Set as Default Schema**。

---

## 步驟二：撰寫 Streamlit 查詢系統程式碼

我們將設計一個包含兩個核心功能的查詢介面。請將這段程式碼寫入，作為前端介面與後端資料庫溝通的橋樑。

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
            
            # 執行查詢 (加上 LIMIT 100 避免資料量過大導致網頁卡頓)
            # 注意：資料表名稱通常無法使用參數化傳遞，所以我們用下拉選單限制使用者的選擇範圍以確保安全
            query = f"SELECT * FROM {table_name} LIMIT 100;"
            cursor.execute(query)
            result = cursor.fetchall()
            
            if result:
                # 抓取欄位名稱並建立 DataFrame
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
                
                # 💡 教學重點：使用「參數化查詢 (%s)」來防範 SQL Injection
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

> [!TIP]
> **資安小知識 (SQL Injection)**
> 在區塊二的程式碼中，我們使用了 `cursor.execute(query, (變數,))` 的寫法，而不是直接用字串把 SQL 語法拼起來。這樣能確保使用者輸入的內容只會被當成「純文字」處理，而不會被當作 SQL 指令執行，是保護資料庫的關鍵防線！
