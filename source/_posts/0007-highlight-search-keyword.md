---
title: Highlight 表格搜尋關鍵字
date: 2025-05-19 20:40:00
tags:
  - Frontend
  - Angular
  - Web APIs
categories:
  - ['Frontend', 'Angular', 'Web APIs']
---

資料表格經常會有搜尋關鍵字的功能，當使用者在搜尋框輸入特定關鍵字時，資料表格會將符合該關鍵字的資料過濾出來，熱門的資料表格套件都有提供相關功能，像是：[DataTables](https://datatables.net/)、[Ag Grid](https://www.ag-grid.com/) 等。以下圖為例，Ag Grid 的 [Quick Filter](https://www.ag-grid.com/angular-data-grid/filter-quick/) 功能可以讓使用者針對整個資料表格進行關鍵字查詢：

<img
  style="max-width: 500px;"
  src="ag-grid-quick-filter-example.png"
  alt="Ag Grid Quick Filter Example"
/>

這個功能很方便，不過面對較複雜、欄位較多的資料表格時，就比較難一眼看出是哪個欄位有符合該關鍵字查詢。於是就出現了 **高亮(Highlight)** 關鍵字的搜尋功能，當使用者在輸入框輸入特定關鍵字時，資料表格內符合該關鍵字的文字會像使用 `Ctrl` + `F` 的搜尋方式一樣 **加上底色**，來強調文字。

## 傳統的 Highlight 解法

要做到上述 Highlight 最直覺的做法是 **在符合關鍵字的文字周圍動態插入 span 元素**，假如本來的結構如下：

```html
<div class="row">
  <div class="cell">John</div>
  <div class="cell">28</div>
</div>
```

當在輸入框搜尋關鍵字「Jo」時，會抓取所有欄位的元素，並將符合「Jo」的文字包在 `<span>` 內再以 CSS 的 `class` 來將其上底色：

```html
<div class="row">
  <div class="cell"><span class="highlight">Jo</span>hn</div>
  <div class="cell">28</div>
</div>
```

這種方式在每次更新表格時，會新增或移除大量的 DOM 元素，造成瀏覽器頻繁進行 **佈局(Layout)** 與 **繪製(Paint)** 進而影響渲染效能，在複雜、 **虛擬捲軸(Virtual Scroll)** 的情境下，更容易因此導致明顯的卡頓與資源消耗。

## Custom CSS Highlight API

Custom CSS Highlight API 可以讓開發者在不改變 DOM 結構的情況下，以 JavaScript 結合 CSS `::highlight(name)` 偽元素來樣式化文字。

## Tree Walker

## 高效能 Highlight 解法
