# Concept - Move Semantics and Ownership

## Derived From

- Conversation Note: [[20-languages/cpp/conversation-notes/Conversation Note - Move Ownership and Value Categories]]
- Deep Dive: [[20-languages/cpp/deep-dives/Deep Dive - Move Ownership Matrix4x4 and Resource Transfer]]
- Deep Dive: [[20-languages/cpp/deep-dives/Deep Dive - std move xvalue and Return Path Selection]]

## Problem

move semantics 解決的不是「怎麼把 bytes 搬得更快」而已，而是：當一個 object 擁有 resource，而且原 owner 不再需要它時，如何把 ownership 交給另一個 object。

## Original Context To Keep

原始對話先從 `Matrix4x4` 這種純數值 type 問起，接著追問一維陣列、dynamic buffer、ownership、move 為什麼存在。這個順序很重要，因為它先排除了錯誤想法：move 不是對所有 type 都有巨大效益。

## Why Naive Fails

naive view：

```text
move = faster copy
```

對 `float` 或 `Matrix4x4 { float m[16]; }` 這種沒有外部 resource 的 type，move 可能只是 copy 那些 bytes，沒有 resource 可以偷。

move 真正有意義的情境通常是：

```text
object owns resource handle / pointer
-> destination takes handle
-> source becomes valid empty state
```

## Mental Model

move 是 ownership transfer，不是 object teleportation。

```text
before:
    a owns buffer
    b owns nothing

move:
    b takes buffer pointer
    a becomes empty but valid

after:
    b destructor frees buffer
    a destructor does nothing harmful
```

## Details

Typical move constructor:

```cpp
class Buffer {
public:
    Buffer(Buffer&& other) noexcept
        : ptr(other.ptr), size(other.size) {
        other.ptr = nullptr;
        other.size = 0;
    }

private:
    char* ptr = nullptr;
    size_t size = 0;
};
```

move 後要維持 invariant：

- destination owns the resource.
- source is valid and moved-from; it may be empty, but that is a type-specific design choice.
- destructor can safely run on both objects.

對 standard library objects，moved-from state 通常是 valid but unspecified。這代表 invariant 仍成立，但不能假設原本值還在；只能安全呼叫沒有額外 precondition 的操作，或先檢查 precondition。

## Relation To RVO

Move 和 RVO 都在回答 object delivery，但層級不同：

- Move：先有 source object，再把 resource ownership 轉給 destination object。
- RVO / copy elision：如果可以，source object 不必先獨立存在，直接在 destination storage 建構。

所以 RVO 更接近「不要搬」，move 是「可以偷資源」。

## Implementation

graphics / engine 常見適合 move-only 或 move-aware 的 type：

- CPU heap buffer
- texture / buffer handle
- file handle
- command buffer owner
- temporary mesh / image object

這些 type 通常不應該 shallow copy，可能要禁止 copy、允許 move。

## Verification

檢查 moved-from object：

- destructor 是否 safe。
- assignment 是否 safe。
- invariant 是否仍成立。
- double free 是否不可能。

## Links

- [[20-languages/cpp/object-resource-semantics/Concept - Object Delivery - Copy Move In-place Construction]]
- [[20-languages/cpp/object-resource-semantics/Concept - std move vs Move Constructor]]
- [[20-languages/cpp/object-resource-semantics/Concept - RAII and Rule of 0 3 5]]

## Source

- [[20-languages/cpp/Source Map - ChatGPT CPP RVO 解釋|Source Map]]: source lines `6743-8894`, `8022-8400`, `8434-8894`, `12974-14258`
