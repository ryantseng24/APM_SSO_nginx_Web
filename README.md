# F5 APM 整合 NGINX Basic Auth 實作與測試紀錄

## 1. 文件目的與背景
本文件紀錄如何使用 F5 APM 搭配 NGINX 上的 HTTP Basic Auth 實現單一登入（SSO）功能。此流程可改善使用者體驗，避免出現瀏覽器原生的登入視窗，並統一由 APM 控制身份驗證流程。

本文件提供內部工程師在後續實作類似架構時的完整參考依據。

---

## 2. 架構圖

```
使用者 → [F5 APM 登入頁面] → APM 驗證（Local DB）
               ↓
        [注入 Authorization Header]
               ↓
          → [NGINX Basic Auth] → Web GUI
```

> APM 角色：
> - Web Form 收集帳密
> - 執行本地認證
> - 對後端 NGINX 加入 Authorization Header 完成 Basic Auth SSO

---

## 3. 環境資訊

| 項目 | 說明 |
|------|------|
| APM 版本 | BIG-IP v15.1 |
| NGINX 版本 | Open Source 1.18 |
| NGINX IP | 10.8.52.127 |
| APM VS IP | 10.8.38.101 |
| 測試帳號 | testuser / 123456 |

---

## 4. NGINX Basic Auth 設定

1. 建立 `.htpasswd` 檔案：
```bash
sudo apt install apache2-utils
htpasswd -c /etc/nginx/.htpasswd testuser
```

2. NGINX 設定（/etc/nginx/sites-enabled/default）：
```nginx
server {
    listen 80;
    server_name 10.8.52.127;

    auth_basic "Restricted Area";
    auth_basic_user_file /etc/nginx/.htpasswd;

    root /var/www/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

3. 重啟 NGINX：
```bash
sudo nginx -t && sudo systemctl reload nginx
```

---

## 5. F5 APM 設定步驟

### 5.1 Access Policy 設定
- Access Profile 名稱：`nginx_sso_profile`
- 流程：
  - Logon Page
  - LocalDB Auth
  - SSO Credential Mapping（可選）
  - Allow

### 5.2 建立 HTTP Basic SSO Profile
- 名稱：`nginx_basic_sso`
- Username Source: `session.logon.last.username`
- Password Source: `session.logon.last.password`

### 5.3 SSO Domain Mapping
- 對應 APM Access Profile → SSO/Auth Domains
- Domain 設定為：`10.8.52.127`

### 5.4 Virtual Server 設定
- 指向後端 NGINX 的 Pool
- 套用 Access Profile、HTTP Profile、SSL（如有）

---

## 6. 測試流程與驗證結果

### 6.1 測試步驟
1. 使用者開啟 APM VS URL（例如：https://10.8.38.101）
2. APM 顯示 Web Form，使用 testuser 登入
3. 登入成功後自動導向 NGINX
4. NGINX 無跳出瀏覽器 Basic Auth 視窗，直接顯示網頁

### 6.2 驗證 Log
在 NGINX `access.log` 中可看到：
```
Authorization: "Basic dGVzdHVzZXI6MTIzNDU2"
```
代表 APM 成功注入 Basic Auth Header。

---

## 7. APM Debug 設定與 Log 檢查

### 啟用 Debug Log：
```bash
tmsh modify sys db log.access.level value debug
tail -f /var/log/apm
```

### 建議在 Policy 中加入：
- Variable Assign：輸出 session.logon.last.username
- Log：寫入變數內容以方便 debug

---

## 8. 常見問題與備註

| 問題 | 原因 | 解法 |
|------|------|------|
| 沒看到 Basic Auth 視窗 | APM 攔截了後端 401 | 確認 SSO Profile 是否關閉或未綁定 |
| SSO 無作用 | Session 變數未正確對應 | 確認 Credential Mapping 正常、SSO Profile 正確引用 |
| Reload NGINX 不會登出 | Basic Auth 無狀態 | 關閉瀏覽器或清除快取才會登出 |

---

## 9. 結論與建議

- F5 APM 可順利取代瀏覽器 Basic Auth 彈窗，提供整合式登入體驗
- 適用於內部環境中使用 NGINX GUI 的應用場景
- 建議搭配 logout 機制，例如導向 `/vdesk/hangup.php3`
- 若未來使用者帳號眾多，可考慮整合 LDAP / AD

---

## 10. 附錄

### .htpasswd 加密測試
```bash
echo "testuser:`openssl passwd -apr1 123456`" > .htpasswd
```

### cURL 測試 Basic Auth
```bash
curl -u testuser:123456 http://10.8.52.127/
```

---

（完）

