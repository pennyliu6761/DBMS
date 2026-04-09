# 實作四：關聯式資料表建置 (從理論到實務)

**授課教師：劉鎮豪**
**課程：資料庫管理系統**

在前面的實作中，我們體會了如何從關聯式資料庫中「撈取」並「組合」資料。本章節我們將角色對調，回到資料庫設計師的視角，學習如何從無到有，建置出具備嚴謹關聯性的資料表。我們將以最常見的 **「教務系統：科系與學生」** 為例。

---

## 📖 課前觀念：為什麼需要關聯 (Relationship)？

在設計資料庫時，初學者常犯的錯誤是將所有資料塞進同一張「大表」(Excel 思維)。例如，把學生的姓名、電話，以及他所屬的「科系名稱」、「系辦公室位置」、「系主任名字」全放在一起。

> [!WARNING]
> **大表設計的致命缺點 (資料重複與異常)：**
> 如果有 1000 個工管系學生，那「工業工程與管理學系」、「管理學院公文櫃」這些字眼就會在資料庫裡重複出現 1000 次！更糟的是，如果工管系換了系主任，系統必須一口氣更新 1000 筆資料，萬一漏改一筆，資料就會出現不一致的災難。

### 核心解法：資料庫正規化 (Normalization) 與鍵值 (Keys)

為了解決這個問題，我們將資料拆分成兩張表，並透過特定的「鑰匙」將它們串接起來：

1. **主鍵 (Primary Key, PK)：** 每張表都必須有一個獨一無二的識別碼。就像你的身分證字號，用來證明「這筆資料是誰」。
   - `departments` (科系表) 的 PK 是 `dept_id` (科系代號)。
   - `students` (學生表) 的 PK 是 `student_id` (學號)。
   
2. **外鍵 (Foreign Key, FK)：** 用來指向另一張表的主鍵。它就像是一個「連結標籤」，告訴系統這筆資料歸屬於哪裡。
   - 我們在 `students` 表中加入一個 `dept_id` 欄位作為 FK。當某個學生的 `dept_id` 填入 `IEM` 時，系統就知道他隸屬於資管系。

> [!IMPORTANT]
> **一對多關係 (1:N)：**
> 一個科系可以有很多個學生，但一個學生通常只隸屬於一個科系。在關聯式資料庫中，**外鍵 (FK) 永遠建立在「多 (N)」的那一方**。(把科系代號放在學生表裡)

---

## 💻 步驟一：使用 SQL 語法建立資料表 (DDL)

理解理論後，我們來實際用 SQL 的 `CREATE TABLE` 語法建立這兩張表。請打開 MySQL Workbench，開啟一個新的 SQL 查詢視窗 (Query Tab)。

> [!NOTE]
> **建表順序非常重要！** > 由於學生表 (FK) 必須參考科系表 (PK)，所以**必須先建立被參考的「科系表」，才能建立「學生表」**。

### 1. 建立科系表 (`departments`)

1. 在查詢視窗輸入以下語法：

    ```sql
    CREATE TABLE departments (
        dept_id VARCHAR(10) PRIMARY KEY,    -- 設為主鍵 (例如: 'IM', 'IE')
        dept_name VARCHAR(50) NOT NULL,     -- 科系名稱，NOT NULL 代表必填
        office_location VARCHAR(100)        -- 辦公室位置，可以不填
    );
    ```

2. 點擊閃電圖示 ⚡ 執行。

   <img width="716" height="157" alt="image" src="https://github.com/user-attachments/assets/e4b726d3-3988-41cb-be1e-b155c9978878" />

### 2. 建立學生表 (`students`) 並設定外鍵約束

1. 接著輸入以下語法建立學生表：

    ```sql
    CREATE TABLE students (
        student_id VARCHAR(10) PRIMARY KEY, -- 設為主鍵 (例如: 'S001')
        student_name VARCHAR(50) NOT NULL,  -- 學生姓名
        phone VARCHAR(20),                  -- 電話
        dept_id VARCHAR(10),                -- 準備用來當作外鍵的欄位
        
        -- 👇 核心重點：宣告外鍵約束 (Foreign Key Constraint)
        FOREIGN KEY (dept_id) REFERENCES departments(dept_id)
    );
    ```

2. 點擊閃電圖示 ⚡ 執行。

   <img width="709" height="241" alt="image" src="https://github.com/user-attachments/assets/7f5b4a4a-2f91-44e9-b7d7-0c7c9ef8ffde" />

> [!TIP]
> **什麼是外鍵約束 (Constraint)？**
> 這是一種資料庫的防呆機制。一旦設定了外鍵，如果你試圖在學生表中新增一個 `dept_id` 為 `XYZ` (不存在於科系表的代號) 的學生，資料庫會**直接報錯並拒絕新增**，確保資料的絕對完整性 (Referential Integrity)！

---

## 🖱️ 步驟二：使用 MySQL Workbench 圖形化介面快速建立

雖然會寫 SQL 語法是基本功，但在業界實務上，面對動輒幾十個欄位的資料表，工程師通常會使用 Workbench 內建的圖形化介面 (GUI) 來快速建表。

我們現在試著用 GUI 建立一張「課程表 (`courses`)」：

1. 在 MySQL Workbench 左側的 **SCHEMAS** 面板中，展開你的資料庫 (例如 `demo_db`)。
2. 對著 **Tables** 資料夾按右鍵，選擇 **Create Table**。
3. **設定基本屬性：**
    - 在中間上方的 `Table Name` 輸入 `courses`。
4. **設定欄位 (Columns)：**
    - 點擊下方區塊的 `Column Name`，開始輸入欄位：
        - 第一列輸入 `course_id`，勾選 **PK** (主鍵) 與 **NN** (必填)。Datatype 選 `VARCHAR(10)`。
        - 第二列輸入 `course_name`，勾選 **NN**。Datatype 選 `VARCHAR(50)`。
        - 第三列輸入 `credits` (學分)，Datatype 選 `INT`。
      
   <img width="507" height="230" alt="image" src="https://github.com/user-attachments/assets/b06a974d-61d7-4c3c-87a4-54965996c450" />

5. **套用變更：**
    - 設定完成後，點擊右下角的 **Apply** 按鈕。
    - Workbench 會自動幫你把剛剛點選的介面，**翻譯成對應的 SQL 語法**讓你確認！
   
   <img width="693" height="260" alt="image" src="https://github.com/user-attachments/assets/0218d433-ffa4-4b46-a8df-5666a70c6d25" />

    - 確認無誤後，再次點擊 **Apply**，接著按 **Finish**。
   
   <img width="400" height="159" alt="image" src="https://github.com/user-attachments/assets/3440794a-c8c2-4a3e-9aba-d629f5ac7ea1" />

6. 回到左側 SCHEMAS 面板，點擊重整按鈕 🔄，你就會看到 `courses` 資料表已經建好囉！

   <img width="201" height="315" alt="image" src="https://github.com/user-attachments/assets/3446e052-e462-4d45-a004-9947e2355b30" />

> [!TIP]
> **Workbench 縮寫小字典：**
> * **PK (Primary Key)**：主鍵。
> * **NN (Not Null)**：不可為空值（必填）。
> * **UQ (Unique)**：唯一值（該欄位內容不可與別人重複，例如 Email）。
> * **AI (Auto Increment)**：自動遞增（通常用於整數主鍵，由資料庫自動 +1）。

---

## 步驟三：產生實體關聯圖 (ER Diagram)

想確認我們剛剛建立的資料表之間是否有正確連線嗎？Workbench 可以幫我們一鍵產生漂亮的關聯圖！

1. 點選 Workbench 最上方選單列的 **Database** -> **Reverse Engineer...** (反向工程)。

   <img width="430" height="217" alt="image" src="https://github.com/user-attachments/assets/196d8ddb-6a35-4d8b-9d2e-3c5ba09f5827" />
   
2. 一路點擊 **Next**。

   <img width="878" height="304" alt="image" src="https://github.com/user-attachments/assets/45590f6e-a7f0-4065-876a-a56ad00c2ae3" />

3. 在 `Select Schemas` 步驟中，勾選我們剛才操作的資料庫。

   <img width="538" height="236" alt="image" src="https://github.com/user-attachments/assets/b1628b00-ebb8-4659-94f0-5f12d2614d8d" />

4. 繼續點擊 **Next** 直到 **Execute** 執行完畢，最後點擊 **Finish**。

   <img width="480" height="221" alt="image" src="https://github.com/user-attachments/assets/7c6bc4a9-929c-4760-8c6d-86631e900f3e" />

5. 畫面上將會出現一份精美的 **EER Diagram**，你可以清楚地看見表與表之間的「一對多」連線 (一條線連接著一個鑰匙與一個多重分支圖示)，這對於期末專題的架構展示非常實用！
   
   <img width="602" height="429" alt="image" src="https://github.com/user-attachments/assets/f03e7604-62ee-4d33-9b16-a69cba175521" />
