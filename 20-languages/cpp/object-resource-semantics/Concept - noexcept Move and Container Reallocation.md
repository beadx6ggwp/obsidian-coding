# Concept - noexcept Move and Container Reallocation

## Derived From

- Conversation Note: [[20-languages/cpp/conversation-notes/Conversation Note - Report Topic Object Delivery RAII and Object Not Data]]
- Deep Dive: [[20-languages/cpp/deep-dives/Deep Dive - Buffer Bug RAII Rules and noexcept Move]]

## Problem

`noexcept` move 不只是效能標記。對 standard containers 來說，它會影響 reallocation 時能不能安全地 move elements。

## Original Context To Keep

原始對話是在 RAII / Rule of 5 之後問到 `noexcept move`。它不是孤立語法，而是接在 resource owner 的 move operation 是否能被 library 信任。

## Why Naive Fails

naive view：

```text
move 比 copy 快
-> vector reallocation 一定用 move
```

但如果 move constructor 可能 throw，container 在搬一半時可能破壞 strong exception guarantee。這時 container 可能改用 copy，或在不可 copy 的 type 上面臨更受限的策略。

## Mental Model

vector reallocation：

```text
allocate new storage
-> construct elements in new storage
-> destroy old elements
-> release old storage
```

如果 move 是 `noexcept`：

```text
old element can be moved safely
-> if later operation fails, library can reason about state
```

如果 move 可能 throw：

```text
moving may alter source
then throw
-> old vector state may be hard to restore
```

## `std::move_if_noexcept`

這個規則可以用 `std::move_if_noexcept` 具體化：

```text
if T is nothrow move constructible
    use T&&
else if T is not copy constructible
    still use T&&
else
    use const T& so copy can preserve strong exception guarantee
```

所以 `noexcept` 不是「加速器開關」，而是 type 對 generic library 說：

```text
relocating me by move will not throw;
you can choose the move path without losing rollback reasoning.
```

## Details

```cpp
class Buffer {
public:
    Buffer(Buffer&& other) noexcept;
    Buffer& operator=(Buffer&& other) noexcept;
};
```

對 resource owner 來說，move 通常只是 steal pointer / handle，理論上不應配置新 resource，因此常可以也應該是 `noexcept`。

用 type trait 表達就是：

```cpp
static_assert(std::is_nothrow_move_constructible_v<Buffer>);
```

如果這個 assertion 不成立，`std::vector<Buffer>` 之類的 container 在 reallocation 時可能不走你期待的 move path。

## Relation To Rule of 5

如果你手寫 move constructor / move assignment，應同時檢查：

- 是否真的不會 throw。
- moved-from state 是否 valid。
- destructor 是否 safe。
- copy 是否要 delete 或 deep copy。

## Relation To RVO

RVO 成功時不需要 move。但只要有 fallback path，move constructor 的 quality 仍然重要。`return obj;` 保留 NRVO 機會；NRVO 失敗時，move fallback 可能接手。

## Verification

可以用 `std::is_nothrow_move_constructible_v<T>` 檢查 type trait，也可以用 vector reallocation logging type 觀察 copy / move path。

## Links

- [[20-languages/cpp/object-resource-semantics/Concept - RAII and Rule of 0 3 5]]
- [[20-languages/cpp/object-resource-semantics/Concept - Move Semantics and Ownership]]
- [[20-languages/cpp/object-resource-semantics/Concept - Object Delivery - Copy Move In-place Construction]]

## Source

- [[20-languages/cpp/Source Map - ChatGPT CPP RVO 解釋|Source Map]]: source lines `15144-16161`, `15650-16161`
- [cppreference - std::move_if_noexcept](https://en.cppreference.com/w/cpp/utility/move_if_noexcept)
- [C++ Core Guidelines - C.66](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines?lang=en#Rc-move-noexcept)
