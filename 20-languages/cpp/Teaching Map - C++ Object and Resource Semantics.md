# Teaching Map - C++ Object and Resource Semantics

## Purpose

這份 note 是 Teaching Architecture，不是正式教學文章。

Current expression and talk-style decisions are captured in [[20-languages/cpp/Teaching Notes - C++ Object and Resource Semantics|Teaching Notes]]. Use that note before revising the final chapter outline.

它的任務是先決定：

- 教學主線從哪裡切入。
- 哪些 Deep Dive 是主線，哪些是 side branch。
- Conversation Notes 的原始思考如何被保留。
- Concept cards 在教學裡何時被引用。
- 之後的 `Learning Path` 應該怎麼分章。

原始文章與 Conversation Notes 不在這個階段更動。Deep Dive / Concept 已經經過 [[20-languages/cpp/Knowledge Audit - C++ Object and Resource Semantics|Knowledge Audit]] 的第一輪校準，這份 Teaching Map 只負責把它們排成可教、可讀、可回顧的路線。

## Layer Contract

| Layer | Role | Teaching Use |
| --- | --- | --- |
| Source archive | 保存完整原始對話 | 不直接拿來教學，但所有整理都要能回溯 |
| Question Trail | 保存代表性原始問題與轉折 | 每章開頭可引用原始問題，保留為什麼會問 |
| Conversation Note | 保存完整子對話與思考順序 | 用來防止 Deep Dive 變成二手整理 |
| Knowledge Audit | 校準事實、版本、mental model 邊界 | 寫正式教學前的 correctness gate |
| Deep Dive | 把某條思考線重寫成完整推理文章 | 每個 teaching module 的主體來源 |
| Concept | 穩定、短、可查的 lookup card | 每章結尾或旁註使用，不取代推理過程 |
| Teaching Map | 安排教學架構 | 決定主線、旁支、coverage |
| Learning Path | 最終可讀教學路線 | 下一階段才產出 |

## Main Teaching Decision

RVO 是這組筆記的原始入口，但不應該被教成孤立的 compiler trick。正式教學的第一個畫面仍然應該是具體 code：

```cpp
T make() {
    T x;
    return x;
}
```

但這段 code 背後要拉出的 first principle 是 object delivery：

```text
A 要生成一個 T。
B 要怎麼拿到這個 T？
```

原始學習路徑：

```text
意外遇到 RVO
-> 追問 return by value 為什麼不用 copy
-> 追到 move / storage / lifetime / RAII / Rust
```

正式概念骨架應該反過來：

```text
Object delivery problem
-> Copy worry
-> Copy semantics: usable object, not just similar bytes
-> Buffer shows copy can be wrong, expensive, or impossible
-> Move appears as ownership transfer when source can be abandoned
-> RVO / NRVO / prvalue 是 return-by-value 的 no-transfer case
-> RAII / type invariants 把 resource lifetime 提升成 type semantics
```

也就是：

```text
RVO = motivating hook
return local = first concrete teaching case
Object delivery = underlying teaching frame
```

核心句：

```text
RVO 是避免搬移。
move 是搬移不可避免時，便宜地轉移 ownership。
現代 C++ 的重點不是到處 std::move，
而是讓物件盡量直接出生在它最後要待的位置。
```

## Teaching Thesis

C++ object/resource semantics 的核心問題不是「怎麼最佳化 copy」，而是：

```text
一個 T 在程式中被建立、交付、移動、擁有、銷毀時，
它的 lifetime 從哪裡開始？
resource ownership 由誰維護？
交付後語意是否仍然被保留？
哪些 operations 合法？
operation 後 invariant 是否仍成立？
compiler / library 可以在哪些條件下省略、移動或重建 object？
```

這些問題不要壓成同一層技巧。最新分層：

```text
Lifetime semantics:
    RAII
    resource lifetime follows object lifetime

Transfer semantics:
    copy / move
    object 被交付時 ownership / invariant 是否保持

Construction placement semantics:
    RVO / NRVO / copy elision / emplace
    object 能不能直接在 destination storage 形成

Validity / invariant:
    valid state
    moved-from state
    ownership invariant
```

## Previous Learning Path Draft

This table is the earlier architecture draft. The current official readable route has been revised into a talk-style outline in [[20-languages/cpp/Learning Path - C++ Object and Resource Semantics|Learning Path]], using [[20-languages/cpp/Teaching Notes - C++ Object and Resource Semantics|Teaching Notes]] as the decision source.

| Module | Teaching Question | Naive Model To Break | Main Ideas | Primary Deep Dives | Concept Cards |
| --- | --- | --- | --- | --- | --- |
| 0. Original Hook | 為什麼這組筆記會從 RVO 開始？ | RVO 是孤立最佳化技巧 | RVO 是入口，不是全局中心 | [[20-languages/cpp/deep-dives/Deep Dive - RVO Return Slot and Naive Copy Model]] | [[20-languages/cpp/object-resource-semantics/Concept - RVO and NRVO]] |
| 1. Object Delivery Problem | `T` 要從 A 交到 B 時有哪些策略？ | return/pass/store 都只是 copy bytes | Copy / Move / In-place 是三種 object delivery strategy | [[20-languages/cpp/deep-dives/Deep Dive - Object Not Just Data Report Thesis]] | [[20-languages/cpp/object-resource-semantics/Concept - Object Delivery - Copy Move In-place Construction]] |
| 2. Return By Value | 為什麼 `return T{}` 不一定需要 temporary？ | callee 先建，再 copy 到 caller | result object、C++17 prvalue、RVO / NRVO | [[20-languages/cpp/deep-dives/Deep Dive - RVO Return Slot and Naive Copy Model]], [[20-languages/cpp/deep-dives/Deep Dive - Half RVO Misconception Storage and Lifetime]] | [[20-languages/cpp/object-resource-semantics/Concept - Copy Elision and C++17 prvalue]], [[20-languages/cpp/object-resource-semantics/Concept - Storage vs Object Lifetime]] |
| 3. `std::move` And Value Categories | `std::move` 到底搬了什麼？ | `std::move(x)` 會搬 object | xvalue、move constructor、return path selection、C++23 fallback | [[20-languages/cpp/deep-dives/Deep Dive - std move xvalue and Return Path Selection]], [[20-languages/cpp/deep-dives/Deep Dive - Value Categories Beyond std move]] | [[20-languages/cpp/object-resource-semantics/Concept - std move vs Move Constructor]], [[20-languages/cpp/object-resource-semantics/Concept - Value Categories - lvalue xvalue prvalue]] |
| 4. Move As Ownership Transfer | 為什麼 `Matrix4x4` move/copy 差不多，但 `Buffer` move 很重要？ | move = faster copy | inline value data vs resource owner、moved-from state、ownership transfer | [[20-languages/cpp/deep-dives/Deep Dive - Move Ownership Matrix4x4 and Resource Transfer]] | [[20-languages/cpp/object-resource-semantics/Concept - Move Semantics and Ownership]] |
| 5. Storage And Lifetime | 有 storage 就有 object 嗎？ | memory allocation = object construction | storage、lifetime、placement new、`construct_at` | [[20-languages/cpp/deep-dives/Deep Dive - Half RVO Misconception Storage and Lifetime]], [[20-languages/cpp/deep-dives/Deep Dive - Destination First Construction Across RVO Emplace and Placement New]] | [[20-languages/cpp/object-resource-semantics/Concept - Storage vs Object Lifetime]], [[20-languages/cpp/object-resource-semantics/Concept - Placement New and construct_at]] |
| 6. Destination-First Construction | 什麼時候不要先建立 `T`？ | `emplace` / factory lambda 只是語法糖 | `emplace` boundary、factory delayed construction、final storage known first | [[20-languages/cpp/deep-dives/Deep Dive - Destination First Construction Across RVO Emplace and Placement New]], [[20-languages/cpp/deep-dives/Deep Dive - Factory Lambda Is Not Syntax Sugar]] | [[20-languages/cpp/object-resource-semantics/Concept - In-place Construction and emplace]], [[20-languages/cpp/object-resource-semantics/Concept - Factory Lambda and Delayed Construction]] |
| 7. RAII And Rule Of 0/3/5 | 為什麼寫 destructor 會牽動 copy/move？ | class 只是 fields + destructor | resource owner 的 lifecycle operations、Rule of Zero、`noexcept` move | [[20-languages/cpp/deep-dives/Deep Dive - Buffer Bug RAII Rules and noexcept Move]] | [[20-languages/cpp/object-resource-semantics/Concept - RAII and Rule of 0 3 5]], [[20-languages/cpp/object-resource-semantics/Concept - noexcept Move and Container Reallocation]] |
| 8. Type Semantics | type 為什麼不只是 data layout？ | type = struct fields | valid states、operations、invariants、regular / semiregular、generic programming | [[20-languages/cpp/deep-dives/Deep Dive - C Convention to Cpp Semantic Lifting]], [[20-languages/cpp/deep-dives/Deep Dive - Type Operations Laws Invariants and Stepanov]] | [[20-languages/cpp/object-resource-semantics/Concept - C vs C++ Semantic Lifting]], [[20-languages/cpp/object-resource-semantics/Concept - C++ Type as Operations and Invariants]] |
| 9. Design Space | C++ / Rust / GC 在解同一個問題嗎？ | Rust = C++ move + strict compiler | ownership invariant 放在 programmer / type / compiler / runtime 哪一層 | [[20-languages/cpp/deep-dives/Deep Dive - Ownership Design Space Cpp Rust GC]] | [[20-languages/rust/Concept - Rust Ownership Compared With C++ Move]] |

## Primary Path

正式 `Learning Path` 應該採用這條主線：

```text
Return by value: how does the caller get a usable T?
-> Copy semantics: usable object, not just similar bytes
-> C Buffer: representation copy loses meaning
-> C++ Buffer: copy/destroy become type operations
-> RAII / Rule of 0/3/5
-> Move branch: ownership transfer when copy is wrong, expensive, or impossible
-> std::move misconception / value categories
-> return by value / RVO as no-transfer case
-> C convention to C++ type semantics
-> type invariants
```

RVO 可以在前言中作為原始 hook：

```text
我最初只是想知道 RVO 是什麼；
但 RVO 其實逼出了 C++ object/resource semantics 的整條問題鏈。
```

最新教學轉向：

```text
Return by value 應該成為正式開場。
RVO 是 hook，但第一個問題不是「什麼是 RVO」，
而是「caller 那邊到底怎麼出現一個可以正常使用的 T」。

主線不是 RVO，也不只是 move。
核心是 ownership / lifetime / resource semantics 如何被 type operations 保留。

C Buffer 不消失，但它也不該等到 move 之後才出現。
它應該在 copy 焦慮之後出現，
用來證明 representation copy 不等於 semantic copy，
並且替 move 的必要性鋪路。
```

## Side Branches

這些內容有價值，但不應打斷主線：

| Side Branch | Where To Attach | Why It Is Side Branch |
| --- | --- | --- |
| `noexcept` move / vector reallocation | After RAII / Rule of 0/3/5 | 是 generic library + exception guarantee 的案例，不是 object delivery 主線 |
| regularity / concepts / Stepanov | After type invariants | 是 generic programming requirements 的下一層，不是這場 object/resource 主線必經章節 |
| ABI return slot / `sret` | Module 2 or 5 | 是 systems mental model，不是 C++ standard semantics |
| Compiler flags / `-fno-elide-constructors` | Module 2 appendix | 是 verification tool，不是概念主線 |
| raw storage / placement new / `construct_at` | After RVO revisited | 是 low-level lifetime control，不是開場需要的模型 |
| `emplace` / factory lambda | After final-storage extension | 容易把 RVO / emplace / placement new / factory API 混在一起 |
| Engine / browser analogies | Module 6 appendix | 可幫理解 delayed construction，但要避免過度類比 |
| CppCon watchlist | End of path | 作為 further reading，不塞進主線 |
| Thinking method self-analysis | Meta appendix | 保留學習方法，不與 C++ 概念混排 |

## Conversation Note Coverage Matrix

| Conversation Note | Teaching Modules | Coverage Role |
| --- | --- | --- |
| [[20-languages/cpp/conversation-notes/Conversation Note - RVO Basics Return Slot and In-place Analogies]] | 0, 2, 5, 6 | 原始 RVO hook、return slot、half-RVO 誤解、RVO-like analogies |
| [[20-languages/cpp/conversation-notes/Conversation Note - Factory Lambda and Delayed Construction]] | 6 | factory lambda、delayed construction、engine/browser analogy |
| [[20-languages/cpp/conversation-notes/Conversation Note - RVO Verification Compile Flags and Placement New]] | 2, 5 | verification、placement new、`construct_at`、compile flags |
| [[20-languages/cpp/conversation-notes/Conversation Note - Move Ownership and Value Categories]] | 3, 4 | `std::move`、xvalue/prvalue、Matrix4x4、ownership transfer |
| [[20-languages/cpp/conversation-notes/Conversation Note - Report Topic Object Delivery RAII and Object Not Data]] | 1, 7, 8 | object delivery framing、RAII、Rule of 0/3/5、report thesis |
| [[20-languages/cpp/conversation-notes/Conversation Note - C vs C++ Semantic Lifting and Hidden Semantics]] | 8, 9 | semantic lifting、C convention to C++ type semantics、GC bridge |
| [[20-languages/cpp/conversation-notes/Conversation Note - Type Operations Mathematical Analogy and Stepanov]] | 5, 8 | storage/lifetime systems diagrams、algebra analogy、Stepanov |
| [[20-languages/cpp/conversation-notes/Conversation Note - Rust Comparison and CppCon Watchlist]] | 9 | Rust ownership comparison、CppCon watchlist |
| [[20-languages/cpp/conversation-notes/Conversation Note - Knowledge Tree and C++ Theme Expansion]] | All modules / appendix | skill tree and larger C++ object/resource theme |
| [[20-languages/cpp/conversation-notes/Conversation Note - Thinking Method Self Analysis and Expert Viewpoints]] | Meta appendix | layered semantic decomposition and learning method |

## Deep Dive Role Matrix

| Deep Dive | Role In Teaching |
| --- | --- |
| [[20-languages/cpp/deep-dives/Deep Dive - RVO Return Slot and Naive Copy Model]] | Main explanation for return-by-value and naive copy failure |
| [[20-languages/cpp/deep-dives/Deep Dive - Half RVO Misconception Storage and Lifetime]] | Corrects storage/lifetime confusion and ABI mental model boundary |
| [[20-languages/cpp/deep-dives/Deep Dive - std move xvalue and Return Path Selection]] | Main explanation for `std::move`, xvalue, C++23 fallback nuance |
| [[20-languages/cpp/deep-dives/Deep Dive - Value Categories Beyond std move]] | Support material for value category vocabulary |
| [[20-languages/cpp/deep-dives/Deep Dive - Move Ownership Matrix4x4 and Resource Transfer]] | Main explanation for why move is about ownership, not speed |
| [[20-languages/cpp/deep-dives/Deep Dive - Destination First Construction Across RVO Emplace and Placement New]] | Main bridge from RVO to in-place construction family |
| [[20-languages/cpp/deep-dives/Deep Dive - Factory Lambda Is Not Syntax Sugar]] | Main delayed construction case study |
| [[20-languages/cpp/deep-dives/Deep Dive - Buffer Bug RAII Rules and noexcept Move]] | Main RAII / Rule of 0/3/5 / noexcept chapter source |
| [[20-languages/cpp/deep-dives/Deep Dive - Object Not Just Data Report Thesis]] | Module 1 framing and final report thesis |
| [[20-languages/cpp/deep-dives/Deep Dive - C Convention to Cpp Semantic Lifting]] | Type semantics and C-to-C++ bridge |
| [[20-languages/cpp/deep-dives/Deep Dive - Type Operations Laws Invariants and Stepanov]] | Generic programming / regularity / operation laws |
| [[20-languages/cpp/deep-dives/Deep Dive - Ownership Design Space Cpp Rust GC]] | Cross-language design-space endpoint |
| [[20-languages/cpp/deep-dives/Deep Dive - Cpp Object Resource Skill Tree and Thinking Method]] | Appendix / meta learning map |

## Concept Card Use

Concept cards should not be the first teaching layer. Use them as:

- quick definitions after a full explanation;
- recap blocks at the end of modules;
- cross-links when a term reappears;
- review cards after reading the Deep Dive.

The formal teaching article should not be assembled by concatenating Concepts. That would recreate the earlier problem: compact lookup notes lose the original question pressure and reasoning depth.

## Learning Path Output Rules

When writing the final `Learning Path`:

- open each module with the original question or failure model;
- preserve at least one link to the relevant Conversation Note;
- explain naive model first, then break it;
- mark whether a claim is standard semantics, library behavior, guideline, or ABI/model;
- keep RVO as the original hook, but teach object delivery as the core problem;
- avoid saying RVO / `emplace` / factory lambda are the same mechanism;
- use Concept cards only after the reasoning is established.

## Next File

Created next phase file:

- [[20-languages/cpp/Learning Path - C++ Object and Resource Semantics]]

That file is a readable route through the modules, not a full textbook chapter yet.
