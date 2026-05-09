# Concept - RAII and Rule of 0 3 5

## Derived From

- Conversation Note: [[20-languages/cpp/conversation-notes/Conversation Note - Report Topic Object Delivery RAII and Object Not Data]]
- Deep Dive: [[20-languages/cpp/deep-dives/Deep Dive - Buffer Bug RAII Rules and noexcept Move]]
- Deep Dive: [[20-languages/cpp/deep-dives/Deep Dive - Object Not Just Data Report Thesis]]

## Problem

如果 object 擁有 resource，constructor / destructor / copy / move 就不是語法細節，而是 resource semantics。

RAII 和 Rule of 0/3/5 是為了避免 resource leak、double free、shallow copy、invalid moved-from state。

## Original Context To Keep

原始對話不是突然列出 Rule of 0/3/5，而是從 Buffer case 和報告主題一路追到「C++ object 不只是 data」。這個 Concept 要保留這個入口：Rule 是被 resource ownership 逼出來的，不是背誦規則。

## Why Naive Fails

naive `Buffer`：

```cpp
struct Buffer {
    char* ptr;
    size_t size;
};
```

如果直接用 compiler-generated copy：

```text
a.ptr == b.ptr
```

兩個 object 可能都以為自己 owns 同一塊 memory，最後 double free。這就是「object 不是只有 data layout」的核心例子。

## Mental Model

RAII:

```text
constructor establishes ownership/invariant
destructor releases resource
copy/move define resource delivery semantics
```

Rule of 3:

如果你需要自訂 destructor、copy constructor、copy assignment，通常三者都要一起考慮。

Rule of 5:

C++11 後再加上 move constructor、move assignment。

Rule of 0:

如果可以，把 resource 交給 `std::vector`、`std::unique_ptr`、`std::string` 這種已經正確管理 resource 的 member，讓 compiler-generated special members 正確工作。

## Details

manual owner 的核心責任：

- destructor 釋放 resource。
- copy 做 deep copy 或直接 delete。
- move transfer ownership。
- moved-from object 保持 valid。
- assignment 處理 self-assignment / exception safety。

大多數高階 code 應優先追求 Rule of 0。

## Implementation

```cpp
class Buffer {
public:
    Buffer(size_t n);
    ~Buffer();

    Buffer(const Buffer&);
    Buffer& operator=(const Buffer&);

    Buffer(Buffer&&) noexcept;
    Buffer& operator=(Buffer&&) noexcept;

private:
    char* ptr = nullptr;
    size_t size = 0;
};
```

如果不想允許 copy：

```cpp
Buffer(const Buffer&) = delete;
Buffer& operator=(const Buffer&) = delete;
```

## Verification

測試：

- copy 後兩邊是否各自 owns 正確 resource。
- move 後 source destructor 是否 safe。
- repeated assignment 是否不 leak。
- exception path 是否不破壞 invariant。

## Links

- [[20-languages/cpp/object-resource-semantics/Concept - Move Semantics and Ownership]]
- [[20-languages/cpp/object-resource-semantics/Concept - noexcept Move and Container Reallocation]]
- [[20-languages/cpp/object-resource-semantics/Concept - C vs C++ Semantic Lifting]]

## Source

- [[20-languages/cpp/Source Map - ChatGPT CPP RVO 解釋|Source Map]]: source lines `15144-16161`, `16409-21742`
