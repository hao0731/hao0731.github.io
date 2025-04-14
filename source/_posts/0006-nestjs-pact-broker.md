---
title: NestJS 結合 Pact Broker 完善契約測試
date: 2025-03-08 17:20:00
tags:
  - Backend
  - NestJS
  - Contract Testing
  - Pact
  - Pact Broker
categories:
  - ['Backend', 'NestJS', 'Contract Testing']
---

## 什麼是 Pact Broker？

契約測試可以用更快、更低的成本來測試服務之間的介面是否有破壞性變更，強化服務之間介面的穩固性。但此測試方式面臨了一些挑戰：

* **契約管理**：在微服務架構中，每個服務之間的契約可能數量很多且版本頻繁變動。需要一個集中的平台來儲存、追蹤這些契約檔案，並協助團隊理解每個契約的來源與歷史。
* **驗證結果管理**：當 Provider 驗證 Consumer 所提交的契約時，我們需要一個地方來儲存這些驗證結果，讓團隊成員可以清楚知道目前哪個版本的服務與哪個契約已經完成驗證、是否相容。

為了解決上述問題，Pact 團隊打造了 [Pact Broker](https://docs.pact.io/pact_broker) 這個工具。它是一個專門設計來儲存和管理 Pact 契約與驗證結果的服務。透過 Pact Broker，我們可以更有效地管理微服務之間的契約、驗證狀態、版本關係，將其融入 CI/CD 即可打造自動化、高效的契約測試流程。

> **補充**：有關於 Pact 相關的介紹可以參考[官方文件](https://docs.pact.io/)或是我之前寫的[文章](https://hao0731.github.io/2025/03/08/0005-nestjs-pact/)。

### 參與者 (Pacticipants)

**參與者 (Pacticipants)** 一詞是 Pact 與英文中的參與者 - Participants 合併後產生的單字。在 Pact Broker 中，最基本的單位就是參與契約測試的「服務」，也就是 Consumer 與 Provider，這些服務稱之為 Pacticipants。

> **補充**：根據官方的說法，Pact Broker 的作者很後悔使用 Pacticipants 這個詞 XD

### 版本 (Versioning)

在 Pact Broker 的架構下，共有三種資源擁有版本，分別是：**Consumer 應用程式的版本** 、 **Provider 應用程式的版本** 與 **契約檔案的版本** 。

#### 契約版本

每一個被發佈到 Pact Broker 的契約都會有一個版本號，這塊是由 Pact Broker 自動處理的，開發人員並不需要針對契約設定版本。

{% mermaid %}
graph TD
  C0["Consumer"]

  subgraph Pact broker
    P0["Contract (version:abc)"]
  end

  C0 --> P0
{% endmermaid %}

#### Consumer 應用程式的版本

每當一份契約被發佈到 Pact Broker 時，它會跟 **Consumer 的名稱** 、 **Consumer 應用程式的版本** 與 **Provider 的名稱** 產生關聯。其中，Consumer 的名稱與 Provider 的名稱會在撰寫契約測試時指定，版本的部分則是 Consumer 在發佈契約時指定的版本號，這個版本號必須是唯一的。這裡值得一提的是 Pact Broker 會針對 Consumer 發佈的契約進行雜湊比對，如果發佈的契約並沒有任何異動，則會將 Consumer 應用程式的版本與已經存在的契約建立關聯。

{% mermaid %}
graph TD
  subgraph Consumer
    C0["Consumer v0.0.0"]
    C1["Consumer v0.0.1"]
    C2["Consumer v0.0.2"]
  end

  subgraph Contracts
    P1["Contract A (hash: abc123)"]
    P2["Contract B (hash: def456)"]
  end

  C0 --> P1
  C1 --> P1
  C2 --> P2
{% endmermaid %}

讓多個 Consumer 應用程式版本指向同一個版本的契約不僅可以減少重複的內容，還可以避免重複驗證的情形，舉例來說，Consumer 版本為 `v0.0.0` 與 `v0.0.1` 時，並沒有改變契約的內容，那麼假設 Provider 已經針對 `v0.0.0` 發佈的契約進行驗證且通過，`v0.0.1` 也會視為驗證通過。

> **注意**：為了讓檢查重複契約的機制可以順利運作，在撰寫測試的時候，應該要 **避免隨機產生資料的行為** ，因為如果有隨機產生的資料，進行雜湊的時候一定會不同，就會導致明明沒有改變契約內容卻因隨機資料而判定為契約有異動的情況。

#### Provider 應用程式的版本

Provider 與 Consumer 一樣需要定義應用程式版本，該版本會跟 Consumer 發佈的契約產生關聯，每當 Provider 發佈新版本時，需要針對關聯的契約進行驗證，確保 Provider 的異動可以通過契約測試。

{% mermaid %}
graph TD
  subgraph Consumer
    C0["Consumer v0.0.0"]
  end

  subgraph Contracts
    CT1["Contract A (hash: abc123)"]
  end

  subgraph Provider
    P0["Provider v0.0.0"]
    P1["Provider v0.0.1"]
  end

  C0 --> CT1
  CT1 -- ❌ --- P0
  CT1 -- ✅ --- P1
{% endmermaid %}

從上方概念圖可以看出，Consumer 版本 `v0.0.0` 產生的契約在 Provider 版本 `v0.0.0` 時驗證失敗，後來 Provider 釋出 `v0.0.1` 重新進行驗證就通過了，這裡可以看出是 Provider 在 `v0.0.0` 時有問題，所以釋出 `v0.0.1` 進行修正。

#### Consumer 與 Provider 版本策略

為了發揮契約測試的最大效用，會建議不論是開發功能的 `feature/*` 分支、準備部署到 Staging 環境的 `release/*` 分支又或是正式版的 `main` 分支都執行契約測試，這樣的好處是可以確保在各個階段都能驗證介面是否符合契約內容，及早發現問題。但也代表 Consumer 與 Provider 在版本策略上需要做出改變。

在過去，版本的定義時間點可能會發生在部署到某個環境之前，這就表示開發功能的 `feature/*` 分支 **並不會有一個定義好的版本** ，那針對需要給定 Consumer 應用程式版本的 Pact Broker 來說就不符合規則，所以要改變的策略就是 **預定義版本**。根據 Pact 官方建議，可以在版本上添加 Git SHA 這類唯一且可識別版本的資訊，確保版本號一定不會有重複且能夠做到預定義版本。下圖是使用 Git Graph 繪製出的預定義版本情境，可以看到除了 `dev` 分支本身的 commit 有對應的版本外，`feature/a` 這個分支上也有定義版本：

{% mermaid %}
%%{init: { 'gitGraph': { 'mainBranchName': 'dev' } }}%%
gitGraph
  commit id: "4fc667fb" tag:"v0.0.1-4fc667fb"
  commit id: "2eed1c17" tag:"v0.0.1-2eed1c17"

  branch feature/a
  checkout feature/a
  commit
  commit id: "563c6421" tag:"v0.0.1-563c6421"
  commit
  commit
  commit
  commit id: "c517b5d3" tag:"v0.0.1-c517b5d3"

{% endmermaid %}

### 矩陣 (Matrix)

Matrix 是 Pact Broker 的核心功能，它是一張 Consumer 發佈契約與 Provider 驗證結果的記錄表，從這張表可以看出哪些 Consumer 版本發佈的契約在哪個 Provider 版本下是通過驗證的，進而得知 Consumer 版本與 Provider 版本之間的相容性。

下方是一張範例表，從該表可以看出，Banana 這個 Provider 在釋出 `1.1.0` 的時候去驗證 Apple `1.0.0` 發佈的契約，驗證結果為不通過，就表示 Banana `1.1.0` 這個版本 **不相容** 於 Apple `1.0.0`，所以後來 Banana 釋出了 `1.1.1` 進行修復，從驗證結果來看是有正確修復的，就表示 Banana `1.1.1` 相容於 Apple `1.0.0`，而最後一筆可以看出，Apple 釋出了 `1.1.0` 也相容於 Banana `1.1.1` 版本：

|Consumer|Consumer Version|Provider|Provider Version|Verification Result|
|--------|----------------|--------|----------------|-------------------|
|Apple   |1.0.0           |Banana  |1.0.0           |✅                 |
|Apple   |1.0.0           |Banana  |1.1.0           |❌                 |
|Apple   |1.0.0           |Banana  |1.1.1           |✅                 |
|Apple   |1.1.0           |Banana  |1.1.1           |✅                 |

> **補充**：Pact Broker 有提供十分強大的 Matrix UI 讓 Pacticipant 的開發者可以清楚知道上述的關係，後續會再做進一步的說明。

## 架設 Pact Broker

## Pact CLI