# Concept - C++ Type as Operations and Invariants

## Derived From

- Conversation Note: [[20-languages/cpp/conversation-notes/Conversation Note - Type Operations Mathematical Analogy and Stepanov]]
- Deep Dive: [[20-languages/cpp/deep-dives/Deep Dive - Type Operations Laws Invariants and Stepanov]]
- Deep Dive: [[20-languages/cpp/deep-dives/Deep Dive - C Convention to Cpp Semantic Lifting]]

## Problem

一個 C++ type 不只是 data layout。對非 trivial type 來說，更重要的是：它允許哪些 operation，這些 operation 是否保持 invariant，以及成本模型是否符合預期。

## Original Context To Keep

原始對話先問「有資源的 object 可不可複製，是不是跟線性代數、離散數學類似」，再追到抽象代數、Stepanov、generic programming。

這個 Concept 要保留這個限制：數學類比有用，但不能把所有 C++ object semantics 都硬說成群、環、體。

## Why Naive Fails

naive view：

```text
type = memory layout
```

這對 `int` 這類 trivial type 尚可，但對 resource-owning type 不夠。

`Buffer` 的本質不是只有：

```text
char* ptr
size_t size
```

還包含：

- 哪些 state 是 valid。
- copy 是否存在。
- move 是否 transfer ownership。
- destructor 是否會 free。
- operation 後 invariant 是否保持。

## Mental Model

```text
Type = valid states + operations + invariants + cost model
```

例如 `Buffer`：

- valid states：`ptr == nullptr` 或 `ptr` 指向一塊由此 object 擁有的 buffer。
- operations：construct、destroy、copy、move、assign。
- invariants：同一塊 owned buffer 不能被兩個 owner 重複 delete。
- cost model：copy 可能 O(n)，move 應該 O(1)。

## Details

有些 operation 需要明確定義：

```cpp
Buffer(const Buffer&);            // deep copy?
Buffer(Buffer&&) noexcept;        // ownership transfer?
Buffer& operator=(const Buffer&); // release old then copy?
```

有些 operation 應該禁止：

```cpp
Buffer(const Buffer&) = delete;
Buffer& operator=(const Buffer&) = delete;
```

這等於說：這個 type 的 operation set 裡沒有 copy。

## Relation To Math Analogy

抽象代數的有用點：

- 不是只有 element set，還要有 operations。
- operations 有定義域，不是所有操作都存在。
- operations 要保持 structure / invariant。
- 不同 representation 可以代表同一 abstract value。

限制：

- C++ object 還有 lifetime、mutation、resource ownership、exception safety、cost model。
- 不是每個 type 都要對應到標準代數結構。

## Stepanov Connection

generic programming 不只是 template 能吃任何 type，而是 algorithm 要求 type 滿足某組 operation laws。

例如 sort 需要 ordering relation；container algorithm 需要 iterator category；regular / semiregular type 有一組可複製、可賦值、可比較的行為期待。

`semiregular` / `regular` 可以當成 calibrated vocabulary：

- `semiregular`：像普通值一樣可 default construct、copy、assign。
- `regular`：再加上 equality comparison，接近 built-in value type 的行為期待。

但 resource owner 不一定要 regular。`unique_ptr` 類型的語義是 unique ownership，因此 copy operation 不存在反而是正確設計。

## Verification

整理一個 type 時，逐項問：

- constructor 是否建立 valid state？
- destructor 是否合法結束 lifetime？
- copy 是否產生獨立 valid object？
- move 是否讓 destination 接手，source 仍 valid？
- assignment 是否處理 self-assignment / exception safety？
- operation 後 invariant 是否仍成立？

## Links

- [[20-languages/cpp/object-resource-semantics/Concept - C vs C++ Semantic Lifting]]
- [[20-languages/cpp/object-resource-semantics/Concept - RAII and Rule of 0 3 5]]
- [[20-languages/cpp/object-resource-semantics/Concept - Move Semantics and Ownership]]

## Source

- [[20-languages/cpp/Source Map - ChatGPT CPP RVO 解釋|Source Map]]: source lines `25572-26750`, `29595-32623`, `33390-33800`
