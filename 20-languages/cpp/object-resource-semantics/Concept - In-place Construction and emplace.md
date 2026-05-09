# Concept - In-place Construction and emplace

## Derived From

- Conversation Note: [[20-languages/cpp/conversation-notes/Conversation Note - RVO Basics Return Slot and In-place Analogies]]
- Conversation Note: [[20-languages/cpp/conversation-notes/Conversation Note - RVO Verification Compile Flags and Placement New]]
- Deep Dive: [[20-languages/cpp/deep-dives/Deep Dive - Destination First Construction Across RVO Emplace and Placement New]]

## Problem

如果 destination storage 已經知道，為什麼要先在別處建一個 `T`，再 move / copy 到 destination？

`emplace` family 的目標是：把 constructor arguments 送到 container / wrapper 裡，讓 `T` 直接在 final storage 被建構。

## Original Context To Keep

原始對話是在找「跟 RVO 同構的情境」。重點不是把所有東西都叫 RVO，而是找出共同方向：避免 premature materialization，讓 object 在目的地出生。

> [!note] Quick visual
> in-place construction family overview。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_22_01 (5).png]]

## Why Naive Fails

`push_back(T(args...))` 的 mental model：

```text
construct temporary T
-> move/copy into vector element slot
```

如果 `T` 很大、不可 move、或 move 有語意成本，這不是理想路徑。

## Mental Model

`emplace_back(args...)` 的 mental model：

```text
vector reserves element slot
-> forwards args
-> constructs T directly in element slot
```

> [!note] Quick visual
> `std::vector::emplace_back` as RVO-like construction into element slot。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_21_33 (1).png]]

## Details

常見例子：

- `vector.emplace_back(args...)`
- `optional.emplace(args...)`
- `map.try_emplace(key, args...)`
- `make_unique<T>(args...)`
- placement new / `construct_at`

它們不是同一個語言規則，但都在往同一個工程方向走：destination-first construction。

## Important Boundary

`emplace` 不是 magic。

如果你已經先建立好一個 `T`：

```cpp
T value(args...);
vec.emplace_back(std::move(value));
```

那 `value` 已經 materialize 了，`emplace_back` 只是在 element slot 呼叫 move constructor。

另外，`vector.emplace_back(args...)` 只描述新 element 的 construction site；如果 vector reallocate，既有 elements 仍可能 move/copy 到新 buffer。`optional.emplace(args...)` 則沒有 reallocation，但如果原本有 contained value，會先 destroy 舊 value 再建新的。

## Verification

用 logging type 比較：

- `push_back(T(args...))`
- `emplace_back(args...)`
- `emplace_back(T(args...))`

觀察 constructor / move constructor 呼叫次數。

## Links

- [[20-languages/cpp/object-resource-semantics/Concept - Placement New and construct_at]]
- [[20-languages/cpp/object-resource-semantics/Concept - Factory Lambda and Delayed Construction]]
- [[20-languages/cpp/object-resource-semantics/Concept - Object Delivery - Copy Move In-place Construction]]

## Source

- [[20-languages/cpp/Source Map - ChatGPT CPP RVO 解釋|Source Map]]: source lines `1687-2201`, `6042-6253`
