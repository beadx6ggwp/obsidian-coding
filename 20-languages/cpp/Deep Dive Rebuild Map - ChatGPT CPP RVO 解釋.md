# Deep Dive Rebuild Map - ChatGPT CPP RVO 解釋

## Purpose

這份 map 記錄 Deep Dive layer 的重建決策。

原本的 Deep Dive 不是完整從 Conversation Note 產生，而是先由早期 Concept draft 放大，因此會把原始對話裡的追問、卡住點、修正過程壓扁。這次重建採用新的來源關係：

```text
Source Archive
-> Question Trail
-> Conversation Note
-> Deep Dive
-> Concept
```

Deep Dive 可以引用 cppreference / ISO draft / Core Guidelines / CppCon，但外部來源只能用來校準與延伸，不能取代 Conversation Note 的問題脈絡。

## Rebuild Policy

- 舊七篇 Deep Dive 視為無效草稿，已移除，不保留舊檔名作為主架構。
- 新 Deep Dive 的 topic boundary 由 Conversation Notes 決定，不由舊 Concept 或舊 Deep Dive 決定。
- 一篇 Conversation Note 可以拆成多篇 Deep Dive，尤其當原始對話本來就有多條思考線。
- 一篇 Deep Dive 可以跨多篇 Conversation Note，但只能在原始對話本來就跨到那個問題時這樣做。
- 每篇 Deep Dive 必須保留 `Original Questions`、`Conversation Reconstruction`、`Rebuild Source Rule`。
- Deep Dive 的任務不是變成 cppreference 摘要，而是把「你為什麼這樣問、卡在哪裡、最後模型怎麼修正」重寫成可閱讀文章。
- Concept notes 之後要從新 Deep Dive 重新抽出，不能再從舊 Concept 壓縮稿互相抄。

## Removed Old Deep Dives

以下舊 Deep Dive 因為不是從 Conversation Note 重建，已移除：

- `Deep Dive - From RVO to Object Delivery Model`
- `Deep Dive - Storage Object Lifetime and Return Slot`
- `Deep Dive - Why return std move Can Be Worse`
- `Deep Dive - Factory Lambda Delayed Construction and Final Storage`
- `Deep Dive - C++ Object Is Not Just Data`
- `Deep Dive - From C Convention to C++ Type Semantics`
- `Deep Dive - Type Operations Invariants and Mathematical Analogy`

## Regenerated Deep Dive Map

| New Deep Dive | Primary Conversation Notes | Why This Boundary Exists |
| --- | --- | --- |
| [[20-languages/cpp/deep-dives/Deep Dive - RVO Return Slot and Naive Copy Model]] | [[20-languages/cpp/conversation-notes/Conversation Note - RVO Basics Return Slot and In-place Analogies]] | 對應最初的 `RVO 是什麼`、naive return-by-value model、caller-provided return slot。 |
| [[20-languages/cpp/deep-dives/Deep Dive - Half RVO Misconception Storage and Lifetime]] | [[20-languages/cpp/conversation-notes/Conversation Note - RVO Basics Return Slot and In-place Analogies]], [[20-languages/cpp/conversation-notes/Conversation Note - RVO Verification Compile Flags and Placement New]] | 專門處理「一個 object lifetime 能不能前半段 RVO 後半段 normal」這個誤解。 |
| [[20-languages/cpp/deep-dives/Deep Dive - Destination First Construction Across RVO Emplace and Placement New]] | [[20-languages/cpp/conversation-notes/Conversation Note - RVO Basics Return Slot and In-place Analogies]], [[20-languages/cpp/conversation-notes/Conversation Note - RVO Verification Compile Flags and Placement New]] | 把 RVO、`emplace`、placement new、`construct_at` 放在 destination-first construction 這個共同模型下。 |
| [[20-languages/cpp/deep-dives/Deep Dive - Factory Lambda Is Not Syntax Sugar]] | [[20-languages/cpp/conversation-notes/Conversation Note - Factory Lambda and Delayed Construction]] | 原始對話在 factory lambda 追問很多，核心不是語法糖，而是 materialization timing 與 final storage。 |
| [[20-languages/cpp/deep-dives/Deep Dive - std move xvalue and Return Path Selection]] | [[20-languages/cpp/conversation-notes/Conversation Note - Move Ownership and Value Categories]] | 把 `std::move`、xvalue、return path selection、NRVO fallback 拆出來，避免被 move ownership 混在一起。 |
| [[20-languages/cpp/deep-dives/Deep Dive - Move Ownership Matrix4x4 and Resource Transfer]] | [[20-languages/cpp/conversation-notes/Conversation Note - Move Ownership and Value Categories]] | 對應 Matrix4x4、dynamic array、ownership transfer、move 為什麼存在這條長追問。 |
| [[20-languages/cpp/deep-dives/Deep Dive - Value Categories Beyond std move]] | [[20-languages/cpp/conversation-notes/Conversation Note - Move Ownership and Value Categories]] | 原始對話另外深入問 `lvalue`、`xvalue`、`prvalue`、`T&&` return，不應只附在 `std::move` 裡。 |
| [[20-languages/cpp/deep-dives/Deep Dive - Buffer Bug RAII Rules and noexcept Move]] | [[20-languages/cpp/conversation-notes/Conversation Note - Report Topic Object Delivery RAII and Object Not Data]] | Buffer bug、RAII、Rule of 0/3/5、`noexcept` move 是 resource safety 主線，和報告 thesis 相關但應獨立。 |
| [[20-languages/cpp/deep-dives/Deep Dive - Object Not Just Data Report Thesis]] | [[20-languages/cpp/conversation-notes/Conversation Note - Report Topic Object Delivery RAII and Object Not Data]] | 對應報告題目從 RVO 轉向 `object is not just data` / `object delivery` 的過程。 |
| [[20-languages/cpp/deep-dives/Deep Dive - C Convention to Cpp Semantic Lifting]] | [[20-languages/cpp/conversation-notes/Conversation Note - C vs C++ Semantic Lifting and Hidden Semantics]] | 對應「C 明明也能做，為什麼 C++ 要這麼複雜」與 C convention 被提升成 type semantics。 |
| [[20-languages/cpp/deep-dives/Deep Dive - Type Operations Laws Invariants and Stepanov]] | [[20-languages/cpp/conversation-notes/Conversation Note - Type Operations Mathematical Analogy and Stepanov]] | 對應 operation / invariant / law 的數學類比，以及 Stepanov generic programming 脈絡。 |
| [[20-languages/cpp/deep-dives/Deep Dive - Ownership Design Space Cpp Rust GC]] | [[20-languages/cpp/conversation-notes/Conversation Note - C vs C++ Semantic Lifting and Hidden Semantics]], [[20-languages/cpp/conversation-notes/Conversation Note - Rust Comparison and CppCon Watchlist]] | 對應 C++、Rust、GC 在 ownership / lifetime / resource policy 上的設計空間比較。 |
| [[20-languages/cpp/deep-dives/Deep Dive - Cpp Object Resource Skill Tree and Thinking Method]] | [[20-languages/cpp/conversation-notes/Conversation Note - Knowledge Tree and C++ Theme Expansion]], [[20-languages/cpp/conversation-notes/Conversation Note - Thinking Method Self Analysis and Expert Viewpoints]] | 對應最後從單點技術抽成 C++ object/resource skill tree 與 layer decomposition method。 |

## Notes Kept As Primary Context

這些 Conversation Notes 不是被 Deep Dive 取代，而是保留為 primary context：

- [[20-languages/cpp/conversation-notes/Conversation Note - RVO Basics Return Slot and In-place Analogies]]
- [[20-languages/cpp/conversation-notes/Conversation Note - Factory Lambda and Delayed Construction]]
- [[20-languages/cpp/conversation-notes/Conversation Note - RVO Verification Compile Flags and Placement New]]
- [[20-languages/cpp/conversation-notes/Conversation Note - Move Ownership and Value Categories]]
- [[20-languages/cpp/conversation-notes/Conversation Note - Report Topic Object Delivery RAII and Object Not Data]]
- [[20-languages/cpp/conversation-notes/Conversation Note - C vs C++ Semantic Lifting and Hidden Semantics]]
- [[20-languages/cpp/conversation-notes/Conversation Note - Type Operations Mathematical Analogy and Stepanov]]
- [[20-languages/cpp/conversation-notes/Conversation Note - Rust Comparison and CppCon Watchlist]]
- [[20-languages/cpp/conversation-notes/Conversation Note - Knowledge Tree and C++ Theme Expansion]]
- [[20-languages/cpp/conversation-notes/Conversation Note - Thinking Method Self Analysis and Expert Viewpoints]]

## Next Phase

下一步應重新抽取 Concept notes。Concept notes 應該是新 Deep Dive 的 lookup layer，而不是原始對話內容的替代品。
