---
title: "前端工程師的知識地圖: 了解瀏覽器的運作-背景執行 #6"
description: "Background execution 筆記"
pubDate: "Sep 01 2026"
order: 6
heroImage: ""
---

| 主題           | 學習重點                                                              | 目標                                                      | 驗收標準                                                                                            |
| -------------- | --------------------------------------------------------------------- | --------------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| Web Worker     | Main Thread、Worker Thread、postMessage、structured clone、限制       | 能把 CPU-intensive 工作移出 UI thread                     | 能說明 Worker 適合什麼、不適合什麼；能實作主執行緒與 Worker 傳資料；知道 Worker 不能直接操作 DOM    |
| Service Worker | Proxy-like lifecycle、install/activate/fetch、Cache API、offline、PWA | 理解 Browser 如何攔截 network request、實現 cache/offline | 能說明 Service Worker 與 Web Worker 差異；能建立簡單 offline cache；知道更新與 lifecycle 的基本陷阱 |

## 概覽

分別屬於 Execution Model 中不同的 **agent** 層級:

| 機制           | 主要問題                      | 誰啟動／控制             | 是否長期存活                   | 典型用途                                   |
| -------------- | ----------------------------- | ------------------------ | ------------------------------ | ------------------------------------------ |
| Main Thread    | UI、DOM、事件、框架執行       | 頁面                     | 頁面存活期間                   | render、input、Vue / React runtime         |
| Web Worker     | CPU-heavy 工作阻塞 UI         | 頁面透過 new Worker()    | 通常依賴頁面 / worker 生命週期 | 解析、大量計算、影像／資料處理             |
| Service Worker | Network interception、offline | 透過 origin + scope 註冊 | 事件驅動，可被瀏覽器終止／喚醒 | cache、offline、PWA、push、background sync |

---

## Web Worker

- 適合用來處理 **大量計算**，避免 JS 算太久阻塞 main thread
- **Message Passing:**
  - web worker 有獨立的 **global scope 和 event loop**
  - 和 main thread 透過 message passing 資訊傳遞來互相溝通 → `postMessage()`

```javascript
worker.postMessage(data) // web worker 回傳 message 給 main thread
```

- **不可以直接操作 DOM** → 避免和 main thread 有 **race condition** 問題

| 情境                         | 適合？     | 原因                                      |
| ---------------------------- | ---------- | ----------------------------------------- |
| 大量 JSON / CSV parsing      | 適合       | CPU parsing 可能造成 long task            |
| 圖片像素處理 / 壓縮前處理    | 適合       | 大量 loop / matrix operations             |
| 排序、搜尋、統計數十萬筆資料 | 適合       | CPU-bound，可避免 UI freeze               |
| 加密、hash、大量數學運算     | 常適合     | 計算密集                                  |
| 呼叫一般 REST API            | 通常不需要 | fetch 已是非阻塞 I/O                      |
| 直接修改 DOM                 | 不適合     | Worker 無 DOM access                      |
| 非常小的計算                 | 通常不適合 | 啟動與 message transfer overhead 可能更高 |

---

### `postMessage()`

- 類似傳值: **structured clone** algorithm 複製一份 object 傳遞，不是同一個 object。
- 例外: **Transferable Object**
  - `ArrayBuffer` 傳遞很耗效能 → 直接傳遞同一個 Array **ownership** (只會存在一個 context 中)

```javascript
const buffer = new ArrayBuffer(50 * 1024 * 1024);
worker.postMessage({ buffer }, [buffer]);
```

---

### worker 的管理

| Message protocol | 多個 Worker message 怎麼清楚辨識與管理 |
| ---------------- | -------------------------------------- |
| Worker pool      | 很多工作時，怎麼避免開太多 Worker      |
| 終止與清理       | Worker 不用了怎麼釋放資源              |

#### Message protocol

- 建立統一管理及識別格式

```typescript
type WorkerRequest =
  | { requestId: string; type: "PARSE_CSV"; payload: string }
  | { requestId: string; type: "CALCULATE"; payload: number[] }
```

#### Worker pool

- 開太多 worker 會有多太多 thread → 搶 CPU 影響效能
- **queue:** 只執行固定數量的 worker，其他 worker 排隊等待

#### 終止與清理

- worker 不用了要清除，避免佔用資源

---

## Service Worker

**介於 App 和 Network 之間** → 攔截 fetch request、cache、offline、PWA

```text
App
 | fetch/navigation
 v
Service Worker <----> Cache Storage
 |
 v
Network / Server
```

### Lifecycle

#### 1. register

為這個網站註冊 service worker

```javascript
navigator.serviceWorker.register('/service-worker.js')
```

#### 2. install

瀏覽器安裝 service worker → 實務上會預先將所需的頁面資料作為快取載入 service worker

```javascript
self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open('v1').then((cache) => {
      return cache.addAll([
        '/',
        '/style.css',
        '/app.js'
      ])
    })
  )
})
```

#### 3. activate

啟用 service worker → 實務上會順便將舊的 service worker 清除

```javascript
self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then((keys) => {
      return Promise.all(
        keys
          .filter((key) => key !== 'v2')
          .map((key) => caches.delete(key))
      )
    })
  )
})
```

#### 4. control

可以開始控制頁面 → 例如攔截頁面的 fetch request

```javascript
self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request)
      .then((cached) => cached ?? fetch(event.request))
  )
})
```

---

### Cache Strategy

| 策略                   | 流程                     | 適合                                |
| ---------------------- | ------------------------ | ----------------------------------- |
| Cache First            | cache → miss 才 network | 版本化靜態資源、字型、圖片          |
| Network First          | network → fail 才 cache | 需要新鮮資料但希望 offline fallback |
| Stale-While-Revalidate | 先回 cache，同時背景更新 | 可接受短暫舊資料的內容 / API        |
| Network Only           | 永遠 network             | 高敏感、不可快取請求                |
| Cache Only             | 永遠 cache               | 完整 precache 的固定資源            |

---

## Service Worker / Web Worker

| 面向                | Web Worker             | Service Worker                                       |
| ------------------- | ---------------------- | ---------------------------------------------------- |
| 核心目的            | 平行／背景 CPU 計算    | network proxy / offline / PWA                        |
| 建立方式            | new Worker()           | navigator.serviceWorker.register()                   |
| 與頁面關係          | 通常由建立它的頁面管理 | 註冊到 origin + scope，可控制多個頁面                |
| DOM access          | 不可                   | 不可                                                 |
| 與 Main Thread 溝通 | postMessage            | postMessage / clients                                |
| Network             | 可使用 fetch           | 可攔截 fetch                                         |
| Cache Storage       | 可用，但非核心用途     | 核心常用能力                                         |
| 生命週期            | 由 worker / page 管理  | install / activate + event-driven，可被 browser 喚醒 |
| Offline             | 不是主要用途           | 主要用途之一                                         |
| CPU-heavy           | 典型用途               | 不應拿來當長時間 CPU-heavy daemon                    |

### 常見實務應用

| 情境                           | 適合方案                                      |
| ------------------------------ | --------------------------------------------- |
| 10 萬筆資料排序造成畫面 freeze | Web Worker                                    |
| API 等待 2 秒                  | 一般 async fetch；不是 Worker                 |
| 直接改 DOM animation           | Main Thread + rendering optimization          |
| 網站離線仍能用                 | Service Worker + Cache Storage                |
| 離線保存可查的表單／資料       | IndexedDB，必要時 Service Worker 協調 sync    |
| 快取 hashed JS / CSS / assets  | Service Worker caching / HTTP cache（依架構） |
| Push notification              | Service Worker                                |
| 大型 ArrayBuffer 運算          | Web Worker + transferable                     |

---

