# 瀏覽器執行機制與載入效能：學習筆記

> 範圍：第一層（延續瀏覽器渲染）＋ 第二層（網路與載入）
> 主要依據：HTML Living Standard、W3C 規格、IETF RFC、MDN、web.dev、Chrome for Developers 官方文件（完整清單見文末「來源」）
> 最後整理：2026-11

---

## 目錄

- [0. 前置觀念回顧](#0-前置觀念回顧)
- **第一層：直接延續瀏覽器渲染**
  - [1. Event Loop 的完整順序](#1-event-loop-的完整順序)
  - [2. Core Web Vitals：LCP、INP、CLS](#2-core-web-vitalslcpinpcls)
  - [3. DevTools Performance 面板](#3-devtools-performance-面板)
  - [4. Observer API](#4-observer-api)
- **第二層：網路與載入**
  - [5. 關鍵渲染路徑（Critical Rendering Path）](#5-關鍵渲染路徑critical-rendering-path)
  - [6. HTTP 快取](#6-http-快取)
  - [7. 輸入網址到畫面出現的完整過程](#7-輸入網址到畫面出現的完整過程)
- [面試快速複習](#面試快速複習)
- [來源](#來源)

---

## 0. 前置觀念回顧

讀後面的章節之前，先確認這幾件事：

1. **瀏覽器渲染管線**：`Parse（DOM、CSSOM）→ Style → Layout → Paint → Composite`
2. **一個分頁只有一條主執行緒**。你的 JS、Vue 或 React，以及瀏覽器的 Style、Layout、Paint，全部在同一條主執行緒上輪流執行，一次只能做一件事。
3. **Composite（合成）在另一條執行緒上進行**，所以只改 `transform` 或 `opacity` 的動畫，在主執行緒忙碌時通常還能保持順暢。
4. **Web Worker 是另一條執行緒，不能碰 DOM**。它只能分擔計算，渲染還是主執行緒的工作。
5. **框架的「渲染」**是計算並修改 DOM，**瀏覽器的渲染**是把 DOM 畫成像素。框架修改完 DOM 之後，瀏覽器會自動進行渲染，不需要寫任何程式碼觸發。

後面所有內容都建立在第 2 點上：**主執行緒一次只能做一件事，誰佔太久，其他人就得等。**

---

# 第一層：直接延續瀏覽器渲染

## 1. Event Loop 的完整順序

### 1.1 三種排隊的東西

| 種類 | 是什麼 | 常見來源 |
|---|---|---|
| **Task**（也常被叫做 macrotask） | 一個獨立的工作單位，一次只執行一個 | `<script>` 的執行、`setTimeout`、`setInterval`、使用者事件（click、keydown）、`postMessage`、`MessageChannel` |
| **Microtask** | 目前的 task 結束後，**立刻全部**執行完 | `Promise.then/catch/finally`、`await` 之後的程式碼、`queueMicrotask()`、`MutationObserver` 的 callback |
| **Rendering（更新畫面）** | 瀏覽器判斷這一幀需要更新畫面時才會執行 | `requestAnimationFrame` callback、Style、Layout、`ResizeObserver`、`IntersectionObserver`、Paint |

### 1.2 一輪 Event Loop 的順序

HTML 規格定義的 event loop 處理模型，簡化後如下：

```
┌──────────────────────────────────────────────────────────┐
│ ① 從 task queue 取出「一個」最舊的 task 並執行              │
│                                                          │
│ ② Microtask checkpoint：把 microtask queue「全部」清空      │
│    （執行 microtask 時又產生新的 microtask，也會在這一輪執行完）│
│                                                          │
│ ③ 如果這一幀需要更新畫面（Update the rendering）：            │
│    a. 觸發 resize、scroll 事件，評估 media query            │
│    b. 執行 CSS 動畫、Web Animations                         │
│    c. 執行 requestAnimationFrame 的 callback               │
│    d. 重新計算 Style、Layout                                │
│    e. 執行 ResizeObserver 的 callback（改了尺寸會回到 d）     │
│    f. 更新 IntersectionObserver 的觀察結果                   │
│    g. Paint，把結果交給合成執行緒                             │
│                                                          │
│ ④ 如果有空閒時間：執行 requestIdleCallback                   │
└──────────────────────────────────────────────────────────┘
               ↓ 回到 ①
```

三個重點：

- **一次只取一個 task，但 microtask 一次清空。**
- **③ 不是每一輪都會執行。** 瀏覽器會依照螢幕更新率決定（60Hz 螢幕大約每 16.7ms 一次）。分頁在背景時，可能完全不更新畫面。
- **rAF 在 Style 和 Layout 之前執行**，所以在 rAF 裡修改樣式，會在同一幀生效，很適合寫動畫。

### 1.3 經典題：輸出順序

**題目一**

```js
console.log('1')
setTimeout(() => console.log('2'), 0)
Promise.resolve().then(() => console.log('3'))
queueMicrotask(() => console.log('4'))
requestAnimationFrame(() => console.log('5'))
console.log('6')
```

**答案**：`1 → 6 → 3 → 4`，接著 `2` 和 `5` 的順序**不保證**。

- `1`、`6`：同步程式碼，屬於目前這個 task。
- `3`、`4`：microtask，目前的 task 一結束就全部執行。
- `2`（task）和 `5`（rAF）：要看下一輪先遇到「取下一個 task」還是「這一幀要更新畫面」。實際測試時兩種順序都可能出現。面試時能講出「順序不保證，並說明原因」，比背一個固定答案更好。

**題目二：async/await**

```js
async function foo() {
  console.log('a1')
  await null
  console.log('a2')
}

console.log('start')
foo()
Promise.resolve().then(() => console.log('p'))
console.log('end')
```

**答案**：`start → a1 → end → a2 → p`

- 呼叫 `foo()` 時，`await` 之前的程式碼是**同步**執行的，所以 `a1` 緊接在 `start` 之後。
- `await` 之後的程式碼（`a2`）會被排進 microtask。
- `.then` 的 `p` 也排進 microtask，排在 `a2` 後面。

### 1.4 常見陷阱：Microtask 餓死（starvation）

```js
function loop() {
  Promise.resolve().then(loop)   // 每個 microtask 又產生一個新的 microtask
}
loop()
```

因為 microtask 必須清空才會進入下一步，這段程式碼會讓瀏覽器**永遠到不了渲染階段**，畫面完全凍結。`setTimeout` 寫成同樣的遞迴則不會凍結，因為每次只執行一個 task，中間瀏覽器還有機會渲染。

### 1.5 跟框架的關係

**Vue：`nextTick`**

```js
import { ref, nextTick } from 'vue'

const count = ref(0)

async function increment() {
  count.value++
  count.value++
  count.value++          // 同步改 3 次，Vue 只會修改 DOM 1 次

  await nextTick()       // 等 Vue 把 DOM 修改完
  // 這裡可以讀到更新後的 DOM（例如元素高度）
}
```

- Vue 3 把元件更新排進一個 queue，再用 Promise（也就是 **microtask**）一次執行。`nextTick()` 等的就是這次 flush。
- **重要細節**：`nextTick` 之後 **DOM 已經更新，但瀏覽器還沒有 Paint**，因為 microtask 在渲染之前執行。所以下面這種寫法**看不到 loading**：

```js
loading.value = true
await nextTick()
heavyWork()             // 同步跑 2 秒，loading 根本來不及畫出來
loading.value = false
```

  如果要先把 loading 畫出來，再開始重計算，必須讓出一幀，例如：

```js
loading.value = true
await new Promise(r => requestAnimationFrame(() => setTimeout(r, 0)))
heavyWork()
```

  更好的做法是把 `heavyWork` 丟給 Web Worker（見前一份筆記），或切成小段（見 1.6）。

**React**

- React 18 起，同一個事件或非同步流程裡的多次 `setState` 會自動合併（automatic batching）。
- `useLayoutEffect` 在 DOM 修改後、瀏覽器 Paint **之前**執行。`useEffect` 通常在 Paint **之後**執行。需要「先量尺寸再調整，避免畫面閃一下」時用 `useLayoutEffect`，其他情況用 `useEffect`。

### 1.6 讓出主執行緒（yield）

**Long task（長任務）**：W3C Long Tasks API 和 web.dev 都把**超過 50ms** 的 task 定義為 long task。超過 50ms 的部分稱為 blocking time（阻塞時間）。

要避免 long task，可以把工作切成小段，每段之間讓出主執行緒，讓瀏覽器有機會處理使用者輸入和渲染：

```js
function yieldToMain() {
  // scheduler.yield() 支援度：Chrome / Edge 129+、Firefox 142+，Safari 尚未支援
  if (globalThis.scheduler?.yield) {
    return scheduler.yield()
  }
  return new Promise(resolve => setTimeout(resolve, 0))
}

async function processAll(items) {
  for (let i = 0; i < items.length; i++) {
    process(items[i])
    if (i % 100 === 0) await yieldToMain()   // 每 100 筆讓出一次
  }
}
```

| 方法 | 特性 |
|---|---|
| `setTimeout(fn, 0)` | 支援度最好。缺點是接續的工作會排到 task queue 的**最後面**，可能被其他 task 插隊 |
| `scheduler.yield()` | 專門用來讓出主執行緒。接續的工作會**優先**於其他一般 task 執行 |
| `requestIdleCallback` | 只在主執行緒空閒時執行，優先權最低。主執行緒一直很忙的話，可能**永遠不會執行**。適合送分析資料這類不急的工作 |
| Web Worker | 唯一真正把工作**移出**主執行緒的方法。上面三種都還是在主執行緒上執行，只是切小段 |

---

## 2. Core Web Vitals：LCP、INP、CLS

### 2.1 是什麼

Google 定義的三個「真實使用者體驗」指標，各自衡量一個面向：

| 指標 | 衡量什麼 | 良好 | 需要改善 | 不佳 |
|---|---|---|---|---|
| **LCP**（Largest Contentful Paint） | 載入速度：畫面中最大的內容多久出現 | ≤ 2.5 秒 | 2.5–4 秒 | > 4 秒 |
| **INP**（Interaction to Next Paint） | 互動反應速度：使用者操作後，畫面多久有反應 | ≤ 200ms | 200–500ms | > 500ms |
| **CLS**（Cumulative Layout Shift） | 視覺穩定性：畫面有沒有突然跳動 | ≤ 0.1 | 0.1–0.25 | > 0.25 |

- **評估方式**：取所有頁面瀏覽的**第 75 百分位數**。也就是說，至少 75% 的使用者達到「良好」，整個頁面才算良好。
- **INP 在 2024 年 3 月 12 日取代 FID** 成為正式指標。FID 只量「第一次」互動的延遲，INP 量整個頁面生命週期中的所有互動，FID 已被淘汰。

### 2.2 LCP

**定義**：從使用者開始導覽，到可視範圍內最大的圖片、文字區塊或影片渲染完成的時間。

**常見的 LCP 元素**：首頁大圖（hero image）、Banner、文章標題、商品主圖。

**優化方向**：

```html
<!-- 1. LCP 圖片提高下載優先權 -->
<img src="/hero.webp" fetchpriority="high" width="1200" height="600" alt="...">

<!-- 2. LCP 圖片「不要」加 loading="lazy"（常見錯誤） -->

<!-- 3. 如果 LCP 圖片寫在 CSS 裡（background-image），用 preload 提早下載 -->
<link rel="preload" as="image" href="/hero.webp">
```

- 減少伺服器回應時間（TTFB），例如使用 CDN。
- **純 CSR 的 SPA 對 LCP 不利**：HTML 一開始只有 `<div id="app"></div>`，要等 JS 下載、執行完才會出現內容。SSR 或 SSG（Vue 的 Nuxt）可以讓 HTML 一開始就帶有內容。

### 2.3 INP

**定義**：觀察頁面上所有的點擊、觸控、鍵盤互動，回報延遲**最嚴重**的那一次（每 50 次互動會忽略一次最高值，排除偶發的極端值）。捲動和滑鼠移入（hover）**不算**。

**一次互動的延遲由三段組成**：

```
使用者點擊
   │
   ├─ ① Input delay（輸入延遲）
   │     主執行緒正在忙別的事（例如一個 long task），事件 callback 還沒開始執行
   │
   ├─ ② Processing duration（處理時間）
   │     你的事件 callback 執行的時間（包含 Vue / React 的更新）
   │
   └─ ③ Presentation delay（呈現延遲）
         Style → Layout → Paint，到下一幀真正畫出來
   ▼
畫面出現回饋
```

這就是第 0 章「主執行緒一次只能做一件事」的直接後果：

| 哪一段太長 | 常見原因 | 解法 |
|---|---|---|
| ① Input delay | 使用者點擊時，主執行緒剛好在跑 long task（例如第三方 script、大量初始化） | 切小段（1.6）、移到 Worker、延後執行不急的程式碼 |
| ② Processing | 事件 callback 做太多事 | 先更新畫面給回饋，其他工作 yield 之後再做 |
| ③ Presentation | DOM 太大、Layout 太貴、layout thrashing | 減少 DOM 數量（虛擬捲動）、避免強制同步 layout（見 3.3） |

```js
// 好的做法：先給視覺回饋，再做其他事
async function onSubmit() {
  showSpinner()                // 使用者馬上看到反應
  await yieldToMain()          // 讓瀏覽器先把 spinner 畫出來
  validateForm()
  await saveToServer()
  sendAnalytics()              // 不急的工作放最後
}
```

### 2.4 CLS

**定義**：頁面上**非預期**的版面位移累積分數。

- 每次位移的分數 = **影響範圍比例 × 移動距離比例**。
- 位移會被分組成「session window」：位移之間間隔小於 1 秒的算同一組，每組最長 5 秒。CLS 取**分數最高的那一組**。
- 使用者互動後 500ms 內發生的位移**不算**（例如點擊展開選單，這是預期中的變化）。

**常見原因與解法**：

| 原因 | 解法 |
|---|---|
| 圖片、影片沒有指定尺寸，載入後把下面的內容推開 | 加上 `width`、`height` 屬性，或 CSS `aspect-ratio` |
| 廣告、嵌入內容、API 資料晚到才插入 | 預先保留空間（`min-height`、骨架畫面） |
| 網頁字型載入後替換，字寬改變 | `font-display`、`size-adjust`，或預先載入字型 |
| 用 `top`、`margin` 做動畫 | 改用 `transform`（不會造成 layout 位移，也比較便宜） |

### 2.5 怎麼量

| 類型 | 工具 | 說明 |
|---|---|---|
| **Field data**（真實使用者） | Chrome UX Report（CrUX）、PageSpeed Insights、Search Console、`web-vitals` 套件 | Google 評分以這個為準 |
| **Lab data**（實驗室） | Lighthouse、DevTools Performance 面板 | 用來除錯和重現問題 |

```js
// 在自己的專案收集真實使用者數據
import { onLCP, onINP, onCLS } from 'web-vitals'

onLCP(console.log)
onINP(console.log)
onCLS(console.log)
```

注意：Lighthouse 沒有真人操作，**量不到真正的 INP**，只能用 TBT（Total Blocking Time）當參考。

---

## 3. DevTools Performance 面板

### 3.1 錄製步驟

1. 用**無痕視窗**開啟頁面，避免擴充功能干擾測量結果。
2. 開啟 DevTools → **Performance** 分頁。
3. 在設定中開啟 **CPU throttling（例如 4x slowdown）**，模擬手機等較慢的裝置。開發用的電腦太快，很多問題在本機看不出來。
4. 按下 **Record**，操作你要分析的流程（例如點擊篩選按鈕），再按 **Stop**。
5. 分析載入效能時，改按 **Start profiling and reload page**。

新版 Performance 面板一打開就會即時顯示本機的 **LCP、CLS**，你和頁面互動後也會顯示 **INP**，還能設定讀取 CrUX 的真實使用者數據來比較。

### 3.2 怎麼看結果

```
┌ 時間軸（上方）──────────────────────────────────────────┐
│ CPU 使用量圖：顏色越滿代表主執行緒越忙                     │
│ Frames：每一幀花多久                                     │
├ Main（主執行緒的火焰圖）─────────────────────────────────┤
│ ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓ Task  ◢ ← 右上角紅色三角形 = long task│
│   ▓▓▓▓▓ Event: click                                    │
│     ▓▓▓ 你的函式                                          │
│       ▓ Recalculate Style / Layout                       │
├ 下方分頁 ──────────────────────────────────────────────┤
│ Summary / Bottom-up / Call tree / Event log              │
└─────────────────────────────────────────────────────────┘
```

- **X 軸是時間，Y 軸是呼叫堆疊**。越寬的長條，代表執行越久。
- **顏色大致對應**：黃色 = JS 執行（Scripting）、紫色 = Style 和 Layout（Rendering）、綠色 = Paint。
- **Bottom-up 分頁**：依照「自身耗時」排序，最快找到最花時間的函式。
- 看到 **Forced reflow** 的警告，就是 3.3 說的 layout thrashing。

### 3.3 實戰：Layout Thrashing（強制同步 layout）

```js
// ❌ 錯誤：每一圈都「寫入 → 讀取」，迫使瀏覽器每圈都重算一次 layout
const boxes = document.querySelectorAll('.box')
boxes.forEach(box => {
  box.style.width = box.parentElement.offsetWidth / 2 + 'px'   // 寫完馬上又讀
})

// ✅ 正確：先全部讀完，再全部寫入，只需要重算一次 layout
const widths = [...boxes].map(box => box.parentElement.offsetWidth)  // 全部讀
boxes.forEach((box, i) => {
  box.style.width = widths[i] / 2 + 'px'                            // 全部寫
})
```

**會強制觸發 layout 的常見讀取**：`offsetWidth/Height/Top/Left`、`clientWidth/Height`、`scrollTop/Height`、`getBoundingClientRect()`、`getComputedStyle()`。

### 3.4 練習建議

拿你手邊的 Vue 專案，挑一個「感覺有點卡」的操作（大表格篩選、切換分頁），開 4x CPU throttling 錄一次，回答這三個問題：

1. 有沒有紅色三角形（long task）？最長的是多少 ms？
2. 那個 long task 裡面，黃色（JS）和紫色（Layout）各佔多少？
3. 用 Bottom-up 找出最花時間的函式，它是你寫的程式碼，還是框架或第三方套件？

---

## 4. Observer API

### 4.1 為什麼需要它們

以前要知道「元素有沒有出現在畫面上」，常見的寫法是監聽 `scroll` 事件，在 callback 裡呼叫 `getBoundingClientRect()`。問題是：

- `scroll` 事件觸發非常頻繁。
- `getBoundingClientRect()` 會強制觸發 layout（見 3.3）。
- 兩個加起來，捲動時主執行緒會一直被佔住，畫面變得不順。

Observer API 改由**瀏覽器在適當的時機（大多是渲染流程中）主動通知你**，不用自己一直去問。

### 4.2 四種常用的 Observer

| Observer | 觀察什麼 | 常見用途 | callback 時機 |
|---|---|---|---|
| `IntersectionObserver` | 元素和可視範圍（或某個祖先元素）的交集變化 | 圖片 lazy load、無限捲動、曝光追蹤、進入畫面才播放動畫 | 在渲染流程中（Layout 之後）計算交集，callback 再以 task 的形式執行 |
| `ResizeObserver` | 元素尺寸變化 | 元件依自身寬度調整版面、圖表自動縮放 | 渲染流程中，Layout 之後 |
| `MutationObserver` | DOM 結構、屬性、文字變化 | 監聽第三方套件插入的 DOM | microtask |
| `PerformanceObserver` | 效能數據（long task、LCP、layout shift 等） | 效能監控、自行收集指標 | 非同步 |

### 4.3 IntersectionObserver：無限捲動（Vue 3）

```vue
<template>
  <ul>
    <li v-for="item in list" :key="item.id">{{ item.name }}</li>
  </ul>
  <div ref="sentinel">載入中...</div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue'

const list = ref([])
const sentinel = ref(null)
let observer

onMounted(() => {
  observer = new IntersectionObserver(
    (entries) => {
      if (entries[0].isIntersecting) loadMore()
    },
    { rootMargin: '200px' }   // 距離畫面底部還有 200px 就提早載入
  )
  observer.observe(sentinel.value)
})

onBeforeUnmount(() => observer.disconnect())   // 一定要記得取消，避免記憶體洩漏
</script>
```

補充：單純的圖片 lazy load，直接用原生的 `<img loading="lazy">` 就好，不用自己寫 Observer。但**首屏的 LCP 圖片不要加**（見 2.2）。

### 4.4 ResizeObserver：常見錯誤訊息

```
ResizeObserver loop completed with undelivered notifications.
```

原因：在 ResizeObserver 的 callback 裡又改變了被觀察元素的尺寸，導致同一幀內不斷觸發。瀏覽器為了避免無窮迴圈，會把剩下的通知延到下一幀，並印出這個訊息。解法是避免在 callback 裡修改被觀察元素本身的尺寸，或把修改延到下一幀（`requestAnimationFrame`）。

### 4.5 Vue 生態系的現成工具

VueUse 提供包裝好的版本，會自動處理元件卸載時的 `disconnect`：
`useIntersectionObserver`、`useResizeObserver`、`useMutationObserver`、`useElementVisibility`。

---

# 第二層：網路與載入

## 5. 關鍵渲染路徑（Critical Rendering Path）

### 5.1 定義

瀏覽器從收到 HTML 到**第一次把畫面畫出來**之前，必須完成的步驟和必須等待的資源。這條路徑越短，使用者越快看到東西。

**第一次渲染必須等的資源（critical resources）**：

- HTML（至少前面一部分）
- `<head>` 裡會擋住渲染的 CSS
- `<head>` 裡會擋住解析的同步 JS

### 5.2 CSS 會擋住渲染（render-blocking）

- 預設情況下，瀏覽器要等 CSSOM 建立完成才會渲染。否則畫面會先出現沒有樣式的內容，再閃一下套上樣式（FOUC）。
- 用 `media` 屬性可以讓不符合條件的 CSS 不擋渲染（但瀏覽器**還是會下載**）：

```html
<link rel="stylesheet" href="main.css">                      <!-- 擋渲染 -->
<link rel="stylesheet" href="print.css" media="print">       <!-- 不擋渲染 -->
<link rel="stylesheet" href="wide.css" media="(min-width: 1200px)">  <!-- 螢幕夠寬時才擋 -->
```

- 進階做法：把首屏需要的 CSS（critical CSS）直接 inline 在 `<head>`，其他 CSS 非同步載入。

### 5.3 JS 會擋住解析（parser-blocking）：async、defer、module

```
一般 <script>
HTML 解析 ████████▌          ▐████████
JS 下載             ▓▓▓▓▓▓
JS 執行                   ▒▒
                    └ 解析停住，等下載完、執行完 ┘

<script async>
HTML 解析 ██████████████▌  ▐█████
JS 下載     ▓▓▓▓▓▓▓▓▓▓
JS 執行                 ▒▒      ← 下載完就立刻執行（會短暫打斷解析），順序不保證

<script defer>
HTML 解析 ████████████████████████
JS 下載     ▓▓▓▓▓▓▓▓▓▓
JS 執行                           ▒▒  ← 等 HTML 解析完才執行，依照出現順序
```

| 寫法 | 下載時會擋解析嗎 | 何時執行 | 多支 script 的執行順序 | 適用情境 |
|---|---|---|---|---|
| `<script>` | 會 | 下載完立刻執行 | 依照出現順序 | 幾乎不建議放在 `<head>` |
| `<script async>` | 不會 | 下載完立刻執行 | **不保證** | 獨立的第三方程式（例如分析工具） |
| `<script defer>` | 不會 | HTML 解析完、`DOMContentLoaded` 之前 | 依照出現順序 | 自己的應用程式碼 |
| `<script type="module">` | 不會 | 預設就是 defer 的行為 | 依照出現順序 | 現代的 ES module |

補充：

- 同時寫 `async` 和 `defer` 時，以 `async` 為準。
- `async` 和 `defer` 對 inline script（沒有 `src`）沒有效果。
- **Vite 打包出來的入口就是 `<script type="module">`**，所以 Vue + Vite 專案的主程式預設已經不會擋住 HTML 解析。

### 5.4 Resource Hints：提早告訴瀏覽器

| 寫法 | 作用 | 優先權 | 使用時機 |
|---|---|---|---|
| `<link rel="preload" href="..." as="...">` | **這個頁面**馬上會用到，現在就下載 | 高 | LCP 圖片、首屏字型、藏在 CSS 裡才會被發現的資源。**一定要加 `as`**，否則可能重複下載 |
| `<link rel="modulepreload" href="...">` | 預先下載並解析 ES module | 高 | Vite 會自動幫你加 |
| `<link rel="prefetch" href="...">` | **下一個頁面**可能會用到，有空再下載 | 低 | 使用者很可能點擊的下一頁 |
| `<link rel="preconnect" href="https://...">` | 提早完成 DNS + TCP + TLS 連線 | — | 一定會用到的第三方網域（CDN、API）。只開最重要的幾個，連線本身也有成本 |
| `<link rel="dns-prefetch" href="https://...">` | 只提早做 DNS 查詢 | — | 可能會用到的網域，成本比 preconnect 低 |

```html
<head>
  <link rel="preconnect" href="https://cdn.example.com" crossorigin>
  <link rel="preload" href="/fonts/NotoSansTC.woff2" as="font" type="font/woff2" crossorigin>
  <link rel="preload" href="/hero.webp" as="image" fetchpriority="high">
  <link rel="prefetch" href="/checkout">
</head>
```

- `preload` 是「現在就要」，`prefetch` 是「之後可能要」，兩個不要搞混。用 `preload` 載入其實沒用到的資源，反而會搶走真正重要資源的頻寬。

---

## 6. HTTP 快取

### 6.1 兩個階段：強快取與協商快取

```
瀏覽器要載入 /app.js
        │
        ▼
快取裡有嗎？ ── 沒有 ──▶ 向伺服器下載（200）
        │
       有
        ▼
還「新鮮」嗎？（依 Cache-Control: max-age 判斷）
        │
   ┌────┴────┐
  新鮮       過期
   │          │
   ▼          ▼
直接用快取    帶著驗證資訊問伺服器：「我這份還能用嗎？」
（完全不發請求） If-None-Match: "abc123"（對應 ETag）
 ← 強快取     If-Modified-Since: ...（對應 Last-Modified）
                  │
            ┌─────┴─────┐
         沒變          有變
            │            │
            ▼            ▼
     304 Not Modified   200 + 新檔案
     （不用重新下載內容）
            ← 協商快取（重新驗證）
```

### 6.2 Cache-Control 常用指令

| 指令 | 意思 | 常見誤解 |
|---|---|---|
| `max-age=秒數` | 在這段時間內是新鮮的，可以直接用 | |
| `no-cache` | **可以存**，但每次使用前都要先向伺服器驗證 | ❌ 不是「不要快取」 |
| `no-store` | **完全不要存**，每次都重新下載 | 這個才是真正的「不快取」，用在敏感資料 |
| `private` | 只有瀏覽器可以存，CDN 等中間快取不行 | 用在含有個人資料的回應 |
| `public` | 任何快取都可以存 | |
| `immutable` | 內容永遠不會變，新鮮期間內連驗證都不用 | 搭配檔名有 hash 的檔案 |
| `s-maxage=秒數` | 只對 CDN 這類共用快取生效，覆蓋 `max-age` | |
| `stale-while-revalidate=秒數` | 過期後的這段時間內，先回傳舊版本，同時在背景更新 | |

**沒有設定 `Cache-Control` 不代表不會被快取。** 瀏覽器會自己猜一個快取時間（RFC 9111 稱為 heuristic freshness，常見做法是用「距離 Last-Modified 時間的 10%」）。所以一定要明確設定。

### 6.3 Vue + Vite 專案的標準設定

Vite 打包後的檔案結構：

```
dist/
├─ index.html                 ← 檔名固定，內容會隨每次部署改變
└─ assets/
   ├─ index-B7x9kQ2a.js       ← 檔名帶有內容 hash，內容變了檔名就會變
   └─ index-D4mN8pLz.css
```

| 檔案 | 建議設定 | 原因 |
|---|---|---|
| `/assets/*`（有 hash） | `Cache-Control: public, max-age=31536000, immutable` | 內容變了檔名就會變，舊檔名可以永久快取 |
| `index.html` | `Cache-Control: no-cache` | 每次都要確認有沒有新版本，否則使用者會一直載入舊的 `index.html`，引用舊的 JS 檔 |

這就是「**部署新版之後，使用者還看到舊畫面**」最常見的原因：`index.html` 被強快取了。

### 6.4 快取的層級

瀏覽器找資源時，大致依照下面的順序：

```
Memory Cache（記憶體，分頁關掉就沒了）
   ↓ 沒有
Service Worker（如果有註冊，它可以攔截請求，自己決定回什麼）
   ↓ 沒有攔截或 SW 選擇往下送
HTTP Cache（磁碟快取，依照 Cache-Control 規則）
   ↓ 沒有或過期
CDN（共用快取）
   ↓ 沒有或過期
原始伺服器
```

DevTools Network 面板的 Size 欄位會顯示 `(memory cache)`、`(disk cache)`、`(ServiceWorker)`，可以看出資源是從哪一層拿到的。

---

## 7. 輸入網址到畫面出現的完整過程

這一題把網路和前一份筆記的渲染管線串成一條線。

```
1. 處理網址
   ├ 判斷你輸入的是網址還是搜尋關鍵字
   └ 檢查 HSTS（這個網域是否強制使用 HTTPS）

2. 檢查快取（第 6 章）
   └ Service Worker → HTTP Cache，有可用的快取就直接跳到第 7 步

3. DNS 查詢：網域名稱 → IP 位址
   └ 依序查瀏覽器快取 → 作業系統快取 → DNS 伺服器
     同一個頁面裡，每個不同的網域都要查一次

4. 建立 TCP 連線：三次握手（SYN → SYN-ACK → ACK）

5. TLS 握手（HTTPS）
   └ 交換金鑰、驗證憑證。TLS 1.3 只需要 1 次往返（TLS 1.2 需要 2 次）

6. 送出 HTTP 請求，等待伺服器回應
   └ 可能先由 CDN 的邊緣節點回應

7. 收到 HTML，開始解析（第 5 章）
   ├ 建立 DOM；preload scanner 提早找出後面要下載的資源
   ├ 下載 CSS → 建立 CSSOM
   └ 下載並執行 JS

8. 渲染（前一份筆記）
   └ Style → Layout → Paint → Composite → 畫面出現

9. 框架啟動（SPA）
   └ Vue/React 執行 → 建立元件 → 修改 DOM → 再次渲染 → 頁面可以互動
```

### 7.1 每個階段的優化手段

| 階段 | 優化手段 |
|---|---|
| DNS | `dns-prefetch`、減少使用的網域數量 |
| TCP + TLS | `preconnect`、HTTP/2 連線重複使用、TLS 1.3、HTTP/3 |
| 等待伺服器 | CDN、伺服器端快取、SSR 串流 |
| 下載 | 壓縮（gzip / Brotli）、圖片轉 WebP / AVIF、code splitting |
| 解析與執行 | `defer` / `module`、減少 JS 大小、critical CSS |
| 渲染 | 減少 DOM 數量、避免 layout thrashing |

### 7.2 HTTP/1.1、HTTP/2、HTTP/3

| | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---|---|---|---|
| 傳輸層 | TCP | TCP | QUIC（建立在 UDP 上） |
| 同一條連線同時傳多個請求 | 不行，瀏覽器只好對同一網域開多條連線 | 可以（multiplexing） | 可以 |
| 隊頭阻塞（head-of-line blocking） | 有 | HTTP 層解決了，但 TCP 掉一個封包，整條連線還是要等 | QUIC 讓每個串流獨立，一個串流掉封包不影響其他串流 |

過去 HTTP/1.1 時代的「合併檔案、雪碧圖、網域分散」等優化技巧，在 HTTP/2 之後大多不再必要，有些甚至有反效果。

### 7.3 用 DevTools 看到每個階段

Network 面板點一個請求 → **Timing** 分頁，會看到：

| 欄位 | 對應階段 |
|---|---|
| Queueing / Stalled | 在瀏覽器裡排隊等待 |
| DNS Lookup | 第 3 步 |
| Initial connection | 第 4 步 |
| SSL | 第 5 步 |
| Request sent | 送出請求 |
| Waiting for server response（TTFB） | 第 6 步，伺服器處理時間加上網路往返 |
| Content Download | 下載回應內容 |

---

## 面試快速複習

**Event Loop**

- Q：`setTimeout(fn, 0)` 和 `Promise.then(fn)` 誰先執行？為什麼？
  A：`Promise.then` 先。它是 microtask，目前的 task 一結束就會全部執行；`setTimeout` 是 task，要等下一輪。
- Q：Vue 的 `nextTick` 之後，畫面已經畫出來了嗎？
  A：沒有。DOM 已經更新，但 `nextTick` 是 microtask，在瀏覽器渲染之前執行。
- Q：什麼是 long task？怎麼處理？
  A：超過 50ms 的 task。可以切小段並讓出主執行緒（`scheduler.yield()`、`setTimeout`），或把計算移到 Web Worker。

**Core Web Vitals**

- Q：三個指標和良好的門檻？
  A：LCP ≤ 2.5 秒、INP ≤ 200ms、CLS ≤ 0.1，以第 75 百分位數評估。
- Q：INP 和 FID 差在哪？
  A：FID 只量第一次互動的輸入延遲；INP 量所有互動，而且包含輸入延遲、處理時間、呈現延遲三段，直到下一幀畫出來為止。
- Q：CLS 很高，第一步會檢查什麼？
  A：圖片和影片有沒有指定尺寸、非同步插入的內容有沒有預留空間、字型替換。

**載入**

- Q：`async` 和 `defer` 的差別？
  A：兩者都不會在下載時擋住解析。`async` 下載完立刻執行、不保證順序；`defer` 等 HTML 解析完才依序執行。
- Q：`preload` 和 `prefetch` 的差別？
  A：`preload` 是目前頁面馬上要用、高優先權；`prefetch` 是之後可能用到、低優先權。
- Q：為什麼 CSS 放 `<head>`、JS 放後面或加 `defer`？
  A：CSS 會擋渲染，要越早下載越好；同步 JS 會擋解析，延後執行才不會拖慢首屏。

**快取**

- Q：`no-cache` 和 `no-store` 差在哪？
  A：`no-cache` 可以存但每次要驗證；`no-store` 完全不存。
- Q：部署新版後使用者看到舊畫面，可能原因與解法？
  A：`index.html` 被強快取，或 Service Worker 快取了舊版本。`index.html` 設 `no-cache`，有 hash 的靜態檔設長效快取；有用 Service Worker 的話要處理更新流程。
- Q：304 是什麼？
  A：協商快取的結果。伺服器確認檔案沒變，瀏覽器繼續使用本地快取，不用重新下載內容。

---

## 來源

以下皆為規格、標準文件或瀏覽器廠商與標準組織的官方文件。

**規格與標準**

- WHATWG, HTML Living Standard — Event loop processing model：https://html.spec.whatwg.org/multipage/webappapis.html#event-loop-processing-model
- WHATWG, HTML Living Standard — The script element：https://html.spec.whatwg.org/multipage/scripting.html#the-script-element
- W3C, Long Tasks API：https://www.w3.org/TR/longtasks-1/
- IETF, RFC 9111 HTTP Caching（2022 年 6 月，取代 RFC 7234）：https://www.rfc-editor.org/rfc/rfc9111

**web.dev（Google Chrome 團隊）**

- How the Core Web Vitals metrics thresholds were defined：https://web.dev/articles/defining-core-web-vitals-thresholds
- Interaction to Next Paint (INP)：https://web.dev/articles/inp
- Interaction to Next Paint is officially a Core Web Vital：https://web.dev/blog/inp-cwv-launch
- Optimize Interaction to Next Paint：https://web.dev/articles/optimize-inp
- Optimize long tasks：https://web.dev/articles/optimize-long-tasks
- Largest Contentful Paint (LCP)：https://web.dev/articles/lcp
- Cumulative Layout Shift (CLS)：https://web.dev/articles/cls
- Render-blocking CSS：https://web.dev/articles/critical-rendering-path/render-blocking-css
- Understanding the critical path：https://web.dev/learn/performance/understanding-the-critical-path
- Resource hints：https://web.dev/learn/performance/resource-hints
- Prevent unnecessary network requests with the HTTP Cache：https://web.dev/articles/http-cache

**Chrome for Developers**

- Analyze runtime performance：https://developer.chrome.com/docs/devtools/performance
- Performance panel overview：https://developer.chrome.com/docs/devtools/performance/overview
- Use scheduler.yield() to break up long tasks：https://developer.chrome.com/blog/use-scheduler-yield

**MDN Web Docs**

- In depth: Microtasks and the JavaScript runtime environment：https://developer.mozilla.org/en-US/docs/Web/API/HTML_DOM_API/Microtask_guide/In_depth
- `<script>`: The Script element：https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/script
- Critical rendering path：https://developer.mozilla.org/en-US/docs/Web/Performance/Guides/Critical_rendering_path
- Populating the page: how browsers work：https://developer.mozilla.org/en-US/docs/Web/Performance/Guides/How_browsers_work
- Intersection Observer API：https://developer.mozilla.org/en-US/docs/Web/API/Intersection_Observer_API
- ResizeObserver：https://developer.mozilla.org/en-US/docs/Web/API/ResizeObserver
- MutationObserver：https://developer.mozilla.org/en-US/docs/Web/API/MutationObserver
- Scheduler: yield() method：https://developer.mozilla.org/en-US/docs/Web/API/Scheduler/yield

**框架官方文件**

- Vue.js — Global API: nextTick()：https://vuejs.org/api/general#nexttick
- React — useLayoutEffect：https://react.dev/reference/react/useLayoutEffect

**說明**

- 1.2 的「Update the rendering」步驟是依照 HTML 規格簡化整理，省略了部分細節（例如 fullscreen、focus 相關步驟），實際步驟以規格為準。
- `scheduler.yield()` 的支援度依據 Chrome for Developers 文章（2025 年 3 月）與 MDN 相容性表，瀏覽器支援度可能持續變動。
- 1.5 中 Vue 3 用 Promise（microtask）排程更新，是依據 Vue 3 原始碼的實作行為；官方文件只描述為「等待下一次 DOM 更新 flush」。
