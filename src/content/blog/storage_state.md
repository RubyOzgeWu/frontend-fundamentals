---
title: "前端工程師的知識地圖: 了解瀏覽器的運作-記憶策略 #5"
description: "Storage 筆記"
pubDate: "Sep 01 2026"
order: 5
heroImage: ""
---
| 主題    | 學習重點                                                                    | 目標                                                   | 驗收標準                                                                                                      |
| ------- | --------------------------------------------------------------------------- | ------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------- |
| Storage | localStorage、sessionStorage、IndexedDB、Cache Storage                      | 能根據資料生命週期、容量、同步/非同步需求選擇 storage  | 給你 token、UI preference、大型離線資料、暫存表單等情境，能選出合理 storage 並說明原因                        |
| Cookie  | Cookie lifecycle、Domain、Path、Expires/Max-Age、Secure、HttpOnly、SameSite | 理解 cookie 在 authentication、session、安全性上的角色 | 能解釋 cookie vs localStorage；知道 HttpOnly、Secure、SameSite 解決什麼問題；理解 request 為何會自動帶 cookie |

## Browser storage & Application State

```text
Server / API
 ↓ fetch
Application State (Pinia / reactive / memory)
 ↕ selectively persist
Browser Storage
 ├─ localStorage
 ├─ sessionStorage
 ├─ IndexedDB
 └─ Cache Storage
HTTP State Mechanism
 └─ Cookie → browser may attach to matching requests automatically
```

### Browser Storage + Cookie

| 機制           | 最適合                                                | 生命週期 / scope                            | API 性質                                     | 是否自動送 server          | 實務上通常如何應用                                                                                       |
| -------------- | ----------------------------------------------------- | ------------------------------------------- | -------------------------------------------- | -------------------------- | -------------------------------------------------------------------------------------------------------- |
| localStorage   | 少量、非敏感、跨重啟 preference                       | per-origin；通常持續到清除 / eviction       | 同步；string key/value                       | 否                         | 儲存主題模式、語言偏好、sidebar 展開狀態、使用者 UI 設定；不建議存敏感 token                             |
| sessionStorage | 單一 tab 的短期狀態                                   | per-origin + per-tab；tab session 結束消失  | 同步；string key/value                       | 否                         | 多步驟表單暫存、單一分頁搜尋條件、結帳流程暫存、tab-specific 狀態                                        |
| IndexedDB      | 大量 structured data、offline app data                | per-origin；best-effort / 可請求 persistent | 非同步；transactional                        | 否                         | 離線資料、草稿、大量 JSON、圖片 / Blob、同步佇列、PWA 本地資料庫                                         |
| Cache Storage  | HTTP Request/Response 快取、PWA assets / API response | per-origin；受 quota / eviction             | 非同步；Promise                              | 否                         | Service Worker 快取資源 HTML、JS、CSS、圖片、API response，實作 offline-first / stale-while-revalidate   |
| Cookie         | 小型 server/client shared HTTP state、session id      | 由 cookie attributes 控制                   | HTTP mechanism；document.cookie 本身較偏同步 | 是，符合 scope / policy 時 | Authentication session、refresh token、session id、CSRF 相關 cookie；通常搭配 HttpOnly、Secure、SameSite |

- `localStorage` → 使用者偏好資料、UI
- `sessionStorage` → 單一 Tab 暫存
- `IndexedDB` → 大量結構化資料、離線資料
- `Cache Storage` → HTTP 資源快取
- `Cookie` → Authentication / Session

---

### LocalStorage & SessionStorage

- 以 `String` 方式儲存
- **環境隔離:** 不同的 scheme + host + port → 不同的 storage
- **同步 API:** 大量讀寫會阻塞 main thread

#### LocalStorage:

- **XSS 攻擊:**
  - 透過寫入 JavaScript 取得儲存的敏感資訊，並以使用者的身分惡意操作
  - **防禦方法:** 搭配 **refreshToken** 過期更換 accessToken

#### **SessionStorage:**

- **Tab 隔離:** 不同的 tab (瀏覽器分頁) → 不同的 sessionStorage

---

### IndexedDB

- 儲存大量的 **structured data** (例如: objects, blob) → **離線資料、歷史紀錄、草稿**
- **非同步 API:** 不會阻塞 main thread
- JavaScript-based object-oriented database

#### **Transaction**

- **同一批資料一起寫入資料庫** → 成功一起寫入、其中有一個失敗就失敗

```javascript
const tx = db.transaction(["users", "orders"], "readwrite");

tx.objectStore("users").put(user);
tx.objectStore("orders").put(order);
```

#### **Migration**

- 修改資料庫後，連同升級 IndexedDB → 一樣需要 **Migration thinking**

---

### Cache Storage

- 專門存取 **HTTP request → response** 的資料
- 和 **Service Worker** 搭配 → **PWA 離線存取**的重要技術

```text
Browser
  │
  │ GET /logo.svg
  ↓
Service Worker
  │
  ├─ 查 Cache Storage
  │
  ├─ 有 → 直接回傳
  │
  └─ 沒有 → Network fetch
  │
  ↓
Browser 收到 Response
```

#### 常見策略

1. **Cache-first** → 先找 cache，沒有才 fetch 資料
2. **Network-first** → 先 fetch 資料，沒有才找 cache
3. **Stale-While-Revalidation** → 讀取 cache 的同時背景 fetch 資料，並更新 cache

---

### Cookie

- **HTTP state mechanism**

```text
Server 用 Set-Cookie 寫入
        ↓
Browser 儲存
        ↓
之後符合條件的 request
(Browser 檢查 Cookie)
        ↓
  Domain / Host 符合？
  Path 符合？
  Secure 符合？ (HTTPS)
  HttpOnly? (禁止 JS 使用 document.cookie 方式取得，降低 XSS 攻擊)
  SameSite 允許？ (跨網頁限制: strict, lax, none)
  Cookie 還有效？
        ↓
Browser 自動帶 Cookie (header)
        ↓
Server 用 cookie 裡的 session id 找登入狀態
```

- 適合用來儲存使用者登入的權限資料

#### CSRF 攻擊

- 攻擊者借用使用者已登入的身分 (cookie 拿到 id)，借用使用者的瀏覽器觸發需要權限的 request
- **防禦方法:** server 再多驗證一個 **CSRF Token**

---
