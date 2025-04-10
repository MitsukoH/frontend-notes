# GitHub Copilot 在 WSL 無法使用的除錯紀錄（EAI_AGAIN 問題）

🗓 日期：2025/04/10  
🧠 狀況描述：  
在 Windows + WSL (Ubuntu 22.04) 的 VSCode 開發環境下，GitHub Copilot 無法使用。嘗試執行 `GitHub Copilot: Show Status` 指令無反應，插件 UI 顯示 `FetchError: getaddrinfo EAI_AGAIN api.github.com` 錯誤。

---

## 🐞 問題現象

- Copilot 插件已安裝，Chat 也正常
- 但無法登入、無法顯示狀態、無法自動補全
- 嘗試重新安裝無效
- 確認後發現 **只要在 WSL 中打開 VSCode 就會失敗**

---

## 🧪 錯誤訊息解釋

```
FetchError: getaddrinfo EAI_AGAIN api.github.com
```

這是 DNS 查詢錯誤，代表 **WSL 無法解析 GitHub API 的網域名稱**，與 Copilot 本身無關。

---

## 🛠️ 解法流程

### 1. 編輯 `/etc/wsl.conf` 禁止自動覆寫 DNS 設定

```bash
sudo nano /etc/wsl.conf
```

加入以下內容：

```ini
[network]
generateResolvConf = false
```

### 2. 手動指定 DNS

```bash
sudo nano /etc/resolv.conf
```

改為：

```
nameserver 8.8.8.8
nameserver 1.1.1.1
```

### 3. 鎖定該檔案（避免被 WSL 自動覆寫）

```bash
sudo chattr +i /etc/resolv.conf
```

### 4. 重啟整個 WSL

```powershell
wsl --shutdown
```

然後重新打開 VSCode，使用 `code .` 啟動 WSL 專案。

---

## ✅ 驗證是否成功

- `ping api.github.com` 有回應
- `GitHub Copilot: Show Status` 可顯示
- 開啟 `.ts/.vue` 檔案後，補全功能恢復正常 ✨

---

## 🧡 心得小記

今天原本只是想寫個功能，結果在 Copilot 前卡了 4 個小時，最後才發現是 WSL 的 DNS 問題。

但也因此學會了：

- 如何修改 WSL DNS
- 如何阻止自動覆蓋 resolv.conf
- Copilot 插件的安裝與偵錯流程
- `settings.json` 高階設定方式

> 有些坑踩下去是會痛的，  
> 但感謝這些坑才能發展成樹 🌱

---

_記錄 by 嘎嘎 ✍️_