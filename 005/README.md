# 實作五：資料庫正規化 (1NF, 2NF, 3NF) 理論與 Workbench 實戰

**授課教師：劉鎮豪**
**課程：資料庫管理系統**

在現實世界的企業中，我們拿到的大量原始資料通常是透過 Excel 紀錄的，這些資料充滿了重複、空白與格式混亂。這個章節，我們將在 MySQL Workbench 中模擬這個過程：從一張慘不忍睹的「原始資料表」開始，一步步遵循正規化理論 (Normalization)，將它改造成完美的關聯式架構。

---

## 🛑 步驟零：產生原始未正規化資料 (0NF - 異常的起源)

假設系辦公室的工讀生，用 Excel 記錄了學生的選課與成績資料，並直接匯入到資料庫中。

1. 請在 Workbench 開啟一個新的 SQL 查詢視窗。
2. 執行以下語法，建立一張名為 `raw_student_courses` 的原始大表，並塞入資料：

    ```sql
    CREATE DATABASE IF NOT EXISTS school_demo;
    USE school_demo;

    -- 建立 0NF 原始大表
    CREATE TABLE raw_student_courses (
        student_id VARCHAR(10),
        student_name VARCHAR(50),
        department VARCHAR(50),
        dept_head VARCHAR(50),
        courses_taken VARCHAR(255) -- 這裡塞了全部的選課資料！
    );

    -- 塞入充滿缺陷的原始資料
    INSERT INTO raw_student_courses VALUES
    ('S001', '王小明', '工業工程學系', '劉鎮豪', 'CS101-程式設計(90分), DB201-資料庫管理(85分)'),
    ('S002', '李大華', '工業工程學系', '劉鎮豪', 'CS101-程式設計(70分)');
    ```

    > [!WARNING]
    > **為什麼這被稱為 0NF (未正規化)？**
    > 觀察 `courses_taken` 這一欄，裡面居然用逗號塞了兩門課，連課程名稱和成績都混在一起！這違反了關聯式資料庫最基本的規則：**欄位不可分割 (Atomic)**。在這種狀態下，你完全無法用 SQL 語法算出「程式設計的平均分數」或是「誰被當掉」。

---

## 🛠️ 步驟一：第一正規形 (1NF) - 消除重複群

**🎓 理論核心：** 每個欄位都必須是「單一值 (Atomic)」，不可再分割。陣列、逗號分隔的清單都必須拆成多筆獨立的資料列。

1. 我們需要建立一張符合 1NF 的新表，把那些擠在一起的課程與成績拆開成獨立的欄位。
2. 執行以下 SQL 語法：

    ```sql
    -- 建立 1NF 資料表
    CREATE TABLE table_1nf (
        student_id VARCHAR(10),
        student_name VARCHAR(50),
        department VARCHAR(50),
        dept_head VARCHAR(50),
        course_id VARCHAR(10),
        course_name VARCHAR(50),
        grade INT,
        PRIMARY KEY (student_id, course_id) -- 複合主鍵
    );

    -- 實務上通常透過 Python 等程式語言來清理 0NF 字串並寫入 1NF
    -- 這裡我們直接手動模擬拆解後的乾淨資料寫入：
    INSERT INTO table_1nf VALUES
    ('S001', '王小明', '工業工程學系', '劉鎮豪', 'CS101', '程式設計', 90),
    ('S001', '王小明', '工業工程學系', '劉鎮豪', 'DB201', '資料庫管理', 85),
    ('S002', '李大華', '工業工程學系', '劉鎮豪', 'CS101', '程式設計', 70);
    ```

    > [!TIP]
    > **1NF 達標了，但有什麼問題？**
    > 現在我們可以用 SQL 算平均分數了！但是，你可以看到王小明修了兩門課，所以他的「姓名」、「科系」和「系主任」**被迫重複寫了兩次**。這會造成嚴重的「更新異常 (Update Anomaly)」：如果系主任換人了，你必須更新好幾筆紀錄。

---

## ✂️ 步驟二：第二正規形 (2NF) - 消除部分相依

**🎓 理論核心：** 必須先符合 1NF。並且，非主鍵欄位必須「完全依賴」整個主鍵，不能只依賴「部分主鍵」。這通常發生在我們有「複合主鍵」的時候。

> [!NOTE]
> 在我們的 `table_1nf` 中，主鍵是 `(student_id, course_id)` 兩者結合。
> - `grade` (成績) 確實需要「學號 + 課程代號」才能決定，這很合理。
> - 但是！`student_name` (姓名) 只依賴 `student_id`，跟課程無關；`course_name` (課程名稱) 只依賴 `course_id`，跟學生無關。這就叫做**部分相依 (Partial Dependency)**。

1. 解法是將資料表**一分為三**：學生表、課程表、成績表 (橋接表)。
2. 執行以下 SQL 語法：

    ```sql
    -- 1. 建立學生表 (只留依賴 student_id 的欄位)
    CREATE TABLE students_2nf (
        student_id VARCHAR(10) PRIMARY KEY,
        student_name VARCHAR(50),
        department VARCHAR(50),
        dept_head VARCHAR(50)
    );
    INSERT INTO students_2nf VALUES 
    ('S001', '王小明', '工業工程學系', '劉鎮豪'),
    ('S002', '李大華', '工業工程學系', '劉鎮豪');

    -- 2. 建立課程表 (只留依賴 course_id 的欄位)
    CREATE TABLE courses_2nf (
        course_id VARCHAR(10) PRIMARY KEY,
        course_name VARCHAR(50)
    );
    INSERT INTO courses_2nf VALUES 
    ('CS101', '程式設計'), ('DB201', '資料庫管理');

    -- 3. 建立成績表 (複合主鍵，以及完全依賴複合主鍵的 grade)
    CREATE TABLE grades_2nf (
        student_id VARCHAR(10),
        course_id VARCHAR(10),
        grade INT,
        PRIMARY KEY (student_id, course_id),
        FOREIGN KEY (student_id) REFERENCES students_2nf(student_id),
        FOREIGN KEY (course_id) REFERENCES courses_2nf(course_id)
    );
    INSERT INTO grades_2nf VALUES 
    ('S001', 'CS101', 90), ('S001', 'DB201', 85), ('S002', 'CS101', 70);
    ```

---

## 💎 步驟三：第三正規形 (3NF) - 消除遞移相依

**🎓 理論核心：** 必須先符合 2NF。並且，非主鍵欄位之間「不可以互相依賴」。所有的非主鍵欄位都只能依賴主鍵。

> [!WARNING]
> **觀察 `students_2nf` 表：**
> 裡面有 `department` (系所) 和 `dept_head` (系主任)。
> 確實，知道學號就能查到系主任是誰。但是，系主任其實是依賴「系所」，而「系所」再依賴「學號」。這叫做**遞移相依 (Transitive Dependency)：學號 -> 系所 -> 系主任**。如果我們不改，工業工程系的系主任名字還是會重複出現在每個學生的資料列中！

1. 解法是把系所資料獨立成一張表。
2. 執行以下 SQL 語法，完成最後的 3NF 進化：

    ```sql
    -- 1. 建立獨立的系所表
    CREATE TABLE departments_3nf (
        department VARCHAR(50) PRIMARY KEY,
        dept_head VARCHAR(50)
    );
    INSERT INTO departments_3nf VALUES ('工業工程學系', '劉鎮豪');

    -- 2. 更新學生表，移除 dept_head，並加上外鍵指向系所表
    CREATE TABLE students_3nf (
        student_id VARCHAR(10) PRIMARY KEY,
        student_name VARCHAR(50),
        department VARCHAR(50),
        FOREIGN KEY (department) REFERENCES departments_3nf(department)
    );
    INSERT INTO students_3nf VALUES 
    ('S001', '王小明', '工業工程學系'), 
    ('S002', '李大華', '工業工程學系');
    ```

---

## 👁️ 步驟四：見證奇蹟的時刻 (Workbench 視覺化)

經歷了痛苦的正規化拆解，現在我們要享受成果了。我們將透過 Workbench 將這完美的 3NF 架構畫成圖形。

1. 在 Workbench 頂部選單選擇 **Database** -> **Reverse Engineer...**。
2. 一路點擊 **Next**，勾選我們剛才建立的 `school_demo` 資料庫。
3. 執行到 **Finish** 後，你會得到一份 EER 關聯圖。
4. 在畫布上，把我們剛才建立的 2NF 和 3NF 的表拖曳排好，你可以清楚地向同學展示：
    - 從原本臃腫的一張大表，變成了 `departments_3nf` -> `students_3nf` -> `grades_2nf` <- `courses_2nf` 的完美星型/雪花型架構！
    - 資料再也不會異常重複，且擴充性極佳！
