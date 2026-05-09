# Thinking Method - Layered Semantic Decomposition

## Source

- [[20-languages/cpp/Question Trail - ChatGPT CPP RVO 解釋]]
- [[20-languages/cpp/conversation-notes/Conversation Note - Knowledge Tree and C++ Theme Expansion]]
- [[20-languages/cpp/conversation-notes/Conversation Note - Thinking Method Self Analysis and Expert Viewpoints]]
- [[20-languages/cpp/deep-dives/Deep Dive - Cpp Object Resource Skill Tree and Thinking Method]]

## Original Context To Keep

這篇方法論不是事後硬整理出來的抽象規則，而是從一整串 RVO 對話自然浮出的問題模式：

- 從 `return by value` 這個小點往下追 stack / storage / lifetime。
- 再往上追 ownership / invariant / object delivery。
- 最後比較 C、C++、Rust、GC、ABI、graphics 類比。

保留這個來源很重要，否則這篇會變成泛用的「如何思考」口號。

## What It Is

Layered semantic decomposition 是一種從具體技術問題往下拆 execution，再往上抽 semantic / invariant / design problem 的思考方式。

它適合用在 C++、graphics pipeline、systems、compiler、OS、hardware 等領域。

## Core Pattern

從一個具體 case 開始：

```cpp
T x = makeT();
```

不要停在名詞定義，依序問：

1. 表面上發生什麼？
2. naive execution model 是什麼？
3. naive model 哪裡錯？
4. memory / stack / ABI / compiler lowering 實際怎麼做？
5. language object model 如何規範？
6. ownership / lifetime / invariant 是什麼？
7. cost model 是什麼？
8. 相鄰系統怎麼處理同一問題？
9. 上位問題是什麼？

## Six Layers

### Layer 1 - Phenomenon

表面現象：

- `return by value`
- `std::move`
- `emplace_back`
- `vector reallocation`

### Layer 2 - Execution

它在底層怎麼運作？

- stack frame
- heap allocation
- hidden return slot
- register / memory
- constructor / destructor call

### Layer 3 - Language Semantics

C++ object model 怎麼規定它？

- storage vs object
- lifetime begin / end
- value category
- copy / move constructor
- materialization

### Layer 4 - Invariants

什麼規則必須保持？

- one owner
- source moved-from still valid
- destructor safe
- no dangling reference
- operation preserves valid state

### Layer 5 - Design Space

為什麼語言 / library 要這樣設計？

- zero-overhead abstraction
- resource safety
- API clarity
- exception safety
- compile-time enforcement vs programmer convention

### Layer 6 - Adjacent Systems

同一問題在別處怎麼出現？

- C: manual convention
- Rust: ownership / borrow checker
- GC language: runtime reachability
- graphics: render directly to target
- OS / ABI: return slot and calling convention

## Stop Conditions

這種思考方式的風險是一直往上抽象。需要停止條件：

- 能解釋原始 case。
- 能預測新 case。
- 能寫出可驗證 command / test / small program。
- 能告訴別人何時該用、何時不該用。
- 能回到實作決策。

## Use As Note Template

整理技術筆記時可以用：

```text
Phenomenon:
Naive model:
Mechanism:
Semantic rule:
Invariant:
Cost model:
Adjacent concepts:
Verification:
```

## Relation To Existing Vault Rules

這是 [[_meta/Writing Principles|Writing Principles]] 裡「naive 方法失敗 -> operational model -> math/spec -> implementation -> verification」的通用化版本。

## Source

- [[20-languages/cpp/Source Map - ChatGPT CPP RVO 解釋|Source Map]]: source lines `38026-40145`, `40442-41104`
