# Concept - Value Categories - lvalue xvalue prvalue

## Derived From

- Conversation Note: [[20-languages/cpp/conversation-notes/Conversation Note - Move Ownership and Value Categories]]
- Deep Dive: [[20-languages/cpp/deep-dives/Deep Dive - Value Categories Beyond std move]]
- Deep Dive: [[20-languages/cpp/deep-dives/Deep Dive - std move xvalue and Return Path Selection]]

## Problem

C++ expression 不只看 type，也看 value category。`T`、`T&`、`T&&` 是 type 層；`lvalue`、`xvalue`、`prvalue` 是 expression 層。

move constructor、copy constructor、overload resolution、return path 都會受 value category 影響。

## Original Context To Keep

原始對話不是從教科書分類開始，而是從這些問題被逼出來：

- 為什麼 `std::move` 後是 xvalue？
- `prvalue` 的 `p` 是什麼？
- `return std::move(mat)` 為什麼和 `return mat` 不同？
- `T&&` return / rvalue accessor 到底在做什麼？

## Why Naive Fails

naive view：

```text
rvalue = temporary
lvalue = variable
```

這太粗。`std::move(x)` 不是 temporary object，但它是 xvalue。`T{}` 是 prvalue。`T&& name` 這個 named variable expression 反而是 lvalue。

## Mental Model

先分兩個問題：

```text
Does this expression identify an existing object?
    yes -> glvalue
    no  -> prvalue

Can resources be reused from it?
    yes, expiring object -> xvalue
```

常用分類：

- `lvalue`: 有 identity，通常不能被當成可偷資源。
- `xvalue`: 有 identity，但表示 expiring value，可以被 move。
- `prvalue`: pure value，常用來初始化 object，C++17 後很多情況不先 materialize temporary。

## Details

```cpp
T a;

a;            // lvalue
std::move(a); // xvalue
T{};          // prvalue
```

`T&&` type 不代表 expression 一定是 xvalue：

```cpp
void f(T&& x) {
    x;            // lvalue expression
    std::move(x); // xvalue expression
}
```

這就是為什麼 forwarding / move code 不能只看 type spelling。

## Implementation

設計 API 時，常見 pattern：

- `const T&`: borrow, no ownership transfer.
- `T&&`: caller 願意讓 callee reuse resource.
- `T`: callee 取得自己的 value，可以 copy 或 move 進來。
- return `T`: value delivery，讓 RVO / move fallback 發揮。

避免除非真的需要，否則不要 return `T&&` 指向 local 或 temporary。

## Verification

可以用 overload set 測 category：

```cpp
void probe(const T&);
void probe(T&&);

T x;
probe(x);            // const T& or T&
probe(std::move(x)); // T&&
probe(T{});          // T&&, initialized from prvalue path
```

## Links

- [[20-languages/cpp/object-resource-semantics/Concept - std move vs Move Constructor]]
- [[20-languages/cpp/object-resource-semantics/Concept - Copy Elision and C++17 prvalue]]
- [[20-languages/cpp/object-resource-semantics/Concept - Move Semantics and Ownership]]

## Source

- [[20-languages/cpp/Source Map - ChatGPT CPP RVO 解釋|Source Map]]: source lines `6279-6731`, `9607-11135`
