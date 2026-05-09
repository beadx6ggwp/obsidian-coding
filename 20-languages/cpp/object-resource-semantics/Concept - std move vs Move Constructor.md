# Concept - std move vs Move Constructor

## Derived From

- Conversation Note: [[20-languages/cpp/conversation-notes/Conversation Note - Move Ownership and Value Categories]]
- Conversation Note: [[20-languages/cpp/conversation-notes/Conversation Note - C vs C++ Semantic Lifting and Hidden Semantics]]
- Deep Dive: [[20-languages/cpp/deep-dives/Deep Dive - std move xvalue and Return Path Selection]]

## Problem

`std::move` 這個名字很容易誤導。它不搬資料，也不一定呼叫 move constructor。它只是把 expression cast 成 xvalue，讓後續 overload resolution 可以選到 move operation。

## Original Context To Keep

原始對話中，你不是只問 `std::move` 是什麼，而是在追：

- `return std::move(mat)` 對純數值 `Matrix4x4` 有沒有意義？
- `std::move` 為什麼會讓 NRVO 失效？
- move 的 ownership 到底在哪裡？
- `std::move` 和 move constructor 是不是同一件事？

## Why Naive Fails

naive view：

```text
std::move(x)
-> moves x
```

實際上：

```text
std::move(x)
-> cast x to xvalue
-> later code may call move constructor / move assignment
```

如果後續沒有需要建構或賦值新 object，資料沒有被搬。

## Mental Model

```cpp
T a;
T b = std::move(a);
```

這裡分兩步：

1. `std::move(a)` 把 `a` 這個 expression 變成 xvalue。
2. `T b = ...` 需要初始化 `b`，因此 overload resolution 選 move constructor。

## Details

move constructor 是 type 的 operation：

```cpp
T(T&& other);
```

`std::move` 是 expression cast：

```cpp
static_cast<T&&>(x)
```

所以 `std::move` 只表達「我允許你把這個 object 當成 expiring object 使用」。真正 resource transfer 要由 type 的 move constructor / move assignment 實作。

## In Return Statements

```cpp
T make() {
    T obj;
    return obj;            // NRVO candidate
}

T make2() {
    T obj;
    return std::move(obj); // xvalue, usually disables NRVO
}
```

`return obj` 保留 NRVO 機會。`return std::move(obj)` 改變 expression category，通常讓 compiler 只能走 move path。

C++23 nuance：`return obj;` 這種 move-eligible return expression 在 fallback path 會被當成 xvalue 做 overload resolution，所以不需要為了「讓它 move」手動寫 `std::move`。`return obj;` 同時保留 NRVO shape；`return std::move(obj);` 通常只剩 explicit xvalue path。

## Matrix Example

`Matrix4x4` 如果只是 `float m[16]`，move 可能和 copy 沒有本質差異，因為沒有 heap resource 可以偷。move 的價值主要出現在 resource-owning type。

## Verification

用 logging type 分別測：

- `T b = a;`
- `T b = std::move(a);`
- `return obj;`
- `return std::move(obj);`

觀察 copy / move constructor 是否被呼叫。

## Links

- [[20-languages/cpp/object-resource-semantics/Concept - Value Categories - lvalue xvalue prvalue]]
- [[20-languages/cpp/object-resource-semantics/Concept - Move Semantics and Ownership]]
- [[20-languages/cpp/object-resource-semantics/Concept - RVO and NRVO]]

## Source

- [[20-languages/cpp/Source Map - ChatGPT CPP RVO 解釋|Source Map]]: source lines `197-234`, `6311-6582`, `6743-8894`, `11539-12394`
