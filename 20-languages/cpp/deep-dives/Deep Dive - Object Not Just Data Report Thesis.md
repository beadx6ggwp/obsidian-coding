# Deep Dive - Object Not Just Data Report Thesis

## Derived From Conversation Notes

- [[20-languages/cpp/conversation-notes/Conversation Note - Report Topic Object Delivery RAII and Object Not Data]]
- [[20-languages/cpp/conversation-notes/Conversation Note - Type Operations Mathematical Analogy and Stepanov]]

## Rebuild Source Rule

這篇從報告命名焦慮生成。它不是一般 object model 定義，而是保留「起不好主題代表還沒深入本質」這條推理。

## Original Questions

- source line `14360`: `有沒有更好更生動的主題?`
- source line `14530`: `沒辦法起一個好主題 代表我還沒有真正深入本質`
- source line `18396`: `CPP的物件不只是資料: 從 shallow copy bug 到 move semantics 與 RVO`
- source line `18863`: `Object不只是Data 那他是什麼?`
- source line `20936`: `Object 有 lifetime...ownership...Copy / move 由 type 定義...Storage 與 object 不是同一件事...Cost model`

## Conversation Reconstruction

```text
起點：
    RVO / move 都太局部，報告主題抓不到核心。

轉折：
    Buffer bug 讓主題從技巧變成 object semantics。

收斂：
    Object 不只是 Data，因為 C++ object 還牽涉 lifetime、ownership、operations、storage relation、cost model。
```

## The Thesis

```text
C++ object is not merely fields plus methods.
For non-trivial types, correctness is in lifetime and operations.
```

Formal layer:

```text
object has size, alignment, storage duration, lifetime, type, value, optional name
```

Design layer:

```text
valid states
ownership semantics
allowed operations
invariants
cost model
```

不是每個 object 都有 ownership。`int`、`Vec3`、`Matrix4x4` 可以是 value data；`Buffer`、`vector`、`unique_ptr` 才是 resource owner。

## Why This Is A Better Report Frame

它能串起：

- shallow copy bug
- destructor / RAII
- Rule of 3/5/0
- move semantics
- moved-from state
- noexcept move
- RVO / NRVO
- storage vs lifetime
- type invariants

這比單講 RVO 更像完整報告，因為 RVO 只是「object 在哪裡出生」。

> [!note] 圖解定位
> 這張圖放在 report thesis 附近：常見踩雷都來自 object delivery / lifecycle semantics 沒想清楚。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_22_01 (6).png]]

## External Source Check

- [cppreference - object](https://en.cppreference.com/w/cpp/language/objects): formal object properties.
- [cppreference - lifetime](https://en.cppreference.com/w/cpp/language/lifetime.html): lifetime/storage boundary.
- [C++ Core Guidelines - R.1](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines?lang=en#Rr-raii): RAII.

## Final Mental Model

```text
Data answers: what bytes are stored?
Object semantics answers: when does it live, who owns resource, what operations are legal, and what invariants must hold?
```

## Source

- [[20-languages/cpp/conversation-notes/Conversation Note - Report Topic Object Delivery RAII and Object Not Data]]
- [[20-languages/cpp/Source Map - ChatGPT CPP RVO 解釋|Source Map]]: source lines `14360-21742`

