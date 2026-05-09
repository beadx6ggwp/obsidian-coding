# Concept - Storage vs Object Lifetime

## Derived From

- Conversation Note: [[20-languages/cpp/conversation-notes/Conversation Note - RVO Basics Return Slot and In-place Analogies]]
- Conversation Note: [[20-languages/cpp/conversation-notes/Conversation Note - RVO Verification Compile Flags and Placement New]]
- Deep Dive: [[20-languages/cpp/deep-dives/Deep Dive - Half RVO Misconception Storage and Lifetime]]

## Problem

C++ 裡要分清楚三件事：

- storage：一段可以放 object 的 memory。
- object lifetime：某個 `T` object 從 construction 到 destruction 的期間。
- source-level name：程式碼裡的變數名稱。

RVO / placement new / `optional::emplace` 都會讓這三層變得很容易混淆。

## Original Context To Keep

原始追問的重點是「同一個變數生命週期中，有時候是 RVO，有時候是一般嗎？」這個想法很重要，因為它暴露了你把 variable name、object identity、physical storage 混在一起。

## Why Naive Fails

錯誤模型：

```text
first half:
    local object lives in callee

second half:
    same object becomes caller object through RVO
```

這不是 RVO。RVO / NRVO 如果成功，決定的是 object 一開始建構在哪裡，不是同一個 object lifetime 中途切換 storage 策略。

## Mental Model

```text
storage first:
    allocate raw storage
    -> start object lifetime by construction
    -> end object lifetime by destruction
    -> storage may be reused
```

對 return-by-value 來說：

```text
NRVO success:
    caller return slot is the storage
    obj lifetime starts there

NRVO fail:
    callee local storage has obj lifetime
    caller return slot gets another object by move/copy
```

## Details

placement new 是明確在既有 storage 開始 object lifetime：

```cpp
alignas(T) unsigned char storage[sizeof(T)];
T* p = new (storage) T(args...);
p->~T();
```

RVO 的 return slot 不等於你在 source code 真的寫 placement new，但它們共享一個重要概念：object lifetime 可以在某個已知 destination storage 中開始。

## Implementation

當你看到這類 API，要先問：

- storage 是誰提供的？
- object lifetime 是在哪一行開始？
- destructor 由誰負責呼叫？
- name 是不是只是一個 source-level handle？

這個拆法比「是不是 RVO」更精準。

## Verification

可用 logging constructor / destructor 觀察是否有多個 object lifetime。若想看 compiler-expanded return mechanism，可以補 ABI / assembly / IR，但那已經是 systems-side evidence，不是 C++ source semantics 本身。

## Links

- [[20-languages/cpp/object-resource-semantics/Concept - RVO and NRVO]]
- [[20-languages/cpp/object-resource-semantics/Concept - Placement New and construct_at]]
- [[30-systems/abi-function-call/Concept - Function Call Stack and Return Object Storage]]

## Source

- [[20-languages/cpp/Source Map - ChatGPT CPP RVO 解釋|Source Map]]: source lines `771-1204`, `6042-6253`, `28072-33829`
