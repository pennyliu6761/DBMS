# 實作一：Python 與 MySQL 資料庫連線測試

**授課教師：劉鎮豪**
**課程：資料庫管理系統**

本章節將帶領同學使用 Python 建立一個具備現代化前端介面（UI）的資料庫連線測試程式。我們將透過 `Streamlit` 框架建立網頁介面，並使用 `PyMySQL` 套件與本機端的 MySQL 資料庫進行連線。

## 📝 課前準備

在開始撰寫程式之前，請確認您已具備以下環境：
- [x] 已安裝 **Anaconda** (包含 Spyder)
- [x] 已安裝 **MySQL Server** 與 **MySQL Workbench**

---

## 步驟一：建立測試資料庫 (MySQL Workbench)

首先，我們需要在 MySQL 中建立一個供本次實作使用的空資料庫。

1. 打開 **MySQL Workbench** 並連線至您的本機伺服器 (Local instance)。
2. 在查詢視窗 (Query Tab) 中輸入以下 SQL 語法：

    ```sql
    -- 建立一個名為 demo_db 的資料庫
    CREATE DATABASE demo_db;
    ```

3. 點擊閃電圖示 ⚡ 執行該語法，並在下方的 Output 視窗確認是否執行成功。

---

## 步驟二：安裝必要套件 (Anaconda Prompt)

我們需要安裝負責處理前端網頁介面的 `streamlit`，以及負責連接資料庫的 `pymysql`。

1. 從 Windows 開始選單中，找到並打開 **Anaconda Prompt**。
2. 貼上並執行以下指令進行安裝：

    ```bash
    pip install streamlit pymysql
    ```

    :::info
    **💡 提示：** 若畫面顯示 `Successfully installed...` 即代表安裝完成。
    :::

---

## 步驟三：撰寫前端連線程式 (Spyder)

接下來，我們將撰寫 Python 程式碼來建立連線按鈕與狀態標籤。

1. 打開 **Spyder**。
2. 開啟一個新的檔案，並**務必先存檔**，命名為 `app.py`（建議存在桌面或您好找的資料夾）。
3. 複製並貼上以下完整程式碼：

    ```python
    import streamlit as st
    import pymysql

    # 1. 設定網頁標題與說明
    st.title("🔌 資料庫連線測試")
    st.markdown("請點擊下方按鈕測試 MySQL 連線狀態。")

    # 2. 建立一個觸發按鈕
    if st.button("測試資料庫連線"):
        
        # 使用 try...except 來捕捉可能的連線錯誤
        try:
            # 嘗試建立資料庫連線
            conn = pymysql.connect(
                host="localhost",
                user="root",          # ⚠️ 請填入你自己的 MySQL 帳號
                password="password",  # ⚠️ 請填入你自己的 MySQL 密碼
                database="demo_db",   # 步驟一建立的資料庫
                charset="utf8mb4"     # 確保中文編碼正確
            )
            
            # 若能順利執行到此行，代表連線成功，顯示綠色成功標籤
            st.success("✅ 資料庫連線成功！")
            
            # 測試完畢，關閉連線釋放系統資源
            conn.close() 
            
        except Exception as e:
            # 若連線失敗，顯示紅色錯誤標籤與錯誤詳細訊息
            st.error(f"❌ 連線失敗，錯誤訊息：{e}")
    ```

---

## 步驟四：啟動網頁伺服器並測試

:::warning
**⚠️ 注意：** 請不要在 Spyder 裡面按綠色播放鍵執行！Streamlit 是一個網頁框架，必須透過終端機啟動。
:::

1. 回到 **Anaconda Prompt**。
2. 使用 `cd` 指令切換到你存放 `app.py` 的資料夾。例如，如果存在桌面：

    ```bash
    cd Desktop
    ```

3. 輸入以下指令啟動 Streamlit 伺服器：

    ```bash
    streamlit run app.py
    ```

4. 成功後，瀏覽器會自動彈出一個網頁 (`http://localhost:8501`)。
5. 點擊網頁上的 **「測試資料庫連線」** 按鈕，看看是否出現綠色的成功標籤吧！

<details>
<summary><b>如果出現紅色的錯誤訊息怎麼辦？（點擊展開）</b></summary>

- **Access denied for user**: 代表你的 `user` 或 `password` 打錯了，請回到 `app.py` 檢查。
- **Unknown database**: 代表步驟一的 `demo_db` 沒有建立成功，請回到 Workbench 重新執行 SQL 語法。
- **Can't connect to MySQL server on 'localhost'**: 代表你的 MySQL 伺服器沒有啟動，請檢查 MySQL 服務是否正在運行。
</details>
