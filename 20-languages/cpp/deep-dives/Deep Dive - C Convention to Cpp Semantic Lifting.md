# Deep Dive - C Convention to Cpp Semantic Lifting

## Derived From Conversation Notes

- [[20-languages/cpp/conversation-notes/Conversation Note - C vs C++ Semantic Lifting and Hidden Semantics]]

## Rebuild Source Rule

這篇從「C 明明能解，為什麼 C++ 這麼複雜？」生成。它不是 C/C++ 優劣比較，而是整理 hidden convention 如何被 lifting 到 type semantics。

## Original Questions

- source line `21743`: `整體上明明C能解決 為什麼CPP這些要搞的這麼複雜?`
- source line `22631`: `CPP跟C的本質差異是什麼`
- source line `23133`: `在寫C的時候 其實真正知道「語義」的人 只有開發者本身`
- source line `23640`: `能不能舉幾段範例 講解 C語言隱含的語義 但是被CPP整合好的`

## Conversation Reconstruction

```text
起點：
    你覺得 C++ 很多東西像 C pointer/memory operation 的包裝。

修正：
    C 不是做不到；差別是語義在哪裡保存。

收斂：
    C++ 的一條主線是把 programmer convention 推進 type / object lifetime / library abstraction。
```

## Hidden Semantics In C

`char*` 不告訴你：

- owner or borrower
- single object or array
- nullable or non-null
- lifetime is managed by whom
- shallow copy is safe or not

C code 可以靠 convention 做對；大型系統裡 convention 容易漂移。

## Semantic Lifting Examples

```text
malloc/free
-> RAII

struct shallow copy
-> copy constructor / Rule of 3

resource cannot copy
-> move / unique_ptr

raw pointer ownership unclear
-> smart pointer / span

pointer + length
-> span

lock/unlock
-> lock_guard

union active member
-> variant

out parameter
-> return-by-value + RVO
```

## Design Space

這不是說 C++ 解決所有語義問題。C++ 提供的是可以把語義放進程式結構的工具：

```text
type
constructor / destructor
copy / move
deleted operations
RAII handles
library vocabulary types
```

## External Source Check

- [C++ Core Guidelines - R.1](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines?lang=en#Rr-raii): RAII handle.
- [cppreference - std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr): unique ownership.
- [cppreference - std::span](https://en.cppreference.com/w/cpp/container/span): non-owning sequence view.
- [cppreference - std::variant](https://en.cppreference.com/w/cpp/utility/variant): type-safe union.

## Final Mental Model

```text
C:
    memory + pointer + function + convention

C++:
    object + lifetime + type operations + library vocabulary
```

## Source

- [[20-languages/cpp/conversation-notes/Conversation Note - C vs C++ Semantic Lifting and Hidden Semantics]]
- [[20-languages/cpp/Source Map - ChatGPT CPP RVO 解釋|Source Map]]: source lines `21743-28071`

