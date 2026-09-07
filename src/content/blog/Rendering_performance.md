---
title: "前端工程師的知識地圖: 認識瀏覽器的運作-渲染效能優化 #5"
description: "Rendering Performance 筆記"
pubDate: "Sep 01 2026"
heroImage: ""
---

| 主題                 | 學習重點                                                                    | 目標                          | 驗收標準                                                                        |
| ------------------ | ----------------------------------------------------------------------- | --------------------------- | --------------------------------------------------------------------------- |
| Rendering Pipeline | HTML → DOM、CSS → CSSOM、Render Tree、Layout、Paint、Composite               | 理解「程式碼改動」最後如何變成螢幕畫面         | 能完整說明從 HTML/CSS 到畫面顯示流程；看到效能問題時能判斷可能卡在哪一階段                                  |
| Reflow / Repaint   | Layout trigger、Paint trigger、Composite-only properties、layout thrashing | 知道哪些 DOM/CSS 操作成本高，能避免低效能更新 | 能解釋改 width、background、transform 成本差異；知道為何 animation 常優先使用 transform/opacity |

## Rendering

```text
HTML bytes → HTML Parser → DOM
                           ↘
                            Style Calculation → Render Tree
                           ↗
CSS bytes  → CSS Parser  → CSSOM
                                      ↓
                                    Layout
                                      ↓
                                     Paint
                                      ↓
                                  Composite
                                      ↓
                                    Pixels
```

- 後續若有改動 → 從受影響的流程開始進行更新

---

## DOM, CSSOM, Render tree

### HTML → DOM

- HTML parser → DOM node tree = **文件結構與內容**
- JS 可以介入修改 DOM
  - **parser-blocking:** 解析到 `<script>` 的時候，HTML parser 會阻塞暫停，先下載並執行 JS

### CSS → CSSOM

- CSS parser → stylesheet 解析成 CSSOM = **每個元素應該長什麼樣子**
  - **render-blocking:** CSS parser 還沒解析完成時，畫面渲染會受阻塞暫停，待 CSSOM 完成才會渲染

---

## Style, Layout, Paint, Composite

```text
Style
↓
Layout
↓
Paint
↓
Composite
↓
Pixels
```

### Style Recalculation

- stylesheet 發生變動時需要重新計算
  - `class`、`attribute`、`pseudo-class`
- 大型 DOM、複雜樣式與大量受影響節點都可能讓 style work 增加。

### Layout (Reflow)

- 計算幾何資訊 (元素尺寸和位置)
  - `width` / `height` / `margin` / `padding` / `position` / `font`
- **Layout thrashing**
  - 避免在大型迴圈中: `讀 layout → 改 layout → 再讀 → 再改`
  - Best practice: `一次 Read → Write`

### Paint (Repaint)

- 將外觀轉換成繪製 (決定要畫成什麼像素內容)

```css
color
background
background-color
background-image

border-color
border-style

box-shadow
text-shadow

outline
text-decoration

visibility
```

### Composite

- 一個頁面可以有多個 composite layers
- 個別 layer rasterize → Composite 組合
- 只需要動到 Composite 就不需要動 layout
  - `transform`, `opacity`

### 改動成本差異

- deadline 導致 **dropped frames / jank** 的原因

| 改動                         | 可能重新 Style | Layout | Paint | Composite | 直覺                                     |
| -------------------------- | ---------- | ------ | ----- | --------- | -------------------------------------- |
| width: 100px → 200px       | 是          | 是      | 是     | 是         | box 變大，其他元素位置也可能跟著變。                   |
| background-color           | 是          | 通常否    | 是     | 是         | 幾何不變，但像素外觀變了。                          |
| transform: translateX(...) | 是          | 通常否    | 通常可避免 | 是         | 移動已存在的視覺 layer，而不是重新排版。                |
| opacity                    | 是          | 否      | 常可避免  | 是         | 調整 layer 透明度；是否獨立 compositing 仍依瀏覽器決策。 |

---

## Forced Synchronous Layout 與 Layout Thrashing

### Forced Synchronous Layout

- 原本: 瀏覽器會 **批次處理 layout 的更動** (累積一次處理)
- 例外: Forced Synchronous Layout

```javascript
box.style.width = "400px";  // Write layout 更動
console.log(box.offsetWidth);  // 馬上就要 Read 計算結果
```

  - **迫使瀏覽器馬上更動和計算** ⇒ 影響效能及流暢度 (主執行緒被卡住) ⇒ **dropped frames / jank**

### Layout Thrashing

- **Read → Write → Read → Write…** 反覆循環 ⇒ 導致 **Forced Synchronous Layout**
- Read → Read → Read → Write → Write → Write ⇒ 瀏覽器可以批次處理 (效能較佳)

---

## 實務應用

| 情境                            | 常見問題                         | 建議思路                                                                            |
| ----------------------------- | ---------------------------- | ------------------------------------------------------------------------------- |
| 展開 / 收合 sidebar               | 動畫 width 造成每 frame layout    | 若設計允許，考慮 transform: translate；或降低動畫範圍。                                          |
| 大型 table / list               | DOM 太大，layout / style 成本上升   | pagination、virtualization、content-visibility 等策略。                               |
| drag / resize                 | pointermove 每次讀 / 寫 geometry | requestAnimationFrame throttle；reads / writes 分批；transform 做位移。                 |
| sticky header / scroll effect | scroll handler 高頻 DOM 操作     | 避免同步 layout read / write；可用 IntersectionObserver / requestAnimationFrame，視需求使用。 |
| tooltip / popover 定位          | 先改 DOM 再量測位置                 | 明確安排 measure → mutate，減少多次 getBoundingClientRect()。                             |
| 圖片載入後跳動                       | 圖片尺寸未知，內容重新排版                | 預留 width / height 或 aspect-ratio，降低 layout shift。                               |
| loading skeleton              | 大量 shimmer / 陰影 repaint      | 控制動畫面積與 property；避免全頁高頻 paint。                                                  |
| modal transition              | top / left / width 動畫        | 通常使用 transform + opacity，並量測 layer / paint。                                     |

### 1. Sidebar

- **NO:** 不要改動 `width` (造成 layout reflow)

```css
.sidebar {
  width: 0;
  transition: width 300ms;
}

.sidebar.open {
  width: 240px;
}
```

- **YES:** 改 `transform` `translate`

```css
.sidebar {
  transform: translateX(-100%);
  transition: transform 300ms;
}

.sidebar.open {
  transform: translateX(0);
}
```

### 2. 大型 Table / List

- **NO:** 直接顯示幾千筆
- **YES:** 採取 **Pagination**

### 3. Drag / resize / Scroll

- **NO:** Layout thrashing

```javascript
box.addEventListener("pointermove", e => {
  box.style.left = `${e.clientX}px`;
});
```

- **YES: `requestAnimationFrame`** 把更新排到下一個 browser frame 前批次執行

```javascript
let latestX = 0;
let scheduled = false;

box.addEventListener("pointermove", e => {
  latestX = e.clientX;

  if (scheduled) return;

  scheduled = true;

  requestAnimationFrame(() => {
    box.style.transform = `translateX(${latestX}px)`;
    scheduled = false;
  });
});
```

### 4. Tooltip

- **NO:** Layout thrashing

```javascript
tooltip.style.display = "block";

const rect = button.getBoundingClientRect();

tooltip.style.left = `${rect.left}px`;

const tooltipRect = tooltip.getBoundingClientRect();

tooltip.style.top = `${rect.bottom - tooltipRect.height}px`;
```

- **YES:** `getBoundingClientRect` **先量測再修改**

```javascript
const buttonRect = button.getBoundingClientRect();
const tooltipRect = tooltip.getBoundingClientRect();

tooltip.style.transform =
  `translate(${buttonRect.left}px, ${buttonRect.bottom}px)`;
```

### 5. 圖片

- **NO:** 瀏覽器還沒載入前不知道寬高

```html
<img src="photo.jpg">
```

- **YES:** **預留尺寸**

```html
<img
  src="photo.jpg"
  width="800"
  height="600"
>
```

### 6. Loading Skeleton

- **NO:** 大區塊 repaint

```css
.page {
  animation: shimmer 1s infinite;
  box-shadow: 0 0 50px rgba(0,0,0,.2);
}
```

- **YES:** **小範圍 repaint**

```css
.skeleton-highlight {
  transform: translateX(-100%);
  animation: shimmer 1s infinite;
}

@keyframes shimmer {
  to {
    transform: translateX(100%);
  }
}
```

### 7. Modal

- **NO:** 不要改動 position (造成 layout reflow)

```css
.modal {
  top: -100px;
  transition: top 300ms;
}

.modal.open {
  top: 100px;
}
```

- **YES:** **改動 composite `transform`, `opacity`**

```css
.modal {
  transform: translateY(-20px);
  opacity: 0;

  transition:
    transform 300ms,
    opacity 300ms;
}

.modal.open {
  transform: translateY(0);
  opacity: 1;
}
```
---
## Vue 實務應用

| **Vue / Nuxt 寫法**                    | **為什麼有成本**                                                      | **簡單示範**                    |
| ------------------------------------ | --------------------------------------------------------------- | --------------------------- |
| `v-if` 切換大型 subtree                  | 會真的 insert / remove DOM；::大量節點可能伴隨 Style / Layout / Paint::     | 大區塊頻繁切換時成本高                 |
| `v-show`                             | DOM 不移除，只切 `display`；仍::可能造成 Layout / Paint::                   | 適合頻繁顯示/隱藏                   |
| Transition 動畫 `height`               | `height` 是 layout property，動畫每 frame 都可能重算 Layout               | 優先考慮 `transform`            |
| `watch` 後讀 `getBoundingClientRect()` | Vue patch DOM 後，如果 layout dirty，讀 geometry 可能 ::forced layout:: | 注意 measure / mutate 分離      |
| 大量 reactive updates                  | Vue 雖然會 batching，但仍可能最後 patch 很多 DOM                            | 避免不必要 reactive dependency   |
| 長列表 `v-for`                          | 建立大量 DOM，Style / Layout 成本都會上升                                  | pagination / virtualization |

### v-if / v-show

- 偶爾切換 → `v-if`
- 頻繁切換 → `v-show`

---