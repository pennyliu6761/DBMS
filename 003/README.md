# 實作三：多表關聯查詢 (JOIN) 與進階 UI 互動

**授課教師：劉鎮豪**
**課程：資料庫管理系統**

在前一個實作中，我們學會了如何查詢單一資料表。但「關聯式資料庫 (Relational Database)」最強大的地方，在於將不同資料表串接起來。
本章節將以 Sakila 資料庫為例，帶領同學實作一個進階功能：**「透過下拉選單選擇一位演員，並找出他演過的所有電影」**。

---

## 📖 課前觀念與 SQL 語法解析

在實作之前，我們必須先理解「為什麼資料要分開存？」以及「如何把它們拼回來？」。

### 1. 資料庫的靈魂：正規化 (Normalization) 與多對多關係
在 Sakila 影音資料庫中，你會發現演員和電影的資料並沒有全部塞在同一張表裡，而是被拆成了三張表：
- `actor` (演員表)：只存演員基本資料（Primary Key: `actor_id`）。
- `film` (電影表)：只存電影基本資料（Primary Key: `film_id`）。
- `film_actor` (橋接表)：只存哪位演員演了哪部電影。

> [!IMPORTANT]
> **為什麼不直接在 `film` 表裡面加一個「演員」欄位就好？**
> 實務上， **一部電影有很多演員，一個演員也演過很多電影** ，這在資料庫中稱為 **「多對多 (N:M) 關係」** 。
> 如果硬塞在同一格欄位（例如：`PENELOPE GUINESS, NICK WAHLBERG...`），未來我們將完全無法進行搜尋或統計。因此，建立一張 `film_actor` 作為「橋接表 (Junction Table)」來記錄對應關係，是關聯式資料庫最標準的設計模式。

### 2. 跨表合併：INNER JOIN 語法剖析
要還原演員與電影的關係，我們必須使用 `JOIN` 指令，靠著 Primary Key (主鍵) 與 Foreign Key (外鍵) 將這三張表「縫合」起來。

- **核心 SQL 語法**：
    ```sql
    SELECT f.title, f.release_year 
    FROM film f
    JOIN film_actor fa ON f.film_id = fa.film_id
    JOIN actor a ON fa.actor_id = a.actor_id
    WHERE CONCAT(a.first_name, ' ', a.last_name) = 'PENELOPE GUINESS';
    ```

- **語法詳細解析**：
    1. **`FROM film f`**：我們把 `film` 資料表取個簡短的別名叫做 `f`，方便後續寫程式。
    2. **`JOIN ... ON ...`**：告訴資料庫「我要把哪兩張表接起來，而且是用哪兩個欄位來核對身分」。例如 `f.film_id = fa.film_id` 就是讓電影表和橋接表的 ID 對齊。
    3. **`CONCAT()` 函式**：由於 `actor` 表將名字分成了 `first_name` 與 `last_name`，我們使用 `CONCAT` 將它們中間加一個空白串接起來，方便與前端網頁傳來的「全名」進行比對。

### 3. 🛡️ 進階資安與防呆設計：前端限制取代後端驗證
在實作二中，我們學到文字輸入框 (`st.text_input`) 有被 **SQL Injection（隱碼攻擊）** 的風險，所以我們必須使用 `%s` 參數化查詢。

而在這個實作中，我們將展現另一種更高級的防護與防呆技巧：**動態下拉選單 (`st.selectbox`)**。
與其讓使用者「自己打」演員名字（可能會打錯字、或者惡意輸入 SQL 語法），不如我們**先從資料庫 `SELECT` 出所有合法的演員名單，變成下拉選單讓使用者「只能用選的」**。這不僅提升了使用者體驗 (UX)，也從前端直接阻絕了惡意輸入的可能性！

---

## 步驟一：撰寫 Streamlit 多表查詢程式碼

理解了底層邏輯後，我們來更新 Python 程式碼。請將以下區塊接續貼在您 `app.py` 檔案的最下方。

1. 打開 Spyder 中的 `app.py`。
2. 複製並貼上以下程式碼：

    ```python
    st.divider() # 畫一條分隔線

    # ==========================================
    # 區塊三：演員電影作品查詢 (多表 JOIN 實作)
    # ==========================================
    st.header("3. 演員電影作品查詢 (JOIN)")

    # 步驟 A：從資料庫撈出所有「合法的」演員全單，作為防呆下拉選單的資料來源
    try:
        conn = get_connection()
        cursor = conn.cursor()
        
        # 使用 CONCAT 將 First Name 與 Last Name 合併成全名
        cursor.execute("SELECT CONCAT(first_name, ' ', last_name) FROM actor")
        
        # 將撈出來的二維結果轉換成一維的 Python List: ['PENELOPE GUINESS', 'NICK WAHLBERG', ...]
        actor_names = [row[0] for row in cursor.fetchall()]
        
    except Exception as e:
        st.error(f"讀取演員清單失敗：{e}")
        actor_names = [] # 若失敗則給予空清單，避免後方程式當掉
    finally:
        if 'cursor' in locals(): cursor.close()
        if 'conn' in locals(): conn.close()

    # 步驟 B：建立下拉選單，將剛剛撈出的 actor_names 放入
    if actor_names:
        selected_actor = st.selectbox("請選擇一位演員：", actor_names)

        # 步驟 C：當使用者按下按鈕時，執行三表 JOIN 查詢
        if st.button("查詢他的電影作品"):
            try:
                conn = get_connection()
                cursor = conn.cursor()
                
                # 撰寫多表 JOIN 的 SQL 語法
                # 💡 即使下拉選單已經很安全，我們依然保持好習慣，使用 %s 參數化查詢
                query = """
                SELECT f.title AS '電影名稱', f.release_year AS '發行年份', f.length AS '片長(分鐘)'
                FROM film f
                JOIN film_actor fa ON f.film_id = fa.film_id
                JOIN actor a ON fa.actor_id = a.actor_id
                WHERE CONCAT(a.first_name, ' ', a.last_name) = %s;
                """
                
                # 執行語法，將下拉選單選中的名字 (selected_actor) 傳入 %s
                cursor.execute(query, (selected_actor,))
                result = cursor.fetchall()
                
                if result:
                    # 取得欄位名稱 ('電影名稱', '發行年份', '片長(分鐘)')
                    columns = [col[0] for col in cursor.description]
                    df = pd.DataFrame(result, columns=columns)
                    
                    st.success(f"✅ 找到了 {len(df)} 部由 {selected_actor} 演出的電影！")
                    st.dataframe(df, use_container_width=True)
                else:
                    st.info("這位演員目前沒有電影紀錄。")
                    
            except Exception as e:
                st.error(f"❌ 查詢失敗：{e}")
            finally:
                if 'cursor' in locals(): cursor.close()
                if 'conn' in locals(): conn.close()
    ```

3. **記得存檔！**

---

## 步驟二：重新整理網頁並測試

> [!NOTE]
> Streamlit 具有「熱重載 (Hot-Reload)」的特性。當您修改並儲存 `app.py` 後，不需要在終端機重新啟動伺服器！

1. 回到您剛剛開啟的網頁瀏覽器 (`http://localhost:8501`)。
2. 在網頁右上角點擊 **"Always rerun"**，或者直接按下鍵盤的 `F5` 重新整理網頁。
3. 您會看到網頁最下方出現了「3. 演員電影作品查詢 (JOIN)」的新區塊。
4. **測試任務：**
    - 點開下拉選單，確認裡面是否成功載入了來自 MySQL `actor` 表的演員名單。
    - 選擇任意一位演員（例如：`NICK WAHLBERG`），點擊按鈕，觀察系統是否能透過 JOIN 語法，跨越三張資料表抓出他演過的所有電影。
      
    <img width=50% height=auto alt="image" src="https://github.com/user-attachments/assets/c66cc1a3-59e4-42e7-b9aa-feed70dd416d" />
