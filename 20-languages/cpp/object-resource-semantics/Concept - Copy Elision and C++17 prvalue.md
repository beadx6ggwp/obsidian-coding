# Concept - Copy Elision and C++17 prvalue

## Derived From

- Conversation Note: [[20-languages/cpp/conversation-notes/Conversation Note - RVO Basics Return Slot and In-place Analogies]]
- Conversation Note: [[20-languages/cpp/conversation-notes/Conversation Note - Move Ownership and Value Categories]]
- Deep Dive: [[20-languages/cpp/deep-dives/Deep Dive - RVO Return Slot and Naive Copy Model]]
- Deep Dive: [[20-languages/cpp/deep-dives/Deep Dive - std move xvalue and Return Path Selection]]

## Problem

`copy elision` 常被講成「compiler 幫你省掉 copy」。這在早期或某些 NRVO case 可以這樣理解，但 C++17 prvalue 讓這個說法不夠精確。

某些 return expression 不是先建立 temporary，再把 copy/move 消掉；而是 result object 本來就直接被初始化。

## Original Context To Keep

原始對話先從 RVO 的 return slot model 開始，後來追問 `prvalue` 的 `p` 是什麼、`return std::move(obj)` 為什麼反而不好。這個 Concept 要保留這條脈絡：prvalue path、xvalue path、NRVO path 是不同 return path，不是同一個最佳化程度的三種版本。

## Why Naive Fails

naive model：

```text
T temporary = T{};
T result = std::move(temporary);
```

這會誤導你以為 C++17 只是更會 optimize。真正差異是：在很多 prvalue 初始化情境中，語言不要求 temporary object 先 materialize。

> [!note] Quick visual
> naive、move fallback、RVO / NRVO 三種模型總對照。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_22_47 (3).png]]

## Mental Model

```text
pre-C++17 mental shortcut:
    temporary exists
    -> compiler may elide copy/move

C++17 prvalue model:
    expression computes initialization
    -> result object is initialized directly
```

prvalue 不是「短命 object」的同義詞；它更接近「可以用來初始化 object 的 pure value computation」，直到需要 materialization 時才真正成為 object。

## Details

```cpp
T make() {
    return T{};
}
```

這是 guaranteed copy elision 的典型 case。`T{}` 這個 prvalue 直接初始化 function 的 result object。

```cpp
T make() {
    T obj;
    return obj;
}
```

這是 NRVO candidate。C++ 可以把 `obj` 直接放在 return slot，但不是所有情況都保證。

```cpp
T make() {
    T obj;
    return std::move(obj);
}
```

這不是 prvalue direct construction，也通常不是 NRVO。`std::move(obj)` 產生 xvalue，讓 return path 變成 move construction candidate。

## Implementation

設計 factory / builder API 時，優先讓 caller 用 return-by-value 表達 ownership delivery：

```cpp
Texture makeTexture();

Texture texture = makeTexture();
```

不要因為害怕 copy 就先改成 out parameter。現代 C++ 的 value return 通常是清楚且高效的 default。

## Verification

用一個 logging type 記錄 constructor / copy / move / destructor。分別測：

- `return T{};`
- `T obj; return obj;`
- `T obj; return std::move(obj);`

再用不同 C++ standard 與 `-fno-elide-constructors` 對照。這會真的編譯並執行。

## Links

- [[20-languages/cpp/object-resource-semantics/Concept - RVO and NRVO]]
- [[20-languages/cpp/object-resource-semantics/Concept - std move vs Move Constructor]]
- [[20-languages/cpp/object-resource-semantics/Concept - Value Categories - lvalue xvalue prvalue]]

## Source

- [[20-languages/cpp/Source Map - ChatGPT CPP RVO 解釋|Source Map]]: source lines `18-154`, `6637-6731`, `11539-12394`
