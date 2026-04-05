# 🚀 MySQL 8.0 完整安裝與環境配置指南

![MySQL Version](https://img.shields.io/badge/MySQL-8.0-blue?style=for-the-badge&logo=mysql)
![Platform](https://img.shields.io/badge/Platform-Windows-lightgrey?style=for-the-badge&logo=windows)

這份教學將引導您完成從官網下載、產品安裝、核心參數設定，到最後的系統環境變數配置。無論您是開發初學者或是重新部署環境，都能透過本指南順利啟動資料庫服務。

---

## 1. 📥 前置作業：下載安裝檔

在開始之前，建議先從官方渠道獲取最穩定的安裝程式。

1. **訪問官網**：前往 [MySQL Installer for Windows](https://dev.mysql.com/downloads/installer/) 下載頁面。
   > <img width="1108" height="954" alt="image" src="https://github.com/user-attachments/assets/fa56f8f9-bb4c-41f0-ac21-f28975f42bd4" />
   > <img width="1028" height="691" alt="image" src="https://github.com/user-attachments/assets/bedebed1-ce0a-4e75-ae06-a07196eaf147" />

2. **選擇版本**：建議選擇檔案較大的 **"Community" 完整版** (通常為 500MB+)，這能避免安裝過程中因網路不穩導致組件下載失敗。
   > <img width="865" height="604" alt="image" src="https://github.com/user-attachments/assets/48fc754e-b2e7-4f0f-969f-b9e8058a3705" />

3. **跳過註冊**：在登入提醒頁面，直接點選下方的 **"No thanks, just start my download."** 即可開始下載。
   > <img width="865" height="616" alt="image" src="https://github.com/user-attachments/assets/58bfb81b-dd99-4cc2-b2af-0af9c77330b8" />

---

## 2. ⚙️ 開始安裝流程

1. **選擇安裝類型 (Choosing a Setup Type)**：
   - 推薦選擇 **`Full`** 或 **`Developer Default`**：這會自動包含 Server、Workbench (圖形介面)、Shell 等所有開發必備工具。
   > <img width="861" height="650" alt="image" src="https://github.com/user-attachments/assets/2b961bef-fca0-4d38-8b7b-c4733f56419e" />

2. **檢查需求 (Check Requirements)**：
   - 系統若提示缺少 Visual C++ 庫，點擊 `Execute` 讓程式自動修復。
3. **執行安裝 (Installation)**：
   - 確認安裝清單後點擊 **Execute**，等待所有組件狀態變為「Complete」。
   > <img width="865" height="653" alt="image" src="https://github.com/user-attachments/assets/07b8bae7-195f-4f7b-90ec-e1a46666dde7" />
   > <img width="865" height="653" alt="image" src="https://github.com/user-attachments/assets/b9ec52f6-3396-4673-aa66-1bf4b5fa8ac0" />

---

## 3. 🛠️ 核心產品配置 (Product Configuration)

這是安裝過程中最關鍵的部分，請務必按照以下細節設定：

### A. 網路連線 (Type and Networking)
* **Config Type**: 選擇 `Development Computer` (佔用記憶體較少)。
* **Connectivity**: 預設 **Port 3306**，若此埠號已被佔用（例如安裝了多個資料庫），請修改。
> <img width="865" height="653" alt="image" src="https://github.com/user-attachments/assets/88020579-a075-4e91-a651-df50115147b4" />
> <img width="865" height="653" alt="image" src="https://github.com/user-attachments/assets/0a0608f1-91aa-4c4d-b01f-b125c6ac471a" />

### B. 身份驗證方式 (Authentication Method)
* **推薦選擇**：`Use Legacy Authentication Method (Retain MySQL 5.x Compatibility)`。
* **原因**：雖然新版加密更強，但許多舊版開發工具或連接程式庫（如 Node.js 舊版驅動）可能不支援新密鑰。
> <img width="865" height="653" alt="image" src="https://github.com/user-attachments/assets/324a7c19-bd6a-4e1a-abbf-50970e2a2492" />

### C. 帳戶權限 (Accounts and Roles)
* **MySQL Root Password**：請設定一組強大的密碼並**務必記牢**。Root 是資料庫的最高權限帳號。
> <img width="865" height="653" alt="image" src="https://github.com/user-attachments/assets/a1846f1f-0114-45b3-86ab-e2aca283fbf5" />


### D. Windows 服務 (Windows Service)
* 勾選 **Configure MySQL as a Windows Service**。
* 建議保持預設服務名稱 `MySQL80`，並選擇「自動啟動」，確保每次開機資料庫都會在後台運行。
> <img width="865" height="653" alt="image" src="https://github.com/user-attachments/assets/fe8b852f-aee4-48c9-b555-7627c5c2f872" />
> <img width="865" height="653" alt="image" src="https://github.com/user-attachments/assets/2196180d-78e9-4cdc-beb3-3e60293c76c8" />

### E. 套用與測試
* 在 `Connect To Server` 步驟，輸入剛才設定的 Root 密碼並點擊 **Check**。
* 看到 **Connection succeeded** 綠色字樣後，才代表配置成功！
> <img width="865" height="653" alt="image" src="https://github.com/user-attachments/assets/c10bc5e0-602f-4469-a968-fe9ca51cb694" />
> <img width="865" height="653" alt="image" src="https://github.com/user-attachments/assets/7e28ae1c-e2aa-4563-8cd2-26f697f38c1f" />
> <img width="865" height="653" alt="image" src="https://github.com/user-attachments/assets/d671bd33-c7f4-41b5-ba25-ae4f54d97bc9" />
> <img width="865" height="653" alt="image" src="https://github.com/user-attachments/assets/8260b372-9455-4ff2-86db-e5d3dbc63dda" />
> <img width="865" height="653" alt="image" src="https://github.com/user-attachments/assets/9dc680fb-51aa-45f8-aa7b-59dd9da0cc2d" />
> <img width="865" height="653" alt="image" src="https://github.com/user-attachments/assets/4aa45a23-54f7-4d4a-8901-c7f1c64a0e14" />
> <img width="865" height="653" alt="image" src="https://github.com/user-attachments/assets/cd89252f-a104-4800-9042-c701b1ffa8a1" />
> <img width="865" height="653" alt="image" src="https://github.com/user-attachments/assets/08f2e278-682a-4e55-aa1a-b572a4cac742" />
> <img width="865" height="653" alt="image" src="https://github.com/user-attachments/assets/ec3dc977-5dd6-4b0b-a9ff-86f6dc234c5c" />
---
