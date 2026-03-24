# 第七章：SQL 內建函數、群組查詢與資料視覺化

**授課教師：劉鎮豪**
**課程：資料庫管理系統**
**學校：國立金門大學**

上一章我們學會了如何針對單一條件過濾資料。但在實務的商業環境中，老闆更常問的問題是：「這個月總共賣了多少錢？」、「哪個類別的庫存最少？」。
本章節我們將學習強大的 **SQL 內建函數** 與 **GROUP BY 群組查詢**，並結合 Python (Streamlit)，打造出能自動繪製統計圖表的商業智慧 (BI) 儀表板！

---

## 🛠️ 課前準備：擴充「訂單資料表」(Workbench 操作)

為了練習日期函數與銷售統計，我們需要先在 `kinmen_shop` 資料庫中新增一張「訂單表 (`orders`)」。請在 MySQL Workbench 執行以下語法：

```sql
USE kinmen_shop;

-- 建立訂單表 (包含外鍵指向商品表)
CREATE TABLE orders (
    order_id VARCHAR(10) PRIMARY KEY,
    product_id VARCHAR(10),
    order_date DATE,
    quantity INT,
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);

-- 匯入近期金門觀光季的模擬銷售資料
INSERT INTO orders VALUES 
('ORD001', 'P001', '2026-03-01', 5),
('ORD002', 'P002', '2026-03-05', 20),
('ORD003', 'P001', '2026-03-12', 2),
('ORD004', 'P004', '2026-03-15', 10),
('ORD005', 'P002', '2026-04-02', 15),
('ORD006', 'P006', '2026-04-10', 1),
('ORD007', 'P003', '2026-04-18', 30);
```

---

## 📖 第一部分：理論與 SQL 實戰 (請在 Workbench 測試)

SQL 提供了豐富的內建函數，幫助我們在撈取資料的當下就完成運算與格式化。

### 1. 單列函數 (字串、數值、日期)

這類函數會針對每一筆資料單獨進行處理。

```sql
-- 【字串函數】CONCAT: 將商品類別與名稱合併顯示
SELECT CONCAT('【', category, '】', product_name) AS '商品全名' FROM products;

-- 【數值函數】ROUND: 舉辦 9 折促銷，並將價格四捨五入到整數
SELECT product_name, price, ROUND(price * 0.9) AS '促銷價' FROM products;

-- 【日期函數】YEAR, MONTH: 萃取訂單的年份與月份
SELECT order_id, YEAR(order_date) AS '銷售年', MONTH(order_date) AS '銷售月' FROM orders;
```

### 2. 聚合函數 (Aggregate Functions)

聚合函數能將「多筆資料」壓縮成「一個統計值」，例如總和 (SUM)、平均 (AVG)、最大值 (MAX)、筆數 (COUNT) 等。

```sql
-- 統計店內目前共有幾種商品？(COUNT)
SELECT COUNT(*) AS '總商品數' FROM products;

-- 統計所有「伴手禮」的總庫存量？(SUM)
SELECT SUM(stock) AS '伴手禮總庫存' FROM products WHERE category = '伴手禮';
```

### 3. 群組查詢 (GROUP BY) 與過濾 (HAVING)

如果我們想知道「"每個" 類別的總庫存是多少？」，我們不能只寫 `SUM()`，必須搭配 `GROUP BY` 將資料分門別類。

```sql
-- 依照「商品類別」分組，計算各類別的總庫存
SELECT category AS '商品類別', SUM(stock) AS '類別總庫存' 
FROM products 
GROUP BY category;
```

> [!WARNING]
> **HAVING vs WHERE：**
> 如果要在分組後進一步過濾 (例如：只顯示總庫存低於 200 的類別)，**不可以**使用 `WHERE`。必須使用 `HAVING` 子句，因為 `HAVING` 是專門用來過濾「聚合結果」的。

```sql
-- 找出總庫存量「小於 200」的危險庫存類別
SELECT category, SUM(stock) AS total_stock 
FROM products 
GROUP BY category 
HAVING SUM(stock) < 200;
```

---

## 🤖 第二部分：AI 輔助開發 (生成商業儀表板)

現在，我們要利用剛學會的群組統計，請 AI 幫我們設計一個精美的商業儀表板 (Dashboard)。請嘗試將以下 Prompt 貼給 AI 助理：

> [!TIP]
> **請將以下指令複製貼給 AI 助理：**
>
> 「你是一位資深的資料視覺化工程師。我正在為『金門特產進銷存系統』開發一個商業儀表板。
> 1. 請用 HTML, CSS 與 JavaScript (並引入 Chart.js 視覺化套件) 寫一個單一網頁。
> 2. 網頁上方要有幾個統計卡片 (Cards)，例如『總商品數』、『本月總銷售額』。
> 3. 網頁中間需要一張『長條圖 (Bar Chart)』，用來展示『各類別商品的總庫存量』(X軸為類別，Y軸為數量)。
> 4. 請隨機生成一些合理的假資料來渲染這個圖表，並將版面設計成深色模式 (Dark Mode) 的科技感風格。」

透過這個練習，學生將了解到後端 SQL 算出的 `GROUP BY` 資料，最終是如何變成老闆眼中精美的視覺化圖表！

---

## 💻 第三部分：Python (Streamlit) 實作資料視覺化

Streamlit 最強大的功能之一，就是它內建了圖表繪製功能！我們將把 SQL 的 `GROUP BY` 查詢結果，直接轉化為網頁上的互動式長條圖。

請在 Spyder 開啟 `app.py`，並將以下程式碼覆蓋原本的內容。

> [!IMPORTANT]
> **執行提醒：** 存檔後，請在 Anaconda Prompt 執行 `streamlit run app.py`。

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
        database="kinmen_shop", 
        charset="utf8mb4"
    )

st.title("📊 金門特產銷售與庫存儀表板")
st.markdown("結合 SQL `GROUP BY` 函數與 Streamlit 內建圖表進行資料視覺化。")
st.divider()

# --- 區塊一：各類別庫存統計 (長條圖) ---
st.header("1. 各類別庫存統計分析")

if st.button("產生庫存統計圖表"):
    try:
        conn = get_connection()
        cursor = conn.cursor()
        
        # 使用 GROUP BY 統計各類別總庫存
        query = """
        SELECT category AS '商品類別', SUM(stock) AS '總庫存量' 
        FROM products 
        GROUP BY category
        """
        cursor.execute(query)
        result = cursor.fetchall()
        
        if result:
            # 轉換為 DataFrame
            df = pd.DataFrame(result, columns=['商品類別', '總庫存量'])
            
            # Streamlit 畫圖需要設定 Index (X軸)
            df.set_index('商品類別', inplace=True)
            
            # 💡 魔法指令：直接將 DataFrame 畫成長條圖！
            st.bar_chart(df)
            
            # 同時在下方顯示原始數據表
            st.caption("詳細數據資料：")
            st.dataframe(df, use_container_width=True)
        else:
            st.info("目前沒有資料可供統計。")
            
    except Exception as e:
        st.error(f"❌ 查詢失敗：{e}")
    finally:
        if 'cursor' in locals(): cursor.close()
        if 'conn' in locals(): conn.close()

st.divider()

# --- 區塊二：每月訂單數量走勢 (折線圖) ---
st.header("2. 每月訂單數量走勢")

if st.button("產生銷售走勢圖"):
    try:
        conn = get_connection()
        cursor = conn.cursor()
        
        # 結合 MONTH() 日期函數與 GROUP BY 進行分月統計
        query = """
        SELECT MONTH(order_date) AS '月份', COUNT(order_id) AS '訂單筆數'
        FROM orders
        GROUP BY MONTH(order_date)
        ORDER BY MONTH(order_date) ASC
        """
        cursor.execute(query)
        result = cursor.fetchall()
        
        if result:
            # 將月份加上 '月' 字，讓圖表 X 軸更好看
            columns = ['月份', '訂單筆數']
            df = pd.DataFrame(result, columns=columns)
            df['月份'] = df['月份'].astype(str) + '月' 
            df.set_index('月份', inplace=True)
            
            # 💡 使用折線圖展示時間趨勢
            st.line_chart(df)
        else:
            st.warning("目前沒有訂單資料。")
            
    except Exception as e:
        st.error(f"❌ 查詢失敗：{e}")
    finally:
        if 'cursor' in locals(): cursor.close()
        if 'conn' in locals(): conn.close()
```

---

## 📝 課堂隨堂練習 (Exercises)

請同學根據 `kinmen_shop` 資料庫中的 `products` 與 `orders` 表，在 **MySQL Workbench** 中挑戰以下統計問題：

1. **基本聚合：** 請問店內目前**最貴**的商品價格是多少？(提示：使用 `MAX()`)
2. **條件聚合：** 請問「肉品」類別的商品，平均價格是多少？(提示：使用 `AVG()` 搭配 `WHERE`)
3. **分組統計：** 請列出每一個月份 (使用 `MONTH()`) 的「總銷售數量 (quantity 總和)」。
4. **進階過濾 (HAVING)：** 承上題，請找出總銷售數量 **「大於 10 件」** 的月份。

> [!NOTE]
> 練習題的解答將於下週課程開頭公布，請同學務必親自在 Workbench 上敲擊語法測試！
