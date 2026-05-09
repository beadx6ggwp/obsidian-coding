# Concept Audit - ChatGPT CPP RVO 解釋

## Purpose

這份 audit 用來處理一個問題：目前 `Concept` notes 有些是先前直接壓縮整理出來的，可能太像二手定義卡片。後續要把它們重新對齊到 `Conversation Note` 和校準後的 `Deep Dive`。

原則：

- `Conversation Note` 是 primary source，保留原始追問與上下文。
- `Deep Dive` 是由 Conversation Note 重寫出的推理文章，保留 naive model、卡住點、修正模型。
- `Concept` 只作為 quick lookup card，不能取代前兩者。
- 每個 Concept 都應該知道自己從哪個 Conversation Note / Deep Dive 抽出。

## Current Status

2026-05-09 update:

- All current Deep Dive notes were rebuilt from Conversation Notes, not merely patched from the earlier Concept-derived drafts.
- The old 7 Deep Dive files were removed; the active Deep Dive layer now has 13 regenerated files.
- Each rebuilt Deep Dive includes `Rebuild Source Rule`, `Original Questions`, `Conversation Reconstruction`, and `Source`; source-backed notes also include `External Source Check` / final mental model sections where appropriate.
- The rebuild map is tracked in [[20-languages/cpp/Deep Dive Rebuild Map - ChatGPT CPP RVO 解釋]].
- External source alignment is tracked in [[20-languages/cpp/External Sources - C++ Object Semantics]].
- Concept notes listed below were re-extracted from the rebuilt Deep Dives, not from the original compressed concept cards.
- Concept notes should still preserve their `Conversation Note` provenance so they do not become generic cppreference summaries.

## Audit Table

| Concept | Primary Conversation Source | Deep Dive Source | Status / Next Action |
| --- | --- | --- | --- |
| [[20-languages/cpp/object-resource-semantics/Concept - Object Delivery - Copy Move In-place Construction]] | [[20-languages/cpp/conversation-notes/Conversation Note - Report Topic Object Delivery RAII and Object Not Data]] | [[20-languages/cpp/deep-dives/Deep Dive - Object Not Just Data Report Thesis]], [[20-languages/cpp/deep-dives/Deep Dive - Destination First Construction Across RVO Emplace and Placement New]] | Re-extracted. |
| [[20-languages/cpp/object-resource-semantics/Concept - RVO and NRVO]] | [[20-languages/cpp/conversation-notes/Conversation Note - RVO Basics Return Slot and In-place Analogies]] | [[20-languages/cpp/deep-dives/Deep Dive - RVO Return Slot and Naive Copy Model]], [[20-languages/cpp/deep-dives/Deep Dive - Half RVO Misconception Storage and Lifetime]] | Re-extracted. |
| [[20-languages/cpp/object-resource-semantics/Concept - Copy Elision and C++17 prvalue]] | [[20-languages/cpp/conversation-notes/Conversation Note - RVO Basics Return Slot and In-place Analogies]], [[20-languages/cpp/conversation-notes/Conversation Note - Move Ownership and Value Categories]] | [[20-languages/cpp/deep-dives/Deep Dive - RVO Return Slot and Naive Copy Model]], [[20-languages/cpp/deep-dives/Deep Dive - std move xvalue and Return Path Selection]] | Re-extracted. |
| [[20-languages/cpp/object-resource-semantics/Concept - Value Categories - lvalue xvalue prvalue]] | [[20-languages/cpp/conversation-notes/Conversation Note - Move Ownership and Value Categories]] | [[20-languages/cpp/deep-dives/Deep Dive - Value Categories Beyond std move]] | Re-extracted. |
| [[20-languages/cpp/object-resource-semantics/Concept - std move vs Move Constructor]] | [[20-languages/cpp/conversation-notes/Conversation Note - Move Ownership and Value Categories]], [[20-languages/cpp/conversation-notes/Conversation Note - C vs C++ Semantic Lifting and Hidden Semantics]] | [[20-languages/cpp/deep-dives/Deep Dive - std move xvalue and Return Path Selection]] | Re-extracted. |
| [[20-languages/cpp/object-resource-semantics/Concept - Move Semantics and Ownership]] | [[20-languages/cpp/conversation-notes/Conversation Note - Move Ownership and Value Categories]] | [[20-languages/cpp/deep-dives/Deep Dive - Move Ownership Matrix4x4 and Resource Transfer]] | Re-extracted. |
| [[20-languages/cpp/object-resource-semantics/Concept - Storage vs Object Lifetime]] | [[20-languages/cpp/conversation-notes/Conversation Note - RVO Basics Return Slot and In-place Analogies]], [[20-languages/cpp/conversation-notes/Conversation Note - RVO Verification Compile Flags and Placement New]] | [[20-languages/cpp/deep-dives/Deep Dive - Half RVO Misconception Storage and Lifetime]] | Re-extracted. |
| [[20-languages/cpp/object-resource-semantics/Concept - Placement New and construct_at]] | [[20-languages/cpp/conversation-notes/Conversation Note - RVO Verification Compile Flags and Placement New]] | [[20-languages/cpp/deep-dives/Deep Dive - Destination First Construction Across RVO Emplace and Placement New]] | Re-extracted. |
| [[20-languages/cpp/object-resource-semantics/Concept - In-place Construction and emplace]] | [[20-languages/cpp/conversation-notes/Conversation Note - RVO Basics Return Slot and In-place Analogies]] | [[20-languages/cpp/deep-dives/Deep Dive - Destination First Construction Across RVO Emplace and Placement New]] | Re-extracted. |
| [[20-languages/cpp/object-resource-semantics/Concept - Factory Lambda and Delayed Construction]] | [[20-languages/cpp/conversation-notes/Conversation Note - Factory Lambda and Delayed Construction]] | [[20-languages/cpp/deep-dives/Deep Dive - Factory Lambda Is Not Syntax Sugar]] | Re-extracted. |
| [[20-languages/cpp/object-resource-semantics/Concept - RAII and Rule of 0 3 5]] | [[20-languages/cpp/conversation-notes/Conversation Note - Report Topic Object Delivery RAII and Object Not Data]] | [[20-languages/cpp/deep-dives/Deep Dive - Buffer Bug RAII Rules and noexcept Move]] | Re-extracted. |
| [[20-languages/cpp/object-resource-semantics/Concept - noexcept Move and Container Reallocation]] | [[20-languages/cpp/conversation-notes/Conversation Note - Report Topic Object Delivery RAII and Object Not Data]] | [[20-languages/cpp/deep-dives/Deep Dive - Buffer Bug RAII Rules and noexcept Move]] | Re-extracted. |
| [[20-languages/cpp/object-resource-semantics/Concept - C vs C++ Semantic Lifting]] | [[20-languages/cpp/conversation-notes/Conversation Note - C vs C++ Semantic Lifting and Hidden Semantics]] | [[20-languages/cpp/deep-dives/Deep Dive - C Convention to Cpp Semantic Lifting]] | Re-extracted. |
| [[20-languages/cpp/object-resource-semantics/Concept - C++ Type as Operations and Invariants]] | [[20-languages/cpp/conversation-notes/Conversation Note - Type Operations Mathematical Analogy and Stepanov]] | [[20-languages/cpp/deep-dives/Deep Dive - Type Operations Laws Invariants and Stepanov]] | Re-extracted. |
| [[20-languages/rust/Concept - Rust Ownership Compared With C++ Move]] | [[20-languages/cpp/conversation-notes/Conversation Note - Rust Comparison and CppCon Watchlist]], [[20-languages/cpp/conversation-notes/Conversation Note - C vs C++ Semantic Lifting and Hidden Semantics]] | [[20-languages/cpp/deep-dives/Deep Dive - Ownership Design Space Cpp Rust GC]] | Re-extracted. |
| [[30-systems/abi-function-call/Concept - Function Call Stack and Return Object Storage]] | [[20-languages/cpp/conversation-notes/Conversation Note - RVO Basics Return Slot and In-place Analogies]], [[20-languages/cpp/conversation-notes/Conversation Note - Type Operations Mathematical Analogy and Stepanov]] | [[20-languages/cpp/deep-dives/Deep Dive - Half RVO Misconception Storage and Lifetime]] | Re-extracted. |
| [[_meta/thinking-methods/Thinking Method - Layered Semantic Decomposition]] | [[20-languages/cpp/conversation-notes/Conversation Note - Knowledge Tree and C++ Theme Expansion]], [[20-languages/cpp/conversation-notes/Conversation Note - Thinking Method Self Analysis and Expert Viewpoints]] | [[20-languages/cpp/deep-dives/Deep Dive - Cpp Object Resource Skill Tree and Thinking Method]] | Re-extracted as method note. |

## Re-extraction Order

1. Re-extract Concept notes from calibrated Deep Dives. Done.
2. Update each concept's `Derived From` section. Done.
3. Compress wording only after source alignment. Done for the active Concept set; future concept additions should follow the same order.

## Done Definition

A Concept note is considered recalibrated when it has:

- `Derived From` links to at least one Conversation Note and, when available, one Deep Dive.
- A short `Original Context To Keep` section.
- A `Why Naive Fails` section that preserves the original misconception or question.
- No claim that erases the nuance preserved in the Conversation Note.
