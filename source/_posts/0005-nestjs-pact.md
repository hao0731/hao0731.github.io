---
title: 在 NestJS 使用 Pact 實現契約測試
date: 2025-03-02 18:28:00
tags:
  - Backend
  - NestJS
  - Contract Testing
  - Pact
categories:
  - ['Backend', 'NestJS', 'Contract Testing']
---

在微服務架構中，各個服務由不同團隊獨立開發、部署與維運，這樣的彈性雖然增加了開發效率，但也同時導致服務間互動的不確定性。如何確保每個服務在獨立更新的同時，仍能正確溝通與協作？ **契約測試(Contract Testing)** 正是解決這一問題的有效工具。

## 什麼是 Contract Testing？

Contract Testing 是一種專注於驗證服務間 **介面(Interface)** 正確性的自動化測試方法。此方法論會將服務區分成兩個角色：

* **提供者(Provider)** ：提供 API 的服務。
* **消費者(Consumer)** ：使用 Provider 提供的 API 的服務。

Provider 與 Consumer 之間會 **提前** 約定好 Interface，包含：請求格式、回應格式、錯誤處理機制等。這種約定可以是 OpenAPI、JSON Schema 或是其他 Contract Testing 工具來作為服務間共同遵循的標準，這套約定即 **契約(Contract)** 。而 Contract Testing 有兩種方法，分別是： **消費者驅動的契約測試(Consumer-Driven Contract Testing)** 與 **提供者驅動的契約測試(Provider-Driven Contract Testing)** 。

> **注意**：Contract Testing 並 **不會也不該** 驗證 **完整的** Provider 商業邏輯，僅聚焦在 Interface 的驗證，商業邏輯的部分應屬於 **單元測試(Unit Testing)** 的範疇。

<img
  style="max-width: 500px;"
  src="contract-testing-concept.png"
  alt="Contract testing concept"
/>

### Consumer-Driven Contract Testing

這類型的 Contract Testing 是由 **Consumer 定義契約內容來確保 Provider 提供的服務是否滿足它的期望**。這樣的好處是 Provider 可以根據 Consumer 的實際需求來驗證與實作，達到快速反饋以及減少整合風險的效果，是一個適合內部團隊使用的 Contract Testing 方法。

### Provider-Driven Contract Testing

這類型的 Contract Testing 是由 **Provider 定義契約內容來統一管理 API 的文件與版本**。這樣的好處是 Provider 可以讓所有 Consumer 依據統一的標準進行開發，是一個適合用於對外公開 API 的 Contract Testing 方法。

## 為什麼需要 Contract Testing？

如文章開頭所述，微服務架構使得服務之間的互動增加了不確定性，如果沒有驗證互動正確性的方式，當服務的 Interface 頻繁發生變化時，有可能會因此導致其他服務無法正常運作，造成損失。如果導入 Contract Testing 則可以提早捕捉到不兼容的問題，避免整合時才發現有這個狀況發生。另外，在驗證服務間是否如預期運作最常見的方式即 **端對端測試(E2E Testing)** ，但測試過程可能會涉及許多複雜的商業邏輯與其他依賴的項目，針對僅需驗證 Consumer 與 Provider 之間 Interface 是否符合預期的情境，使用成本較高且流程繁複的 E2E Testing 顯得有些大材小用，使用對完整環境要求低、能夠驗證 Interface 是否符合預期的 Contract Testing 會是更好的選擇。

## Pact

Pact 是一套 **程式碼優先(code-first)** 的 Consumer-Driven Contract Testing 工具，提供多種程式語言的實作，如：[JavaScript](https://docs.pact.io/implementation_guides/javascript/readme)、[Java](https://docs.pact.io/implementation_guides/jvm)、[Golang](https://docs.pact.io/implementation_guides/go) 等。

<img
  style="max-width: 500px;"
  src="pact-logo.png"
  alt="Pact logo"
/>

[圖片來源](https://docs.pact.io/implementation_guides/javascript/readme)

### Pact 的運作流程

<img
  style="max-width: 500px;"
  src="pact-flow-concept.png"
  alt="Pact flow concept"
/>

[圖片來源](https://docs.pact.io/)

Pact 在運作流程上可以拆成兩個階段： **Consumer 階段** 與 **Provider 階段** ：

#### Consumer 階段

Consumer 在自己的測試中使用對應語言的 Pact 函式庫來定義預期的請求(HTTP Method、Path、Headers、Body 等)與回應格式(HTTP Code、Headers、Response Body 等)。測試執行期間，Pact 會啟動一個 Mock Server，Consumer 發送的請求會送到這個 Server，並會收到事前定義好的回應。測試完成後，Pact 會根據這些定義產生一個 JSON 格式的 Contract，用來記錄 Consumer 所期望的 Interface。

#### Provider 階段

Provider 要使用 Consumer 定義的 Contract 來驗證服務提供的內容是否符合期望，所以必須在測試時啟動服務，並使用對應語言的 Pact 函式庫來執行 **Pact 驗證工具(Pact Verifier)** ，會讀取 Consumer 產生的 Contract 檔案並 **重放(Replay)** Contract 內定義的請求，進而驗證服務最終回傳的內容符合 Contract 內定義的格式。如果驗證結果發現回應的格式不如預期，此 Contract Testing 就會失敗，如此一來，便可以及早發現 Interface 不符期待的問題。

## NestJS 與 Pact

Pact 有提供 [PactJS](https://docs.pact.io/implementation_guides/javascript/readme) 套件供 JavaScript、TypeScript 開發者使用。NestJS 固然可以在既有的測試流程中使用此套件來實現 Contract Testing，甚至官方還為 NestJS 實作了 [nestjs-pact](https://github.com/pact-foundation/nestjs-pact) 套件，十分貼心！

### 前置作業

假設已經有兩個基於 NestJS 實作的服務，一個是 Consumer、一個是 Provider，這些服務的專案需透過下方指令將 Pact 相關套件進行安裝：

```bash
$ npm install nestjs-pact @pact-foundation/pact -D
```

### Consumer

假設 Consumer 這個專案有一個 `TodoModule`，該 Module 內有 `TodoController` 與 `TodoService` 並匯入了 `HttpModule` 來呼叫 API。下方是 `TodoController`、`TodoService` 與 `HttpService` 之間的關係，以類別圖來呈現：

<img
  style="max-height: 400px;"
  src="consumer-class-diagram.png"
  alt="Consumer class diagram"
/>

### Provider

## 結論

