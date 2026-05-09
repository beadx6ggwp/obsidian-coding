# Concept - Placement New and construct_at

## Derived From

- Conversation Note: [[20-languages/cpp/conversation-notes/Conversation Note - RVO Verification Compile Flags and Placement New]]
- Deep Dive: [[20-languages/cpp/deep-dives/Deep Dive - Destination First Construction Across RVO Emplace and Placement New]]
- Deep Dive: [[20-languages/cpp/deep-dives/Deep Dive - Half RVO Misconception Storage and Lifetime]]

## Problem

有時候你已經有一塊 raw storage，但還沒有 `T` object。問題是：如何在指定位置開始 `T` 的 object lifetime？

placement new 和 `std::construct_at` 解決的是這個問題。

## Original Context To Keep

原始對話是在理解 return slot 時問到：

```cpp
new (return_slot) std::vector<int>();
```

這是不是「真實語法」？答案是 placement new 的形式是真的，但 compiler 的 return slot 不一定是你能在 C++ source 裡直接拿到的變數。

## Why Naive Fails

錯誤直覺是把 memory allocation 和 object construction 當成同一件事。

```text
malloc / raw buffer:
    only storage exists

constructor / placement new:
    object lifetime starts
```

有 storage 不代表有 object；有 object 才有 constructor-established invariant，也才需要 destructor 結束 lifetime。

> [!note] Quick visual
> placement new / `std::construct_at`：在指定 raw storage 直接開始 object lifetime。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_21_33 (2).png]]

## Mental Model

```text
raw storage
-> construct object there
-> use object
-> destroy object
-> storage becomes raw storage again
```

placement new 是 low-level lifetime control。`std::construct_at` 是 C++20 提供的 library interface，讓這件事比較適合 generic code。

## Details

```cpp
alignas(T) unsigned char storage[sizeof(T)];

T* p = new (storage) T(args...);
p->~T();
```

C++20：

```cpp
alignas(T) unsigned char storage[sizeof(T)];
T* p = std::construct_at(reinterpret_cast<T*>(storage), args...);
std::destroy_at(p);
```

這類 code 必須非常清楚 lifetime、alignment、exception safety 和 destructor responsibility。

和 RVO / return slot 的關係要保留邊界：placement new / `construct_at` 是你在 source code 裡真的能使用的 lifetime-control tool；return slot 是 compiler / ABI 可能採用的 result-object storage model，不是 C++ 標準給你的變數。

## Implementation

常見使用位置：

- `std::optional<T>` internal storage
- container node / buffer management
- arena allocator / object pool
- variant-like active member management
- systems code 裡的 manual lifetime wrapper

## Verification

檢查點：

- storage alignment 是否滿足 `T`。
- constructor 是否真的被呼叫。
- destructor 是否 exactly once。
- exception path 是否不會 destroy 未建構 object。

## Links

- [[20-languages/cpp/object-resource-semantics/Concept - Storage vs Object Lifetime]]
- [[20-languages/cpp/object-resource-semantics/Concept - In-place Construction and emplace]]
- [[20-languages/cpp/object-resource-semantics/Concept - Factory Lambda and Delayed Construction]]

## Source

- [[20-languages/cpp/Source Map - ChatGPT CPP RVO 解釋|Source Map]]: source lines `6042-6253`
