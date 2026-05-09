# Concept - Factory Lambda and Delayed Construction

## Derived From

- Conversation Note: [[20-languages/cpp/conversation-notes/Conversation Note - Factory Lambda and Delayed Construction]]
- Deep Dive: [[20-languages/cpp/deep-dives/Deep Dive - Factory Lambda Is Not Syntax Sugar]]

## Problem

有些 API 如果直接接收 `T value`，呼叫端就必須先把 `T` materialize 成 function argument object，API 再把它 move 到真正 storage。

```cpp
T& construct_from(T value) {
    return *new (&storage) T(std::move(value));
}
```

這裡 `value` 已經是一個 `T` object。最後放進 `storage` 時，通常還需要一次 move。

## Original Context To Keep

原始追問非常關鍵：`factory lambda / delayed construction` 看起來像包一層 lambda，所以你問「這是語法糖嗎？」後面又追問 Unreal、Unity、Chrome、browser、object pool、operation state 的實際場景。

這個 Concept 不能只寫「lambda 延遲建構」。它要保留核心疑問：傳 `T` 和傳「怎麼產生 T」到底改變了什麼？

> [!note] Quick visual
> factory lambda / delayed construction 的總覽：問題不是 lambda 語法，而是 `T` 要在參數位置先出生，還是等 final storage 已知後才出生。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_22_32.png]]

## Why Naive Fails

naive view：

```text
factory() 只是把 T{} 包起來
```

這會漏掉 API boundary 的差異。`T value` 代表 `T` 已經 materialize；`Factory` 代表你傳的是 recipe，API 可以等 final storage 準備好再呼叫它。

## Mental Model

```text
pass T value:
    construct T as parameter
    -> move into final storage

pass Factory:
    final storage is known
    -> call factory
    -> construct result into final storage path
```

## Details

```cpp
template<class Factory>
T& construct_from(Factory factory) {
    return *new (&storage) T(factory());
}
```

這對以下 type 特別重要：

- non-movable object
- move 很貴或語意上不想 move 的 object
- 需要 address-stable 的 object
- construction 需要 API 內部 state / allocator / arena 的 object

> [!note] Quick visual
> 這張圖回答「這是不是語法糖」：factory 版本改變的是 result object 的 materialization timing 與 construction location。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_22_16 (4).png]]

## Important Boundary

不是所有 factory 寫法都自動保證零 temporary。重點不是 lambda 很神，而是 API 是否真的把 `T` 的 materialization 延後到 final storage path。

如果 API 內部先存成 `T value`，或 factory result 先被綁成普通 local 再 move，那就回到 premature materialization。

對 non-copyable / non-movable `T` 要特別精準：factory 應該回傳 same-type prvalue，例如 `return T(args...);`。如果 factory 內部寫 `T local(args...); return local;`，那是 NRVO candidate，但 NRVO 不是 guaranteed；portable reasoning 不能依賴它一定成功。

## Practical Uses

- manual lifetime wrapper
- `optional<T>` 放 non-movable object
- object pool / arena allocator
- async operation state，需要 stable address
- lazy global / delayed initialization
- engine command / resource registration object

## Verification

用 non-copyable / non-movable type 測 API：

```cpp
struct X {
    X(int);
    X(const X&) = delete;
    X(X&&) = delete;
};
```

如果 `T value` 版本無法編譯，但 factory 版本能在 final storage 建構，差異就不是語法糖。

## Links

- [[20-languages/cpp/object-resource-semantics/Concept - Placement New and construct_at]]
- [[20-languages/cpp/object-resource-semantics/Concept - In-place Construction and emplace]]
- [[20-languages/cpp/object-resource-semantics/Concept - Copy Elision and C++17 prvalue]]

## Source

- [[20-languages/cpp/Source Map - ChatGPT CPP RVO 解釋|Source Map]]: source lines `3609-3974`, `4934-5370`, `5130-5305`
