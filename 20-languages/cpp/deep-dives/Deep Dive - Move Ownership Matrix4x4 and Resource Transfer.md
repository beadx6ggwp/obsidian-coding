# Deep Dive - Move Ownership Matrix4x4 and Resource Transfer

## Derived From Conversation Notes

- [[20-languages/cpp/conversation-notes/Conversation Note - Move Ownership and Value Categories]]
- [[20-languages/cpp/conversation-notes/Conversation Note - Report Topic Object Delivery RAII and Object Not Data]]

## Rebuild Source Rule

這篇從 Matrix4x4 和 ownership 追問生成。它不再被塞進 `return std::move` 一篇裡，因為原始對話對 move 的深入很大一段是在問「move 到底搬什麼」。

## Original Questions

- source line `6743`: `Matrix4x4 Calculate() { Matrix4x4 mat; return std::move(mat); }...為什麼move copy一樣? 64bytes可能是什麼?`
- source line `7732`: `MOVE 對於一維陣列怎麼操作`
- source line `8022`: `那move的ownership又是什麼`
- source line `8434`: `move這樣設計的意義是什麼 當初為什麼要設計這個架構`

## Conversation Reconstruction

```text
起點：
    return std::move 之後，你追問 Matrix4x4 為什麼 move/copy 可能一樣。

修正：
    move 不等於比較快；對 inline value data 沒有外部 resource 可偷。

收斂：
    move 的核心是 ownership/resource transfer，而不是 byte-copy speed。
```

## Matrix4x4 Counterexample

```cpp
struct Matrix4x4 {
    float m[16];
};
```

```text
16 * sizeof(float) = 64 bytes
```

這種 type 通常沒有：

- heap buffer
- file handle
- GPU handle
- unique ownership

因此 move 也只能把 inline bytes 轉到另一個 object。它不會突然變成 O(1) resource steal。

## Resource Owner Case

```cpp
struct Buffer {
    char* ptr{};
    size_t size{};

    Buffer(Buffer&& other) noexcept
        : ptr(other.ptr), size(other.size) {
        other.ptr = nullptr;
        other.size = 0;
    }
};
```

這裡 move 是：

```text
destination takes ownership
source remains valid moved-from object
both destructors stay safe
```

## Moved-From Does Not Mean Original Value Is Still Usable

`source remains valid` 的意思不是「source 還保有原本值」。

對 standard library types，常見規則是 valid but unspecified state：

- destructor 可以安全執行。
- assignment 這類沒有額外 precondition 的操作可以安全執行。
- 有 precondition 的操作仍然要先檢查，例如 `str.back()` 前要確認 `!str.empty()`。

對 user-defined resource owner，move constructor 的責任是把 source 放回一個 invariant 成立、destructor safe 的狀態。這個狀態可以是 empty、null handle、zero size，或其他 type 明確定義的 moved-from state。

## Why Move Exists

C++03 的問題：

```text
resource-owning values are expensive or impossible to copy
but they still need to be returned, stored, relocated, and transferred
```

Move semantics 讓：

- `unique_ptr` 這種 non-copyable type 可以自然轉移。
- `vector` reallocation 可以 move elements。
- function 可以 return resource owner。
- ownership transfer 變成 type-defined operation。

## External Source Check

- [cppreference - move constructor](https://en.cppreference.com/w/cpp/language/move_constructor): move constructor semantics.
- [cppreference - std::move](https://en.cppreference.com/w/cpp/utility/move): moved-from standard library objects are valid but unspecified unless otherwise specified.
- [cppreference - std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr): move-only ownership wrapper.
- [C++ Core Guidelines - C.66](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines?lang=en#Rc-move-noexcept): move operations should be noexcept.

## Final Mental Model

```text
Move is meaningful when there is resource ownership to transfer.
For pure inline value data, move may be no better than copy.
```

## Source

- [[20-languages/cpp/conversation-notes/Conversation Note - Move Ownership and Value Categories]]
- [[20-languages/cpp/Source Map - ChatGPT CPP RVO 解釋|Source Map]]: source lines `6743-8890`
