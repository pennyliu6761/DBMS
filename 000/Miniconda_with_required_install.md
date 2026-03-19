# 課前準備：Miniconda 環境建置與資料庫連線測試

本章節將帶領同學安裝輕量版的 Python 開發環境 (Miniconda)，配置 Spyder 編輯器，並透過 `pymysql` 與 `pandas` 套件，測試是否能成功連線並讀取 MySQL 資料庫中的資料。

---

## 步驟一：下載與安裝 Miniconda

1. 前往 Anaconda 官方下載頁面，選擇下載 **Miniconda** (Windows 64-Bit Graphical Installer)。
2. 執行下載的安裝檔，點擊 `Next` 並同意授權條款 (`I Agree`)。
3. 在 Installation Type 步驟，選擇預設的 **Just Me (recommended)**。
4. 在 Choose Install Location 步驟，選擇您要安裝的路徑（建議保持預設），點擊 `Next`。
5. 在 Advanced Installation Options 步驟，建議勾選以下選項，然後點擊 `Install`：
    - [x] Create shortcuts (supported packages only).
    - [x] Clear the package cache upon completion
6. 等待安裝進度條跑完後，點擊 `Next` 並 `Finish` 完成安裝。

---

## 步驟二：安裝必要套件 (Spyder 與連線工具)

我們需要透過終端機安裝程式編輯器 (Spyder) 以及資料庫處理相關的套件。請注意，本次教學我們將使用 `pymysql` 作為連線驅動。

1. 點擊 Windows 開始選單，搜尋並開啟 **Anaconda Prompt**。
2. 在終端機視窗中，輸入以下指令並按 Enter 執行：

    ```bash
    conda install spyder pymysql pandas sqlalchemy
    ```

3. 系統會開始解析套件，當畫面出現 `Proceed ([y]/n)?` 時，請輸入 `y` 並按 Enter 繼續安裝，等待所有套件下載與安裝完成。

---

## 步驟三：啟動 Spyder 編輯器

1. 再次打開 Windows 開始選單，搜尋 **Spyder** 並點擊開啟應用程式。
2. 第一次開啟可能會需要一點時間載入，請耐心等待直到出現程式編輯介面。

---

## 步驟四：撰寫 Python 資料庫讀取測試

接下來，我們將使用最簡潔的原生 `pymysql` 語法來建立連線，並透過游標 (Cursor) 讀取 MySQL 中的資料，最後用 `pandas` 將結果整齊地顯示出來。

1. 在 Spyder 中開啟一個新檔案（或暫存檔 `temp.py`）。
2. 複製並貼上以下程式碼。請務必將**密碼**、**資料庫名稱**與**資料表名稱**修改為您自己的設定：

    ```python
    import pymysql
    import pandas as pd

    # 1. 使用 try...except 捕捉可能的連線錯誤
    try:
        # 建立資料庫連線
        conn = pymysql.connect(
            host="127.0.0.1",
            user="root",          # ⚠️ 請填入你的帳號
            password="1234",      # ⚠️ 請填入你的密碼
            database="sakila",    # ⚠️ 請填入資料庫名稱 sakila是 MySQL Workbench內建資料庫
            charset="utf8mb4"     # 確保中文編碼正確
        )
        
        # 2. 建立游標 (Cursor) 來執行 SQL 指令
        cursor = conn.cursor()
        
        # 3. 執行 SELECT 查詢
        cursor.execute("SELECT * FROM Language")  # ⚠️ 請換成想讀取的資料表名稱
        result = cursor.fetchall()          # 取得所有查詢結果
        
        # 4. 取得欄位名稱，並轉換成 Pandas DataFrame 方便觀看
        if result:
            columns = [col[0] for col in cursor.description]
            df = pd.DataFrame(result, columns=columns)
            
            print("✅ 資料庫連線與讀取成功！")
            print("以下為前五筆資料：")
            print(df.head())
        else:
            print("連線成功，但資料表中目前沒有資料。")
            
    except Exception as e:
        # 若發生錯誤 (如密碼打錯、沒開 MySQL Server)，會顯示紅色叉叉與錯誤訊息
        print(f"❌ 發生錯誤：{e}")
        
    finally:
        # 5. 無論成功或失敗，最後一定要關閉游標與連線，釋放系統資源
        if 'cursor' in locals(): 
            cursor.close()
        if 'conn' in locals(): 
            conn.close()
    ```

3. 點擊上方的「綠色播放鍵 (Run file)」執行程式。
4. 觀察右下角的 Console (控制台) 視窗。如果連線與語法皆正確，您應該會看到 `✅ 資料庫連線與讀取成功！` 以及資料表的內容。
---

## 🔧 常見錯誤排除 (Troubleshooting)

在測試執行的過程中，如果右下角的 Console 出現紅色的錯誤訊息，請對照以下常見狀況進行排除：

- **錯誤訊息包含 `Access denied for user 'root'@'localhost'`**
  
    > [!WARNING]
    > 這代表您的**帳號或密碼輸入錯誤**。請回到 Python 程式碼中，確認 `password = '...'` 的單引號內是否正確填寫了您安裝 MySQL 時設定的密碼。

- **錯誤訊息包含 `Unknown database 'sakila'`**
  
    :::info
    這代表找不到指定的資料庫。請開啟您的 **MySQL Workbench**，確認左側 Schemas 列表欄中，是否真的存在名為 `sakila` 的資料庫。
    :::

- **錯誤訊息包含 `Table 'sakila.Language' doesn't exist`**
  
    :::info
    這代表您的資料庫中沒有這個資料表。請確認您的 SQL 語法 `SELECT * FROM Language` 中的表名是否正確，或是您是否尚未在 Workbench 中建立該資料表。
    :::
