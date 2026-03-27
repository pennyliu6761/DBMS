# 🚀 MySQL 8.0 完整安裝與環境配置指南

![MySQL Version](https://img.shields.io/badge/MySQL-8.0-blue?style=for-the-badge&logo=mysql)
![Platform](https://img.shields.io/badge/Platform-Windows-lightgrey?style=for-the-badge&logo=windows)

這份教學將引導您完成從官網下載、產品安裝、核心參數設定，到最後的系統環境變數配置。無論您是開發初學者或是重新部署環境，都能透過本指南順利啟動資料庫服務。

---

## 📂 目錄
1. [📥 前置作業：下載安裝檔](#1-前置作業下載安裝檔)
2. [⚙️ 開始安裝流程](#2-開始安裝流程)
3. [🛠️ 核心產品配置 (Product Configuration)](#3-核心產品配置-product-configuration)
4. [🌐 系統環境變數設定 (重要)](#4-系統環境變數設定-重要)
5. [✅ 驗證安裝與登入](#5-驗證安裝與登入)

---

## 1. 📥 前置作業：下載安裝檔

在開始之前，建議先從官方渠道獲取最穩定的安裝程式。

1. **訪問官網**：前往 [MySQL Installer for Windows](https://dev.mysql.com/downloads/installer/) 下載頁面。
   > ![Image 1]
2. **選擇版本**：建議選擇檔案較大的 **"Community" 完整版** (通常為 400MB+)，這能避免安裝過程中因網路不穩導致組件下載失敗。
   > ![Image 2]
3. **跳過註冊**：在登入提醒頁面，直接點選下方的 **"No thanks, just start my download."** 即可開始下載。
   > ![Image 3]

---

## 2. ⚙️ 開始安裝流程

1. **選擇安裝類型 (Choosing a Setup Type)**：
   - 推薦選擇 **`Developer Default`**：這會自動包含 Server、Workbench (圖形介面)、Shell 等所有開發必備工具。
   > ![Image 4]
2. **檢查需求 (Check Requirements)**：
   - 系統若提示缺少 Visual C++ 庫，點擊 `Execute` 讓程式自動修復。
3. **執行安裝 (Installation)**：
   - 確認安裝清單後點擊 **Execute**，等待所有組件狀態變為「Complete」。
   > ![Image 5] | ![Image 6]

---

## 3. 🛠️ 核心產品配置 (Product Configuration)

這是安裝過程中最關鍵的部分，請務必按照以下細節設定：

### A. 網路連線 (Type and Networking)
* **Config Type**: 選擇 `Development Computer` (佔用記憶體較少)。
* **Connectivity**: 預設 **Port 3306**，若此埠號已被佔用（例如安裝了多個資料庫），請修改。
> ![Image 7] | ![Image 8]

### B. 身份驗證方式 (Authentication Method)
* **推薦選擇**：`Use Legacy Authentication Method (Retain MySQL 5.x Compatibility)`。
* **原因**：雖然新版加密更強，但許多舊版開發工具或連接程式庫（如 Node.js 舊版驅動）可能不支援新密鑰。
> ![Image 9]

### C. 帳戶權限 (Accounts and Roles)
* **MySQL Root Password**：請設定一組強大的密碼並**務必記牢**。Root 是資料庫的最高權限帳號。
> ![Image 10]

### D. Windows 服務 (Windows Service)
* 勾選 **Configure MySQL as a Windows Service**。
* 建議保持預設服務名稱 `MySQL80`，並選擇「自動啟動」，確保每次開機資料庫都會在後台運行。
> ![Image 11] | ![Image 12]

### E. 套用與測試
* 在 `Connect To Server` 步驟，輸入剛才設定的 Root 密碼並點擊 **Check**。
* 看到 **Connection succeeded** 綠色字樣後，才代表配置成功！
> ![Image 17] | ![Image 18]

---

## 4. 🌐 系統環境變數設定 (重要)

為了讓你在任何地方都能透過終端機 (CMD/PowerShell) 使用 `mysql` 指令，我們必須設定環境變數。

1. **複製路徑**：開啟檔案總管，進入 MySQL Server 的 `bin` 資料夾，預設路徑通常是：
   ```text
   C:\Program Files\MySQL\MySQL Server 8.0\bin
   ```
2. **開啟環境變數**：在 Windows 搜尋框輸入「環境變數」，點擊「編輯系統環境變數」。
3. **編輯 Path**：點擊「環境變數」→ 在下方「系統變數」找 `Path` → 點擊「編輯」→「新增」→ 貼上路徑 → 全部點擊確定儲存。

---

## 5. ✅ 驗證安裝與登入

現在，你可以嘗試使用命令列工具與資料庫互動。

1. 打開 **CMD** 或 **PowerShell**。
2. 輸入以下指令並輸入密碼：
   ```bash
   mysql -u root -p
   ```
3. 成功進入後，可以輸入簡單的 SQL 指令檢查：
   ```sql
   SELECT VERSION();
   SHOW DATABASES;
   ```

---

## 💡 維護小筆記
* **Workbench 閃退？** 可能是顯示卡驅動或 .NET Framework 問題，請嘗試重新安裝 Workbench。
* **忘記密碼？** 你需要透過 `--skip-grant-tables` 模式重新啟動服務來重設密碼。

---
*製作日期：2024年 | 整理人：[你的名字]*
