# Concept - Rust Ownership Compared With C++ Move

## Derived From

- Conversation Note: [[20-languages/cpp/conversation-notes/Conversation Note - Rust Comparison and CppCon Watchlist]]
- Conversation Note: [[20-languages/cpp/conversation-notes/Conversation Note - C vs C++ Semantic Lifting and Hidden Semantics]]
- Deep Dive: [[20-languages/cpp/deep-dives/Deep Dive - Ownership Design Space Cpp Rust GC]]

## Problem

從 C++ move semantics 看 Rust，很容易說成：「Rust 只是把 C++ ownership / lifetime 規則用 compiler 管得更嚴。」這方向有一部分對，但不完整。

## Original Context To Keep

原始對話是在 C++ object/resource semantics 之後問：

```text
那麼 RUST 就是在 CPP 之上繼續解決嗎？
Rust 更加嚴格地把這種思想規則用 compiler 做規範？
```

這個 Concept 要保留比較視角：不是語言排名，而是同一個 ownership / lifetime 問題在不同 design point 的解法。

## Why Naive Fails

naive view：

```text
Rust = C++ move + stricter compiler
```

這會漏掉 Rust 的 API 設計方式也不同。Rust 不只是檢查 C++ 原本的規則，而是把 ownership、borrow、lifetime 放進語言核心，讓 function signature 就表達 aliasing / mutation discipline。

## Shared Question

C++ move 和 Rust ownership 都在回答：

```text
resource / object 在程式中流動時：
誰擁有？
誰可以讀？
誰可以寫？
什麼時候釋放？
move 之後 source 還能不能用？
```

## C++ Move

```cpp
T b = std::move(a);
```

核心：

- `std::move(a)` 把 expression 轉成 xvalue。
- move constructor 定義如何接手 resource。
- source object 仍然存在，必須是 valid moved-from state。
- compiler 不會全面禁止你之後使用 moved-from object。

C++ 保留很高自由度，但很多 invariant 要靠 type author 和 programmer 自己守。

## Rust Move

```rust
let b = a;
// a is no longer usable if T is not Copy
```

核心：

- ownership 是 compiler-tracked rule。
- move 後原 binding 通常不能再使用。
- borrowing rule 限制 aliasing：多個 shared borrow 或一個 mutable borrow。
- lifetime / borrow checker 把很多 dangling / use-after-move 問題提前變成 compile error。

## Key Difference

```text
C++:
    move means type may transfer resource;
    source remains valid moved-from object.

Rust:
    move means ownership transfer;
    old binding is no longer usable.
```

C++ 的 move constructor 是可自定義 operation。Rust move 通常更像 ownership bookkeeping + value relocation，沒有 C++ 那種 user-defined move constructor 模型。

## API Surface Difference

Rust 的差異不只在「move 後不能用」。更大的差異是 function signature 直接表達 ownership / borrowing：

```rust
fn take(x: T)        // takes ownership
fn read(x: &T)       // shared borrow
fn write(x: &mut T)  // exclusive mutable borrow
```

C++ 可以用 `T` / `const T&` / `T&` / `T&&` 表達類似意圖，但 compiler 不會用 borrow checker 全面追蹤 aliasing 和 mutation discipline：

```cpp
void take(T x);
void read(const T& x);
void write(T& x);
void consume(T&& x);
```

這就是為什麼「Rust = C++ move 加嚴格檢查」不夠精準。Rust 把 ownership / borrow discipline 變成 API design 的主體；C++ 則把很多規則留在 RAII type、library convention、review discipline 和 static analysis 裡。

## Verification

比較同一個 resource owner API：

- C++：是否 delete copy、定義 move、moved-from 是否 valid。
- Rust：是否實作 `Clone`、是否 `Copy`、borrow 是否允許 aliasing、move 後原 binding 是否能用。

## Links

- [[20-languages/cpp/object-resource-semantics/Concept - Move Semantics and Ownership]]
- [[20-languages/cpp/object-resource-semantics/Concept - C vs C++ Semantic Lifting]]
- [[20-languages/cpp/object-resource-semantics/Concept - C++ Type as Operations and Invariants]]

## Source

- [[20-languages/cpp/Source Map - ChatGPT CPP RVO 解釋|Source Map]]: source lines `33830-34738`
- [Rust Book - Ownership](https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html)
- [Rust Book - References and Borrowing](https://doc.rust-lang.org/book/ch04-02-references-and-borrowing.html)
- [Rust Reference - Pointer Types](https://doc.rust-lang.org/reference/types/pointer.html)
