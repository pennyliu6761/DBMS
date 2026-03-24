# 第十六章：MySQL/MariaDB 用戶端程式開發 — 使用 Python 與 PHP

**授課教師：劉鎮豪**
**課程：資料庫管理系統**
**學校：國立金門大學**

恭喜大家來到了資料庫課程的最後一站！在前面的章節中，我們已經學會了如何設計、查詢與維護資料庫。現在，我們要學習如何編寫「用戶端程式 (Client-side Application)」，讓資料庫真正地為終端使用者服務。



本章節將介紹兩種業界最主流的開發語言：**Python** 與 **PHP**。我們將探討如何使用 Python 進行高效的後端處理，以及如何利用 PHP 建立動態網頁，打造出一個跨平台的「金門特產店綜合管理系統」。

---

## 📖 第一部分：理論與開發環境配置 (深度解析)

### 1. 用戶端程式的運作原理
用戶端程式並不直接存取硬碟上的資料庫檔案，而是透過 **「資料庫驅動程式 (Driver)」** 發送 SQL 指令給 MySQL 伺服器，並接收回傳的結果集。



### 2. Python 開發環境 (PyMySQL)
Python 具備高可讀性，是資料分析與 AI 開發的首選。
* **安裝套件**：在終端機或 Anaconda Prompt 執行 `pip install pymysql`。
* **核心模組**：`import pymysql`。

### 3. PHP 開發環境 (XAMPP & mysqli)
PHP 是為了 Web 而生的語言，許多知名網站 (如 Facebook, WordPress) 都是以 PHP 為基礎。
* **環境建議**：使用 XAMPP 整合包，它內建了 Apache 網頁伺服器與 PHP 解譯器。
* **核心擴充**：使用 `mysqli` (MySQL Improved) 擴充功能。

---

## 📖 第二部分：SQL 實戰範例 (跨語言語法對照)

無論使用哪種語言，資料庫操作的「四部曲」都是：**建立連線 ➔ 建立游標/執行 SQL ➔ 處理結果 ➔ 關閉連線**。

### 1. Python (PyMySQL) 範例
  ```python
  import pymysql

  # 1. 建立連線
  db = pymysql.connect(host="localhost", user="root", password="123", database="school")
  
  # 2. 建立游標與執行查詢
  cursor = db.cursor()
  cursor.execute("SELECT * FROM 學生")
  
  # 3. 抓取一筆資料並顯示
  data = cursor.fetchone()
  print(f"學生姓名: {data[1]}")
  
  # 4. 關閉連線
  db.close()
  ```

### 2. PHP (mysqli) 範例
  ```php
  <?php
    // 1. 建立連線
    $link = mysqli_connect("localhost", "root", "123", "school");

    // 2. 執行查詢
    $sql = "SELECT * FROM 學生";
    $result = mysqli_query($link, $sql);

    // 3. 處理結果 (用迴圈印出所有學生)
    while ($row = mysqli_fetch_array($result)) {
        echo "學號: " . $row["學號"] . " 姓名: " . $row["姓名"] . "<br/>";
    }

    // 4. 關閉連線
    mysqli_close($link);
  ?>
  ```

---

## 🤖 第三部分：AI 輔助開發 (跨語言邏輯轉換)

在實務工作中，你可能需要將現有的 Python 邏輯改寫成網頁版的 PHP。這時 AI 能提供巨大的幫助。

> [!TIP]
> **【給 AI 的 Prompt - 協助語言邏輯轉換與安全補強】**
>
> 「你是一位全端工程師。我手邊有一段 Python 寫入 MySQL 的程式碼，請幫我將其改寫為 PHP。
> 1. **原始邏輯**：接收表單傳入的商品名稱與價格，執行 `INSERT INTO products`。
> 2. **核心要求**：請使用 PHP 的 `mysqli` 語法。
> 3. **資安補強**：請務必實作『預擬陳述 (Prepared Statements)』來防止 SQL 注入攻擊，就像 Python 的 `%s` 參數化查詢一樣。
> 4. 請加上詳細的中文註解，解釋 PHP 連線與 Python 連線的差異。」

---

## 💻 第四部分：Python (Streamlit) + PHP 綜合實作

我們將建立一個雙平台的「特產店會員管理系統」。店長用 Python (Streamlit) 處理複雜報表，顧客則透過 PHP 網頁註冊會員。

### 1. Python 端：Streamlit 數據分析 (複習與進階)

  ```python
  import streamlit as st
  import pymysql
  import pandas as pd

  # 建立通用連線函式
  def get_db_connection():
      return pymysql.connect(host="localhost", user="root", password="123", 
                             database="kinmen_shop", charset="utf8mb4")

  st.title("🖥️ 特產店管理員後台 (Python)")

  if st.button("查看會員分布"):
      conn = get_db_connection()
      # 💡 使用 pandas 一次讀取所有資料
      df = pd.read_sql("SELECT city, COUNT(*) as count FROM customers GROUP BY city", conn)
      st.bar_chart(df.set_index('city'))
      conn.close()
  ```

### 2. PHP 端：顧客註冊簡易網頁 (`register.php`)

  ```php
  <!DOCTYPE html>
  <html>
  <body>
    <h2>🛍️ 金門特產店 - 新會員註冊</h2>
    <form action="save_user.php" method="post">
      姓名: <input type="text" name="name"><br>
      城市: <input type="text" name="city"><br>
      <input type="submit" value="送出註冊">
    </form>
  </body>
  </html>
  ```

### 3. PHP 端：寫入資料庫 (`save_user.php`)

  ```php
  <?php
    $conn = mysqli_connect("localhost", "root", "123", "kinmen_shop");
    mysqli_query($conn, "SET NAMES utf8"); // 💡 設定編碼避免亂碼

    $name = $_POST["name"];
    $city = $_POST["city"];

    $sql = "INSERT INTO customers (customer_id, customer_name, city) VALUES ('C999', '$name', '$city')";
    
    if (mysqli_query($conn, $sql)) {
        echo "註冊成功！歡迎來到金門特產店。";
    } else {
        echo "註冊失敗：" . mysqli_error($conn);
    }
    mysqli_close($conn);
  ?>
  ```

---

## ❓ 第五部分：常見問題與排解 (FAQ)

1. **Q: 為什麼 PHP 網頁顯示出來的中文都是亂碼？**
   * A: 這是編碼問題！在 PHP 中連線後，務必執行 `mysqli_query($link, 'SET NAMES utf8');`。同時，確保你的 HTML 檔案儲存格式為 UTF-8。
2. **Q: Python 與 PHP 可以同時連線到同一個資料庫嗎？會不會互撞？**
   * A: 完全可以！資料庫伺服器本來就是為了「多使用者並行」而設計的。只要依照第十五章學到的交易與鎖定觀念正確操作，多平台同時存取是沒問題的。
3. **Q: 為什麼 Python 比較好寫，還需要學 PHP？**
   * A: 因為 PHP 運作在 Apache 伺服器上，不需要使用者安裝任何套件，只要有瀏覽器就能看。對於大眾化的網頁（如特產店官網），PHP 的部署難度與成本通常較低。

---

## 📝 第六部分：實戰演練與魔王挑戰

### 🟢 基礎題 (Python 腳本練習)
1. **資料筆數統計**：請撰寫一段 Python 程式碼，連線到 `kinmen_shop` 資料庫，並印出目前的商品總數（使用 `COUNT(*)` 語法）。
2. **PHP 環境測試**：請啟動 XAMPP，建立一個簡單的 `test.php` 並連線到資料庫，若連線成功則印出 "MySQL 連線成功！"。

### 🟡 進階題 (PHP 資料表顯示)
3. **動態表格產生**：請撰寫一個 PHP 網頁 `inventory.php`。該網頁在載入時，會自動到 `products` 表中撈取所有資料，並利用 HTML 的 `<table>` 標籤，將商品名稱與售價整齊地顯示在網頁上。

### 😈 魔王挑戰題 (跨平台即時同步系統)
4. **【特產店全通路管理挑戰】**
   * **情境**：顧客在 PHP 網頁端完成註冊。
   * **挑戰 A**：請修改 PHP 的寫入邏輯，使用 `mysqli` 的 Prepared Statements 確保輸入姓名時包含特殊符號（如單引號）也不會當機。
   * **挑戰 B**：在 Python (Streamlit) 管理員後台，加入一個 `st.empty()` 的容器與一個「更新數據」按鈕。當你在 PHP 註冊完新會員後，按下 Python 端按鈕，圖表必須即時反映出剛才新增的會員資料。
   * **過關標準**：成功展示資料在「PHP 寫入」與「Python 讀取」之間的無縫同步，並體會資料庫作為資料交換中心的強大能力！
