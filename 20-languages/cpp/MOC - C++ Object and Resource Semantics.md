# MOC - C++ Object and Resource Semantics

這組筆記從 `RVO` 開始，但核心不是單一最佳化技巧，而是 C++ 如何處理 object / resource 在程式中的建立、交付、複製、移動、擁有與銷毀。

## Source

- [[00-inbox/ChatGPT-CPP RVO 解釋|ChatGPT-CPP RVO 解釋]]
- [[Source Map - ChatGPT CPP RVO 解釋|Source Map - ChatGPT CPP RVO 解釋]]
- [[20-languages/cpp/Question Trail - ChatGPT CPP RVO 解釋|Question Trail - ChatGPT CPP RVO 解釋]]
- [[20-languages/cpp/Deep Dive Rebuild Map - ChatGPT CPP RVO 解釋|Deep Dive Rebuild Map - ChatGPT CPP RVO 解釋]]
- [[20-languages/cpp/External Sources - C++ Object Semantics|External Sources - C++ Object Semantics]]
- [[20-languages/cpp/Knowledge Audit - C++ Object and Resource Semantics|Knowledge Audit - C++ Object and Resource Semantics]]
- [[20-languages/cpp/Teaching Map - C++ Object and Resource Semantics|Teaching Map - C++ Object and Resource Semantics]]
- [[20-languages/cpp/Learning Path - C++ Object and Resource Semantics|Learning Path - C++ Object and Resource Semantics]]
- [[20-languages/cpp/Teaching Notes - C++ Object and Resource Semantics|Teaching Notes - C++ Object and Resource Semantics]]

## Core Question

當一個 function、container、factory 或 system component 要把一個 `T` 交給另一個地方時，C++ 到底有哪些策略？

- Copy：A 有一份，B 也要一份。
- Move：A 不再擁有，B 接手資源。
- In-place construction：不要先在 A 建，再搬到 B；直接在 B 的 final storage 開始 object lifetime。

核心不是「RVO 是 compiler trick」或「到處加 `std::move`」。更精準的切面是：

```text
RVO / copy elision:
    避免搬移，讓 T 直接在 B 的 final storage 出生。

move:
    搬移不可避免、或 source object 已經存在且 A 不再需要 ownership 時，
    用便宜的 ownership transfer 把 resource 交給 B。

modern C++ design pressure:
    讓 object 盡量直接出生在它最後要待的位置。
```

> [!note] Visual overview
> 這張總覽圖可以當成整組筆記的入口圖。完整圖片配置規劃見 [[20-languages/cpp/Image Map - ChatGPT CPP RVO 解釋|Image Map - ChatGPT CPP RVO 解釋]]。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_22_10.png]]

## Question Trail

如果要回看這組筆記是怎麼從原始對話長出來的，先讀：

- [[20-languages/cpp/Question Trail - ChatGPT CPP RVO 解釋|Question Trail - ChatGPT CPP RVO 解釋]]

這篇保留代表性原始提問、每個思考轉折，以及哪些問題最後被抽成 canonical notes。

## Conversation Notes

Conversation Notes 負責保留某一大段對話的原始追問順序與展開細節。它比 `Concept` 更完整，也比 `Deep Dive` 更貼近原始對話。

- [[20-languages/cpp/conversation-notes/Conversation Note - RVO Basics Return Slot and In-place Analogies|Conversation Note - RVO Basics Return Slot and In-place Analogies]]
- [[20-languages/cpp/conversation-notes/Conversation Note - Move Ownership and Value Categories|Conversation Note - Move Ownership and Value Categories]]
- [[20-languages/cpp/conversation-notes/Conversation Note - Factory Lambda and Delayed Construction|Conversation Note - Factory Lambda and Delayed Construction]]
- [[20-languages/cpp/conversation-notes/Conversation Note - RVO Verification Compile Flags and Placement New|Conversation Note - RVO Verification Compile Flags and Placement New]]
- [[20-languages/cpp/conversation-notes/Conversation Note - Report Topic Object Delivery RAII and Object Not Data|Conversation Note - Report Topic Object Delivery RAII and Object Not Data]]
- [[20-languages/cpp/conversation-notes/Conversation Note - C vs C++ Semantic Lifting and Hidden Semantics|Conversation Note - C vs C++ Semantic Lifting and Hidden Semantics]]
- [[20-languages/cpp/conversation-notes/Conversation Note - Type Operations Mathematical Analogy and Stepanov|Conversation Note - Type Operations Mathematical Analogy and Stepanov]]
- [[20-languages/cpp/conversation-notes/Conversation Note - Rust Comparison and CppCon Watchlist|Conversation Note - Rust Comparison and CppCon Watchlist]]
- [[20-languages/cpp/conversation-notes/Conversation Note - Knowledge Tree and C++ Theme Expansion|Conversation Note - Knowledge Tree and C++ Theme Expansion]]
- [[20-languages/cpp/conversation-notes/Conversation Note - Thinking Method Self Analysis and Expert Viewpoints|Conversation Note - Thinking Method Self Analysis and Expert Viewpoints]]

## Deep Dives

Deep Dive notes 負責把 Conversation Note 裡的某條思考線重寫成完整推理文章：原始問題、對話推進、naive model、卡住點、修正模型、外部來源校準、概念延伸。這一層已依 [[20-languages/cpp/Deep Dive Rebuild Map - ChatGPT CPP RVO 解釋|Deep Dive Rebuild Map]] 完全移除舊七篇，重新從 Conversation Notes 生成。

Deep Dive / Concept 進入教學化之前，先用 [[20-languages/cpp/Knowledge Audit - C++ Object and Resource Semantics|Knowledge Audit]] 校準哪些說法是正確、哪些只是 mental model、哪些需要修正或延伸。

RVO / storage / in-place construction:

- [[20-languages/cpp/deep-dives/Deep Dive - RVO Return Slot and Naive Copy Model|Deep Dive - RVO Return Slot and Naive Copy Model]]
- [[20-languages/cpp/deep-dives/Deep Dive - Half RVO Misconception Storage and Lifetime|Deep Dive - Half RVO Misconception Storage and Lifetime]]
- [[20-languages/cpp/deep-dives/Deep Dive - Destination First Construction Across RVO Emplace and Placement New|Deep Dive - Destination First Construction Across RVO Emplace and Placement New]]
- [[20-languages/cpp/deep-dives/Deep Dive - Factory Lambda Is Not Syntax Sugar|Deep Dive - Factory Lambda Is Not Syntax Sugar]]

Move / ownership / value categories:

- [[20-languages/cpp/deep-dives/Deep Dive - std move xvalue and Return Path Selection|Deep Dive - std move xvalue and Return Path Selection]]
- [[20-languages/cpp/deep-dives/Deep Dive - Move Ownership Matrix4x4 and Resource Transfer|Deep Dive - Move Ownership Matrix4x4 and Resource Transfer]]
- [[20-languages/cpp/deep-dives/Deep Dive - Value Categories Beyond std move|Deep Dive - Value Categories Beyond std move]]

Object / resource semantics:

- [[20-languages/cpp/deep-dives/Deep Dive - Buffer Bug RAII Rules and noexcept Move|Deep Dive - Buffer Bug RAII Rules and noexcept Move]]
- [[20-languages/cpp/deep-dives/Deep Dive - Object Not Just Data Report Thesis|Deep Dive - Object Not Just Data Report Thesis]]
- [[20-languages/cpp/deep-dives/Deep Dive - C Convention to Cpp Semantic Lifting|Deep Dive - C Convention to Cpp Semantic Lifting]]
- [[20-languages/cpp/deep-dives/Deep Dive - Type Operations Laws Invariants and Stepanov|Deep Dive - Type Operations Laws Invariants and Stepanov]]

Design space / meta:

- [[20-languages/cpp/deep-dives/Deep Dive - Ownership Design Space Cpp Rust GC|Deep Dive - Ownership Design Space Cpp Rust GC]]
- [[20-languages/cpp/deep-dives/Deep Dive - Cpp Object Resource Skill Tree and Thinking Method|Deep Dive - Cpp Object Resource Skill Tree and Thinking Method]]

## Core Notes

Concept notes are quick lookup cards. They should be read after the relevant Conversation Note / Deep Dive when context matters. Re-extraction status is tracked in [[20-languages/cpp/Concept Audit - ChatGPT CPP RVO 解釋|Concept Audit - ChatGPT CPP RVO 解釋]].

- [[20-languages/cpp/object-resource-semantics/Concept - Object Delivery - Copy Move In-place Construction|Concept - Object Delivery - Copy Move In-place Construction]]
- [[20-languages/cpp/object-resource-semantics/Concept - RVO and NRVO|Concept - RVO and NRVO]]
- [[20-languages/cpp/object-resource-semantics/Concept - Copy Elision and C++17 prvalue|Concept - Copy Elision and C++17 prvalue]]
- [[20-languages/cpp/object-resource-semantics/Concept - Value Categories - lvalue xvalue prvalue|Concept - Value Categories - lvalue xvalue prvalue]]
- [[20-languages/cpp/object-resource-semantics/Concept - std move vs Move Constructor|Concept - std move vs Move Constructor]]
- [[20-languages/cpp/object-resource-semantics/Concept - Move Semantics and Ownership|Concept - Move Semantics and Ownership]]

## Lifetime And Construction

- [[20-languages/cpp/object-resource-semantics/Concept - Storage vs Object Lifetime|Concept - Storage vs Object Lifetime]]
- [[20-languages/cpp/object-resource-semantics/Concept - Placement New and construct_at|Concept - Placement New and construct_at]]
- [[20-languages/cpp/object-resource-semantics/Concept - In-place Construction and emplace|Concept - In-place Construction and emplace]]
- [[20-languages/cpp/object-resource-semantics/Concept - Factory Lambda and Delayed Construction|Concept - Factory Lambda and Delayed Construction]]

## Resource Management And Type Semantics

- [[20-languages/cpp/object-resource-semantics/Concept - RAII and Rule of 0 3 5|Concept - RAII and Rule of 0 3 5]]
- [[20-languages/cpp/object-resource-semantics/Concept - noexcept Move and Container Reallocation|Concept - noexcept Move and Container Reallocation]]
- [[20-languages/cpp/object-resource-semantics/Concept - C vs C++ Semantic Lifting|Concept - C vs C++ Semantic Lifting]]
- [[20-languages/cpp/object-resource-semantics/Concept - C++ Type as Operations and Invariants|Concept - C++ Type as Operations and Invariants]]

## Cross-Area Notes

- [[30-systems/abi-function-call/Concept - Function Call Stack and Return Object Storage|Concept - Function Call Stack and Return Object Storage]]
- [[20-languages/rust/Concept - Rust Ownership Compared With C++ Move|Concept - Rust Ownership Compared With C++ Move]]
- [[90-reading-notes/cpp/Reading - CppCon Watchlist for C++ Object Semantics|Reading - CppCon Watchlist for C++ Object Semantics]]
- [[_meta/thinking-methods/Thinking Method - Layered Semantic Decomposition|Thinking Method - Layered Semantic Decomposition]]

## Calibration Status

- Immediate correctness fixes are tracked in [[20-languages/cpp/Knowledge Audit - C++ Object and Resource Semantics|Knowledge Audit]].
- Teaching architecture is tracked in [[20-languages/cpp/Teaching Map - C++ Object and Resource Semantics|Teaching Map]].
- Readable route is tracked in [[20-languages/cpp/Learning Path - C++ Object and Resource Semantics|Learning Path]].
- Current talk-style teaching decisions are tracked in [[20-languages/cpp/Teaching Notes - C++ Object and Resource Semantics|Teaching Notes]].
- Current extension anchors:
  - [[20-languages/cpp/object-resource-semantics/Concept - noexcept Move and Container Reallocation|noexcept Move and Container Reallocation]]
  - [[20-languages/cpp/deep-dives/Deep Dive - Ownership Design Space Cpp Rust GC|Ownership Design Space Cpp Rust GC]]
  - [[20-languages/cpp/deep-dives/Deep Dive - Type Operations Laws Invariants and Stepanov|Type Operations Laws Invariants and Stepanov]]

## Next Notes To Extract

- `Chapter 1 - I Made A Local And Returned It`
- `Stepanov Generic Programming and Algebraic Structures`
- `C++ Object Resource Semantics Skill Tree`
- `Report Storyboard - C++ Object Not Just Data`

## Reading Direction

如果要重建當初的思考深度，先讀 `Question Trail`，再讀 `Conversation Notes`。`Deep Dives` 是重寫後的主題文章，不應取代原始對話脈絡。

如果只想快速查概念，讀 `Core Notes`，但目前 Concept notes 是下一階段要重新校準的 lookup layer，不應取代 Conversation Notes 或新 Deep Dives。

概念閱讀順序：先讀 `Object Delivery`，再讀 `RVO and NRVO`。如果卡在 `std::move` 為什麼會破壞 NRVO，先補 `Value Categories` 與 `std move vs Move Constructor`。

第二輪可以讀 `Half RVO Misconception` 與 `Destination First Construction`，再回到 `Storage vs Object Lifetime`、`Placement New`、`emplace` 這些 Concept cards。這樣不會把原始對話裡「我是不是想成半個 RVO」和「哪些 case 同構」混成同一個問題。

最後讀 `Buffer Bug RAII`、`Object Not Just Data`、`C Convention to Cpp Semantic Lifting`、`Type Operations Laws Invariants and Stepanov`，它們負責把單一技巧拉回 C++ 的整體 object/resource model。

如果要把它放進更大的 systems 圖像，讀 `Function Call Stack and Return Object Storage`。如果要保留這份對話的提問方法，讀 `Question Trail` 和 `Layered Semantic Decomposition`。
