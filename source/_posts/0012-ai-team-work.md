---
title: AI 時代下如何建立團隊的 AI 工作流
date: 2026-06-07 16:50:00
tags:
  - AI
  - Software Engineering
  - Harness Engineering
  - Spec-Driven Development

categories:
  - ['Software Engineering', 'AI']
---

## 前言

AI Agent 的出現正在改變軟體工程師的工作方式。過去我們習慣把大量時間花在手寫程式碼、查文件、補測試、修 lint，但現在只要提供足夠清楚的目標與上下文，AI Agent 就可以在很短的時間內產出一大段可運作的程式碼。單純比較「打字產出速度」，人類工程師幾乎不可能追上 AI Agent。

但這不代表軟體工程師的價值會消失，反而是價值重心正在轉移。LLM 本身帶有隨機性，同一個需求在不同上下文、不同提示、不同工具環境下，可能產出完全不同品質的結果。如果沒有一套可被團隊共同遵循的工作流，AI 產出的程式碼很容易變成「看起來很快，但後續很貴」的技術債。

因此，在 AI 時代下，軟體工程師的核心價值會越來越像是在設計一條讓 AI 能正確行動的軌道：如何給足正確的 context、如何把團隊規範變成 AI 能理解的知識、如何讓錯誤可以被工具即時攔下、如何讓每次產出的經驗回饋成下一次更好的流程。換句話說，未來團隊競爭的重點不只是「誰比較會 prompt」，而是「誰能建立一套可被團隊共享、維護、迭代的 AI 工作流」。

## 流程概覽

本文定義的 AI 工作流會從 PRD 開始，逐步轉換成 User Story、Design document、Implementation plan，最後再交由 AI Agent 進行實作與 Code Review。這個流程的重點不是把原本的人類流程全部丟給 AI，而是把每個階段都變成「可追蹤、可審查、可迭代」的 context 資產。

{% mermaid %}
flowchart TD
    A[PRD] --> B[PRD 轉 User Story]
    B --> C[User Story 轉 Design document]
    C --> D[產生 Implementation plan]
    D --> E[AI Agent 進行實作]
    E --> F[Code Review]
    F --> G[Merge / Release]

    S[Team Skills Library] -.-> B
    S -.-> C
    S -.-> D
    S -.-> E
    S -.-> F

    H[Harness Engineering] -.-> E
    F -. Feedback .-> S
    F -. Policy update .-> H
{% endmermaid %}

這裡最關鍵的一步是：團隊必須共同擁有、維護、迭代一套屬於團隊的 **Skills**。如果每個人都靠自己的 prompt、自己的筆記、自己的習慣來驅動 AI Agent，那團隊最後會得到很多風格不一致、品質不穩定、難以維護的產物。反過來說，如果團隊把產出 User Story、撰寫 Design document、產生 Implementation plan、發 PR、做 Code Review 等流程都沉澱成 Skills，就可以讓不同成員在不同任務上仍然用一致的方式協作。

所以，AI 工作流的核心不是「讓 AI 寫更多 code」，而是「讓 AI 在團隊定義好的工程系統裡寫 code」。接下來會先介紹導入這套流程前需要具備的知識與工具，再逐步展開每個流程階段。

## 先備知識與工具

在導入 AI 工作流前，團隊需要先準備兩類東西：一類是讓 AI Agent 知道「該怎麼做」的前饋機制，另一類是讓 AI Agent 在做錯時能被攔下並修正的反饋機制。這也是 **Harness Engineering** 的核心思想。

除此之外，團隊還需要建立 Skills 的管理方式，讓好的流程不只停留在某個人的 prompt 裡，而是可以被安裝、共享、版本控制與共同 review。最後，為了避免 AI Agent 直接從模糊需求跳到程式碼，我們會引入 **Spec-Driven Development** 的概念，讓每次實作都先有足夠清楚的規格與計畫。

### Harness Engineering

**Harness Engineering** 可以理解成替 AI Agent 建立一組工程護欄。AI Agent 很擅長根據上下文快速產出內容，但它不知道你們團隊偏好的 folder structure、命名規則、測試策略、錯誤處理方式，也不會天生知道哪些 legacy pattern 是不能再延續的。因此，我們需要把這些知識整理成 AI 可以讀取的 guide，並搭配工具檢查產出的結果。

在這個思路下，AI Agent 不再是單純靠一段 prompt 自由發揮，而是被放進一個有文件、有命令、有測試、有 review 的環境裡工作。當 guide 寫得越清楚、sensor 設得越完整，AI Agent 的產出就越接近團隊想要的方向。

#### 前饋機制 (Guides / Feedforward)

前饋機制的目標是在 AI Agent 開始實作前，先告訴它「這個專案的正確做法是什麼」。常見做法是把團隊規範整理在專案的 `docs/policies` 資料夾下，例如：

```plaintext
.
├── AGENTS.md
└── docs
    ├── policies
    │   ├── coding-style.md
    │   ├── folder-structure.md
    │   ├── testing-policy.md
    │   └── api-design.md
    ├── designs
    └── plans
```

`docs/policies` 裡面可以放專案的 Coding Style、Folder structure、API 設計原則、測試策略、錯誤處理方式等內容。這些文件不需要寫成冗長的百科全書，重點是要清楚描述「團隊希望 AI Agent 遵守哪些規則」以及「遇到常見情境時應該怎麼選擇」。

接著，在 `AGENTS.md` 中建立索引表，讓 AI Agent 一進到專案就知道要去哪裡讀取規範。例如：

```markdown
## Project Policies

- [Coding Style](docs/policies/coding-style.md)
- [Folder Structure](docs/policies/folder-structure.md)
- [Testing Policy](docs/policies/testing-policy.md)
- [API Design](docs/policies/api-design.md)

## Common Commands

- Lint: `npm run lint`
- Test: `npm test`
- Build: `npm run build`
```

這份 `AGENTS.md` 不應該塞滿所有規則，而是扮演索引與入口。真正的細節放在 `docs/policies`，讓 AI Agent 可以依任務需要逐步讀取。這樣可以避免 context 過度膨脹，也讓團隊在調整某個政策時，有明確的文件可以 review。

#### 反饋機制 (Sensors / Feedback)

反饋機制的目標是在 AI Agent 產出錯誤時，讓錯誤盡早被工具攔下，而不是等到人類 Code Review 才發現。最基本的反饋機制包含：

- **Linter / Formatter**：檢查程式碼風格是否符合團隊規範。
- **Type check**：確保型別與介面沒有被改壞。
- **Testing**：透過 unit test、integration test、contract test 等方式驗證行為。
- **Build**：確保產出的程式碼在實際建置流程中可用。

這些檢查可以綁定在 Git hook 中，例如在 pre-commit 階段執行 formatter、linter 或相關測試，讓 AI Agent 產出的程式碼在 commit 前就先經過第一層防線。若 linter 可以自動修正格式問題，就讓工具直接修正；若測試失敗，AI Agent 可以根據錯誤訊息回頭修補。

不過 Git hook 只能作為本地防線，CI 仍然是必要的第二層防線。尤其在多人協作與多服務專案中，本地環境可能不一致，CI 才是團隊共同信任的檢查結果。理想狀態下，AI Agent 實作完成後應該能主動執行 `lint`、`test`、`build`，並在 PR 描述中附上執行結果。

### Skills management

在個人使用 AI Agent 時，每個人維護自己的 prompt 或 Skills 可能還能接受；但在團隊協作中，如果每個人都維護一套自己的 Skills，很快就會遇到以下問題：

- 大家產出的 User Story、Design document、PR 描述格式都不同。
- 不同成員重複解決相同問題，卻沒有沉澱成團隊資產。
- Skills 的改善只服務個人習慣，無法以團隊協作為優先。
- 新進成員需要靠口耳相傳學會團隊的 AI 使用方式。

因此，團隊應該根據專案性質引入需要的 Skills，甚至建立一套屬於團隊自己的 Skill 庫。例如，團隊可以維護：

- `prd-to-user-story`：將 PRD 轉成團隊固定格式的 User Story。
- `user-story-to-design`：根據 User Story 產出 Design document。
- `write-implementation-plan`：根據 Design document 產生 Implementation plan。
- `create-pr`：依照團隊格式建立 PR。
- `review-ai-pr`：針對 AI Agent 產出的 PR 做初步 Code Review。

為了讓多個微服務專案共享相同的 Skills 庫，可以使用 Vercel 推出的 [`skills` CLI](https://www.skills.sh/docs/cli) 來安裝與管理 Skills。它支援從 GitHub repo、Git URL 或本地路徑安裝 Skills，也可以指定安裝到特定 AI Agent，例如 Codex、Claude Code、Cursor 等。

```bash
$ npx skills add your-org/team-agent-skills --agent codex
```

如果團隊採用 project scope 安裝，就可以讓專案擁有自己的 Skills 版本。當 CLI 版本或團隊流程會產生 project-level lock file 時，建議將 `skills-lock.json` 一起 commit 到 repo 中，用來鎖定當前 Skills 的來源、版本或 hash。如此一來，團隊就可以透過 PR 來 review Skills 的更新，避免某個人默默升級 Skill 後導致產出結果改變。

這個概念很像 `package-lock.json` 或 `go.sum`：我們不只要共享依賴，也要共享「AI Agent 如何工作的知識」。當 Skills 能被版本控制，團隊才有辦法真正做到共享、迭代與回溯。

### Spec-Driven Development

**Spec-Driven Development** 的核心思想是：在開始寫 code 前，先把需求、設計與實作計畫描述清楚。這件事在 AI 時代特別重要，因為 AI Agent 產出 code 的速度很快，如果規格模糊，它也會很快地把模糊需求擴散成大量模糊的程式碼。

傳統開發流程中，工程師可能會把很多設計思考留在腦中，直接一邊寫 code 一邊調整。但對 AI Agent 來說，腦中的上下文不存在；如果沒有明確的 User Story、Design document、Implementation plan，它很容易用自己的假設補洞。這些假設有時候合理，有時候卻會和團隊架構或產品需求衝突。

為了導入這個流程，可以使用 [superpowers](https://github.com/obra/superpowers) 這類 Skills 庫。superpowers 是一套針對 AI coding agents 設計的工程方法論與 Skills 集合，提供像是 brainstorming、writing-plans、test-driven-development、systematic-debugging、requesting-code-review 等能力。它的價值不是讓 AI 變得更會「猜」，而是讓 AI 依照更有紀律的流程工作。

在本文的流程中，我們會借用 superpowers 的思路：先透過 brainstorming 釐清需求，再把需求轉成 Design document，接著產生 Implementation plan，最後才開始實作與 review。這能讓 AI Agent 的每一步都有明確依據，也讓團隊可以在更早的階段發現問題。

## 流程介紹

### PRD 轉 User Story

團隊的 PRD 可能放在 Google Docs、Notion、Confluence，甚至公司內部文件庫中。導入 AI 工作流後，第一步不是把 PRD 複製貼上給 AI 後直接叫它寫 code，而是先透過 MCP 或 Skill 將 PRD 內容拉到 AI Agent 的工作環境中，並轉換成團隊固定格式的 User Story。

這裡可以維護一個團隊專用的 `prd-to-user-story` Skill，要求每次產出的 User Story 都必須包含固定欄位，例如：

- **Background**：為什麼要做這個需求？解決什麼問題？
- **Requirement**：使用者或系統實際需要什麼能力？
- **Acceptance Criteria (AC)**：怎樣才算完成？有哪些可驗收條件？
- **Out of Scope**：這次明確不做什麼？
- **Open Questions**：PRD 中仍不清楚或互相矛盾的地方。

在這個階段，AI Agent 很適合用來找出 PRD 裡不清楚、矛盾或缺少驗收條件的地方。例如同一份 PRD 可能同時寫了「使用者可以修改資料」與「送出後不可編輯」，這時 AI Agent 應該要把它列成 Open Question，而不是自行猜測產品想要哪一種行為。

若需求還不夠清楚，也可以透過 superpowers 的 brainstorming skill 輔助，讓 AI Agent 先提出問題、選項與取捨，再由 PM、Tech Lead 或負責該功能的工程師確認。產生好的 User Story 可以手動貼回專案管理軟體，也可以透過 Skill 或 MCP 在 Jira、Linear 等工具中建立 Ticket。

### User Story 轉 Design document

User Story 確認後，下一步是將它轉成 Design document。這時可以透過 MCP 或 Skill 從 Jira 等專案管理軟體載入 User Story，再交給 AI Agent 產出設計文件。Design document 應該統一放在專案的 `docs/designs` 資料夾下，例如：

```plaintext
docs
└── designs
    ├── user-permission-audit-log.md
    └── order-refund-flow.md
```

Design document 不是流水帳，也不是做完就丟掉的紀錄，而是專案的重要 context 資產。它應該描述這次功能的設計背景、系統邊界、資料流、API 變更、資料庫變更、測試策略、風險與取捨。未來當功能迭代、架構調整或 bug 修正時，也應該回頭更新相關 Design document。

這麼做的好處是，AI Agent 之後在處理同一個領域的任務時，可以讀取既有 Design document，理解當初為什麼這樣設計，而不是只看目前的程式碼表面。對人類工程師來說，Design document 也能降低 onboarding 成本，讓新成員更快理解系統脈絡。

### 產生 Implementation plan

有了 Design document 後，才進入 Implementation plan。這個階段可以透過 superpowers 的 writing-plans 類型流程，要求 AI Agent 根據這次迭代的 Design 產生具體的實作計畫。

我的做法是將尚未真正實作的 plan 統一放在 `docs/plans/active`：

```plaintext
docs
└── plans
    ├── active
    │   └── 2026-06-07-user-permission-audit-log-plan.md
    └── completed
```

Implementation plan 應該包含這次會修改哪些模組、預期新增哪些檔案、每個 task 的順序、需要補哪些測試、如何驗證、有哪些風險與 rollback 方式。它不需要細到每一行 code 怎麼寫，但必須細到 AI Agent 可以依照 plan 逐步執行，而不是邊做邊猜。

plan 產生後，應該先由產生該 plan 的人 review，並持續與 AI Agent 溝通改善方案。例如，要求 AI Agent 拆小任務、補測試策略、確認資料庫 migration 風險、避免一次修改太多不相關檔案。確認沒問題後，可以發 PR 讓團隊共同 review 這份 plan 是否合理。

這一步很值得花時間，因為 review plan 的成本遠低於 review 一大包已經寫完的程式碼。如果設計方向錯了，在 plan 階段修正只需要改文件；如果等 AI Agent 寫完幾千行 code 才發現方向錯了，成本就會高很多。

### AI Agent 進行實作

Implementation plan 通過 review 後，AI Agent 才正式開始實作。這時候 AI Agent 應該根據 `docs/plans/active` 中的 plan 逐步處理任務，並在過程中讀取 `AGENTS.md`、`docs/policies`、相關 Design document 與團隊 Skills。

在實作階段，前面設置的 Git hook 就會開始發揮作用。每次 AI Agent 修改程式碼並準備 commit 時，pre-commit 可以先執行 formatter、linter 或部分測試，提早攔下明顯違反 policy 的程式碼。如果檢查失敗，AI Agent 應該根據錯誤訊息修正，而不是跳過 hook 或關閉檢查。

實務上，不建議讓 AI Agent 一次完成太大的變更。比較好的方式是依照 Implementation plan 的 task 拆分實作，每完成一個小段落就執行對應驗證。這能降低上下文失控的機率，也讓人類更容易 review。

當實作完成後，AI Agent 應該執行專案定義的 `lint`、`test`、`build` 等命令，確認結果後再將 implementation plan 從 `docs/plans/active` 移到 `docs/plans/completed`。這個動作代表該 plan 已經完成，不再是待執行狀態，也方便未來回頭查找某個功能當初是如何落地的。

### Code Review

AI Agent 實作完成後，可以透過 MCP、Skill 或 CLI 協助建立 PR。PR 描述中應該包含 User Story、Design document、Implementation plan 的連結，以及本地執行過的驗證命令與結果。這樣 reviewer 不需要從一大段 diff 裡反推需求，而是可以沿著 context 從需求一路看回實作。

在 Code Review 階段，可以引入 AI Code Review 機制協助處理第一層檢查，例如：

- 是否有明顯的效能問題，如 N + 1 query、多餘迴圈、重複 I/O。
- 是否有依照 Implementation plan 實作，而不是多做或漏做。
- 是否缺少測試或驗收條件沒有被覆蓋。
- 是否違反 `docs/policies` 中定義的架構與風格。
- 是否出現不必要的大範圍重構。

這些檢查可以降低人類 reviewer 的負擔，讓人類把時間留給更高層次的判斷，例如：這個設計是否真的符合產品目標？是否引入長期維護風險？是否有安全或資料一致性的問題？

但 AI Code Review 不應該被理解成「人類不用 review」。AI 可以幫忙找出明顯問題，也可以確認實作是否對齊 plan，但最終的工程判斷、產品取捨與責任仍然需要人類承擔。特別是當 AI Review 一再指出同類問題時，團隊不應該只是在單次 PR 中修掉，而是要回頭更新 Skills、`docs/policies` 或測試工具，讓同樣的錯誤下次更早被攔住。

## 結論

AI Agent 讓程式碼產出速度大幅提升，但真正決定團隊效率的，不是單次生成了多少 code，而是 **團隊能不能建立一套穩定、可重複、可審查的工作流**。沒有 context、規格與工具防線的 AI 開發，很容易只是把技術債產生得更快。

本文介紹的流程從 PRD、User Story、Design document、Implementation plan 到實作與 Code Review，目的就是把模糊需求逐步轉成可被 AI Agent 正確執行的工程資產。其中，團隊共享的 Skills、`AGENTS.md`、`docs/policies`、Git hook、測試與 CI，都是讓 AI 從「自由發揮」變成「受控協作」的重要機制。

未來軟體工程師的價值，會越來越體現在如何設計這些機制、維護團隊知識、判斷 AI 產出的正確性，並把每次協作的經驗回饋到下一次更好的流程裡。AI 不會取代工程紀律，反而會放大工程紀律的重要性。
