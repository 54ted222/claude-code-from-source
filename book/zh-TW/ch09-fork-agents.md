# 第九章：Fork Agent 與 Prompt Cache

## 百分之九十五的洞察

當一個父 agent 同時產生五個子 agent 時，每個子 agent 的 API 請求中絕大部分內容是完全相同的。系統提示詞是一樣的。工具定義是一樣的。對話歷史是一樣的。觸發產生的 assistant 訊息也是一樣的。唯一不同的只有最後的指令：「你負責資料庫遷移」、「你撰寫測試」、「你更新文件」。

在一個已有暖身對話的典型 fork 場景中，共享前綴可能高達 80,000 個 token。每個子 agent 的專屬指令可能只有 200 個 token。這代表 99.75% 的重疊率。Anthropic 的 prompt cache 對快取命中的輸入 token 提供九折優惠。如果你能讓那 80,000 個 token 在第二到第五個子 agent 中命中快取，你就把這四個請求的輸入成本降低了 90%。對父 agent 而言，這就是在同一次平行調度中花 $4 和花 $0.50 的差別。

問題在於 prompt cache 是位元組精確比對的。不是「夠相似」。不是「語義等價」。從系統提示詞的第一個位元組到每個子 agent 內容開始分歧之前的最後一個位元組，每個位元組都必須完全一致。多一個空格、工具定義順序不同、一個過期的 feature flag 改變了系統提示詞片段——快取就會失效。整個前綴會以全價重新處理。

Fork agent 是 Claude Code 對這個限制的解答。它們不只是「帶著上下文產生子 agent」的便利功能——它們是一個偽裝成編排功能的 prompt cache 利用機制。Fork 系統中的每一個設計決策都回溯到同一個問題：我們如何保證平行子 agent 之間有位元組完全相同的前綴？

---

## Fork 子 Agent 繼承了什麼

一個 fork agent 從父 agent 繼承四樣東西，而且是透過引用或位元組精確複製來繼承，而非重新計算。

**1. 系統提示詞。** 不是重新生成——而是直接傳遞。父 agent 已經渲染好的系統提示詞位元組透過 `override.systemPrompt` 傳入，取自 `toolUseContext.renderedSystemPrompt`。這正是父 agent 最近一次 API 呼叫中發送的那個確切字串。

**2. 工具定義。** Fork agent 定義中宣告了 `tools: ['*']`，但由於 `useExactTools` flag 設為 true，子 agent 直接接收父 agent 已組裝好的工具陣列。不經過篩選、不重新排序、不重新序列化。

**3. 對話歷史。** 父 agent 與 API 之間交換的每一條訊息——使用者輪次、assistant 輪次、工具呼叫、工具結果——都透過 `forkContextMessages` 被複製到子 agent 的上下文中。

**4. Thinking 設定與模型。** Fork 定義指定 `model: 'inherit'`，這會解析為父 agent 的確切模型。相同的模型意味著相同的 tokenizer、相同的 context window、相同的 cache 命名空間。

Fork agent 的定義本身非常精簡——幾乎是一個空操作：

Fork agent 的定義被刻意設計得極為精簡——它繼承父 agent 的一切。它指定所有工具（`'*'`），繼承父 agent 的模型，使用 bubble 模式處理權限（讓提示浮現到父 agent 的終端機），並提供一個永遠不會被實際呼叫的空操作系統提示詞函式——真正的提示詞透過 override 通道傳入，已經渲染完成且位元組穩定。

---

## 位元組相同前綴的技巧

送給 Claude 的 API 請求有特定的結構：系統提示詞，然後是工具，然後是訊息。要讓 prompt cache 命中，從請求開始到某個前綴邊界的每一個位元組在不同請求之間必須完全相同。

Fork agent 透過確保三個層次被凍結來達成這一點：

**第一層：透過傳遞而非重新計算的系統提示詞。**

當父 agent 的系統提示詞在最近一次 API 呼叫中被渲染時，結果被捕獲在 `toolUseContext.renderedSystemPrompt` 中。這是經過所有動態插值之後的字串——GrowthBook feature flag、環境細節、MCP 伺服器描述、skill 內容、CLAUDE.md 檔案。Fork 子 agent 接收的正是這個確切的字串。

為什麼不直接再次呼叫 `getSystemPrompt()`？因為系統提示詞的生成不是純函式。GrowthBook flag 會隨著 SDK 取得遠端設定而從冷啟動狀態轉換為暖狀態。一個在父 agent 第一輪回傳 `false` 的 flag，到 fork 子 agent 啟動時可能回傳 `true`。如果系統提示詞中包含一個以該 flag 為條件的區塊，重新渲染的提示詞就會產生哪怕只有一個字元的差異。快取失效。80,000 個 token 的前綴以全價重新處理，乘以五個子 agent。

傳遞已渲染的位元組消除了這整類的分歧問題。

**第二層：透過精確傳遞的工具定義。**

一般的 sub-agent 會經過 `resolveAgentTools()`，它會根據 agent 定義中的 `tools` 和 `disallowedTools` 陣列篩選工具池，套用權限模式差異，並可能重新排序工具。產生的序列化工具陣列會與父 agent 的不同——不同的子集、不同的順序、不同的權限標記。

Fork agent 完全跳過這一步：

```typescript
const resolvedTools = useExactTools
  ? availableTools  // parent's exact array
  : resolveAgentTools(agentDefinition, availableTools, isAsync).resolvedTools
```

`useExactTools` flag 僅在 fork 路徑上被設為 true。子 agent 原封不動地取得父 agent 的工具池。相同的工具、相同的順序、相同的序列化。這包括在子 agent 的工具池中保留 Agent 工具本身，即使子 agent 被禁止使用它——移除它會改變工具陣列並使快取失效。

**第三層：訊息陣列的建構。**

這是 `buildForkedMessages()` 仔細處理的工作。這個函式建構最後兩條訊息，位於共享歷史與每個子 agent 專屬指令之間：

`buildForkedMessages()` 函式建構最後兩條訊息，位於共享歷史與每個子 agent 專屬指令之間。演算法如下：

1. 複製父 agent 的 assistant 訊息（保留所有 `tool_use` 區塊及其原始 ID）。
2. 對每個 `tool_use` 區塊，建立一個帶有固定佔位字串的 `tool_result`（所有子 agent 完全相同）。
3. 建構一條包含所有佔位結果的使用者訊息，後面接著用 boilerplate 標籤包裹的每個子 agent 專屬指令。
4. 回傳 `[clonedAssistantMessage, userMessageWithPlaceholdersAndDirective]`。

```typescript
// Pseudocode — illustrates the message construction
function buildChildMessages(directive, parentAssistant) {
  const cloned = cloneMessage(parentAssistant)
  const placeholders = parentAssistant.toolUseBlocks.map(b =>
    toolResult(b.id, CONSTANT_PLACEHOLDER)  // Byte-identical across children
  )
  const userMsg = createUserMessage([...placeholders, wrapDirective(directive)])
  return [cloned, userMsg]
}
```

每個子 agent 產生的訊息陣列看起來像這樣：

```
[...shared_history, assistant(all_tool_uses), user(placeholder_results..., directive)]
```

在指令之前的每一個元素在所有子 agent 之間都是相同的。`FORK_PLACEHOLDER_RESULT`——一個固定字串 `'Fork started -- processing in background'`——確保連工具結果區塊也是位元組相同的。`tool_use_id` 的值也是相同的，因為它們引用的是同一條 assistant 訊息。只有最後的文字區塊——包含每個子 agent 專屬指令的部分——是不同的。

快取邊界正好落在那個最後文字區塊之前。在它之上的所有內容——可能數萬個 token 的系統提示詞、工具定義、對話歷史和佔位結果——對第一個之後的每個子 agent 都以九折價格命中快取。

---

## Fork Boilerplate 標籤

每個子 agent 的指令被包裹在一個 boilerplate XML 標籤中，它有兩個用途：指導子 agent 如何行為，以及作為遞迴 fork 偵測的標記。

Boilerplate 包含大約 10 條規則。其中關鍵的幾條：

- **覆蓋父 agent 的 fork 指令。** 父 agent 的系統提示詞說「預設採用 forking」——boilerplate 明確告訴子 agent：「那條指令是給父 agent 的。你就是那個 fork。不要產生 sub-agent。」
- **靜默執行，報告一次。** 在工具呼叫之間不要產生對話文字。直接使用工具，然後產生一份結構化的摘要。
- **保持在範圍內。** 子 agent 不得擴展超出其指令的範圍。
- **結構化輸出格式。** 回應必須遵循 Scope/Result/Key files/Files changed/Issues 的模板，使得當多個子 agent 同時回報時，父 agent 容易解析結果。

規則 1 特別有趣。父 agent 的系統提示詞——fork 子 agent 為了快取原因而完整繼承的——包含「當你有平行工作時預設採用 forking」之類的指令。如果子 agent 遵循這條指令，它就會嘗試 fork 自己的子 agent，造成無限遞迴的 agent 鏈。Boilerplate 明確覆蓋：「那條指令是給父 agent 的。你就是那個 fork。」

結構化輸出格式（Scope/Result/Key files/Files changed/Issues）不是裝飾性的。它將子 agent 的輸出限制在事實性的報告，使得當五個子 agent 同時回報時，父 agent 更容易解析和彙總結果。

---

## 遞迴 Fork 防護

Fork 子 agent 在其工具池中保留了 Agent 工具。這是必要的——移除它會改變序列化的工具陣列並使 prompt cache 失效。但如果子 agent 在沒有指定 `subagent_type` 的情況下實際調用了 Agent 工具，fork 路徑就會再次觸發，建立一個孫子級 fork。這個孫子 agent 會繼承更大的上下文（父 agent + 子 agent 的對話），產生它自己的 fork，依此類推。

兩個防護措施阻止了這種情況：

**主要防護：querySource 檢查。** 當 fork 子 agent 被產生時，其 `context.options.querySource` 被設為 `'agent:builtin:fork'`。`call()` 方法在允許 fork 路徑之前會檢查這個值：

```typescript
// In AgentTool.call():
if (effectiveType === undefined) {
  // Fork path -- but are we already in a fork?
  if (querySource === 'agent:builtin:fork') {
    // Reject: already a fork child
  }
}
```

這是快速路徑。它只檢查 options 物件中的一個字串。

**備援防護：訊息掃描。** Fork 防護使用兩層保護：在產生時設定的 `querySource` 標記（快速路徑——一次字串比較），以及一個掃描訊息歷史尋找 boilerplate XML 標籤的備援機制。備援機制之所以存在，是因為 `querySource` 在 autocompact 過程中會被保留，但在某些未正確傳遞的邊緣情況下，訊息掃描備援能捕捉到遞迴。這是一種雙重保險的做法，其中檢查的成本（掃描訊息）與意外遞迴 fork 的成本（失控的 API 開銷）相比微不足道。

為什麼需要備援？因為 Claude Code 有一個 autocompact 功能，它會在上下文過長時重寫訊息陣列。Autocompact 可以重寫訊息內容但會保留 options 中的 `querySource`。理論上，單靠 `querySource` 就足夠了。實務上，訊息掃描備援能捕捉 `querySource` 未正確傳遞的邊緣情況——這是一種雙重保險的做法，其中檢查的成本（掃描訊息）與意外遞迴 fork 的成本（失控的 API 開銷）相比微不足道。

---

## 同步到非同步的轉換

Fork 子 agent 最初在前景執行：它的訊息串流到父 agent 的終端機，父 agent 阻塞等待完成。但如果子 agent 執行時間太長怎麼辦？Claude Code 允許在執行過程中將前景 agent 推到背景——使用者（或自動逾時）可以把正在執行的前景 agent 推到背景，而不會丟失任何工作。

這個機制出奇地乾淨：

1. 當前景 agent 透過 `registerAgentForeground()` 註冊時，會建立一個背景信號 promise。

2. 父 agent 的同步迴圈在 agent 的訊息串流與背景信號之間競爭：

```
while (true) {
  const result = await Promise.race([
    iterator.next(),         // next message from agent
    backgroundSignal,        // "move to background" trigger
  ])
  if (result === BACKGROUND_SIGNAL) break
  // ... process message
}
```

3. 當背景信號觸發時，前景迭代器透過 `iterator.return()` 被優雅地終止。這會觸發 generator 的 `finally` 區塊來處理清理工作。

4. 一個新的 `runAgent()` 實例以 `isAsync: true` 啟動，使用相同的 agent ID 和到目前為止累積的訊息歷史。Agent 從中斷處繼續，現在在背景執行。

5. 原始的同步 `call()` 回傳 `{ status: 'async_launched' }`，父 agent 繼續其對話。

不會丟失任何工作，因為訊息歷史就是 agent 的狀態。磁碟上的 sidechain 記錄包含 agent 產生的每一條訊息。新的非同步實例從這份記錄重放，並從同步實例停止的地方接續。

---

## 自動背景化

當 `CLAUDE_AUTO_BACKGROUND_TASKS` 環境變數或 `tengu_auto_background_agents` GrowthBook flag 啟用時，前景 agent 會在 120 秒後自動被推到背景：

透過環境變數或 feature flag 啟用後，前景 agent 會在 120 秒後自動背景化。停用時，函式回傳 0（不自動背景化）。

這是一個有成本影響的 UX 決策。前景 agent 會阻塞父 agent 的終端機——使用者無法輸入、無法發出新指令、無法產生其他 agent。兩分鐘的時間足夠 agent 同步完成大多數快速任務（串流輸出是有用的回饋），但又短到不會讓長時間執行的任務佔據終端機。

在 fork 實驗下，自動背景化的問題變得無關緊要：所有 fork 產生的子 agent 從一開始就被強制為非同步。`run_in_background` 參數從 schema 中完全隱藏。每個 fork 子 agent 都在背景執行，完成後透過 `<task-notification>` 回報，父 agent 永遠不會阻塞。

---

## 何時不使用 Fork

Fork 是多種編排模式之一，在三種情況下它被刻意排除：

**Coordinator 模式。** Coordinator 模式和 fork 模式是互斥的。Coordinator 有結構化的委派模型：它維護一個計畫、以明確的提示詞分配任務給 worker、並追蹤進度。Fork 的「繼承一切」方式會破壞這個模型。一個被 fork 的 coordinator 會繼承父 coordinator 的系統提示詞（其中寫著「你是 coordinator，委派工作」），子 agent 就會嘗試編排而非執行。`isForkSubagentEnabled()` 函式會先檢查 `isCoordinatorMode()`，如果啟用則回傳 false。

**非互動式 session。** SDK 和 API 消費者（`--print` 模式、Claude Agent SDK）在沒有終端機的情況下運作。Fork 的 `permissionMode: 'bubble'` 會將權限提示浮現到父 agent 的終端機——而在非互動模式下終端機不存在。與其為此建構一個獨立的權限流程，fork 路徑直接被停用。SDK 消費者改用明確的 `subagent_type` 選擇。

**明確指定 subagent_type。** 當模型指定了 `subagent_type`（例如 `"Explore"`、`"Plan"`、`"general-purpose"`）時，fork 路徑不會被觸發。Fork 僅在 `subagent_type` 被省略時才啟動。這讓模型可以在「我想要一個有自己系統提示詞和工具集的專門 agent」（明確指定類型）和「我想要一個繼承我的上下文的複製體來平行處理這件事」（省略類型）之間做選擇。

---

## 經濟效益分析

考慮一個具體的場景。一位開發者要求 Claude Code 重構一個模組。父 agent 分析了程式碼庫，形成計畫，並平行調度五個 fork 子 agent：一個更新資料庫 schema、一個重寫服務層、一個更新路由、一個修正測試、一個更新型別定義。

在對話的這個時間點，共享的上下文相當龐大：
- 系統提示詞：約 4,000 個 token
- 工具定義（40 多個工具）：約 12,000 個 token
- 對話歷史（分析 + 規劃）：約 30,000 個 token
- 包含五個 tool_use 區塊的 assistant 訊息：約 2,000 個 token
- 佔位工具結果：約 500 個 token

共享前綴總計：約 48,500 個 token。每個子 agent 的專屬指令：約 200 個 token。

不使用 fork（五個獨立 agent，各自有全新的上下文和自己的系統提示詞）：
- 每個子 agent 處理自己的系統提示詞 + 工具 + 任務提示
- 沒有快取共享（不同的系統提示詞、不同的工具集）
- 成本：5 倍全額輸入處理

使用 fork（位元組相同的前綴）：
- 子 agent 1：48,700 個 token 以全價計費（第一個請求快取未命中）
- 子 agent 2-5：48,500 個 token 以 10% 價格計費（快取命中）+ 每個 200 個 token 以全價計費
- 子 agent 2-5 的有效成本：約 4,850 + 200 = 約 5,050 個 token 等價

節省的幅度隨上下文大小和子 agent 數量而增長。對於一個有 100K token 歷史的暖 session 產生 8 個平行 fork，快取節省可以超過沒有共享時輸入 token 成本的 90%。

這就是為什麼 fork 系統中的每一個設計決策——傳遞而非重新計算、精確的工具傳遞、佔位結果、甚至在子 agent 的工具池中保留被禁止使用的 Agent 工具——都為了一件事做最佳化：位元組相同的前綴。每個決策都用少量的優雅或安全性換取可衡量的 API 成本降低。

---

## 設計張力

Fork 系統做了一些值得理解的明確權衡：

**隔離性 vs. 快取效率。** Fork 子 agent 繼承一切，包括可能與其任務無關的對話歷史。一個重寫測試的子 agent 不需要父 agent 討論資料庫 schema 設計的那 15 條訊息。但包含這些訊息正是讓前綴保持相同的關鍵。剔除無關歷史可以節省 context window 空間，代價是使快取失效。這個設計的賭注是：快取節省大於上下文的額外開銷。

**安全性 vs. 快取效率。** Agent 工具留在 fork 子 agent 的工具池中，即使子 agent 不得使用它。移除它會更安全（子 agent 甚至無法嘗試 fork），但會改變工具陣列的序列化。Boilerplate 標籤和遞迴 fork 防護是補償性控制——執行時期的防止而非靜態移除。

**簡潔性 vs. 快取效率。** 佔位工具結果是一個謊言。子 agent 對父 agent assistant 訊息中的每個 tool_use 區塊看到的都是 `'Fork started -- processing in background'`，無論那些工具呼叫實際做了什麼。這是可以接受的，因為子 agent 的指令告訴它該做什麼——它不需要父 agent 調度輪次中準確的工具結果。但這意味著子 agent 的對話歷史在技術上是不連貫的。佔位字串的選擇是為了簡潔和一致性，而非準確性。

這些權衡中的每一個都反映了同樣的優先順序：當你按 token 為 API 呼叫付費且規模龐大時，位元組相同的前綴值得你為之扭曲整個架構。

---

## 實踐應用：為 Prompt Cache 效率而設計

Fork agent 模式可以推廣到 Claude Code 之外。任何從相同上下文調度多個平行 LLM 呼叫的系統都能從快取感知的請求建構中受益。原則如下：

**1. 傳遞已渲染的提示詞，不要重新計算。** 如果你的系統提示詞包含任何動態內容——feature flag、時間戳、使用者偏好、A/B 測試變體——捕獲渲染後的結果並以值傳遞給子 agent。重新計算會有分歧的風險。

**2. 凍結工具陣列。** 如果你的子 agent 需要不同的工具集，你就放棄了工具區塊的快取共享。考慮保留完整的工具集，使用執行時期防護（如 fork boilerplate 的「不要使用 Agent」）來替代編譯時期的移除。

**3. 最大化共享前綴，最小化每個子 agent 的後綴。** 結構化你的訊息陣列，讓所有共享內容排在前面，每個子 agent 的專屬內容附加在最後。交錯排列共享和專屬內容會打碎快取邊界。

**4. 對可變內容使用固定佔位字串。** 當訊息結構需要對先前工具呼叫的回應時，在所有子 agent 之間使用相同的佔位字串，而非實際的（各不相同的）結果。

**5. 衡量損益平衡點。** 快取共享有其開銷：每個子 agent 更大的 context window（它們攜帶無關的歷史）、執行時期防護而非靜態安全、架構複雜度。計算你的平行化模式（多少個子 agent、共享前綴有多大）在扣除額外上下文 token 之後是否真的節省了成本。

Fork agent 系統的核心是一個 prompt cache 利用引擎。它回答了每個多 agent 系統建構者最終都會面臨的問題：當快取對重複前綴提供 90% 的折扣時，你願意為了獲得這個折扣而把架構重組到什麼程度？Claude Code 的答案是：非常徹底。
