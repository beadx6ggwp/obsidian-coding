# Deep Dive - Type Operations Laws Invariants and Stepanov

## Derived From Conversation Notes

- [[20-languages/cpp/conversation-notes/Conversation Note - Type Operations Mathematical Analogy and Stepanov]]
- [[20-languages/cpp/conversation-notes/Conversation Note - C vs C++ Semantic Lifting and Hidden Semantics]]

## Rebuild Source Rule

這篇從抽象代數類比與 Stepanov 追問生成。它不是宣稱 C++ type 等於 group，而是保留原始對話如何用數學類比逼出 operation / invariant 思考法。

## Original Questions

- source line `25572`: `如果有資源 那可不可複製...是不是跟線性代數 離散數學的概念類似`
- source line `29571`: `從抽象代數(群環體等等)去類比 會怎麼看`
- source line `30544`: 追問 `Group = Set + operation + identity + inverse + laws`
- source line `31603`: `CPP的設計思路 就是沿著數學的抽象代數出發的嗎?`
- source line `31929`: `Alexander Stepanov 是誰`
- source line `32096`: `很多操作 是可以像數學theorme那樣 來思考哪些是正確操作?`

## Conversation Reconstruction

```text
起點：
    resource 可不可 copy 被你連到 operation 是否合法。

深化：
    group = set + operation + laws 成為 type = valid states + operations + invariants 的類比。

修正：
    C++ 整體不是抽象代數推導；STL / generic programming / concepts 這條線更接近。

收斂：
    可以像 theorem 一樣檢查 operations，但要加 lifetime / side effects / exception / cost model。
```

## Useful Analogy

```text
carrier set
-> valid state set

operation
-> constructor / copy / move / compare / assign

closure
-> operation after-state remains valid

laws
-> equality relation, ordering law, ownership invariant
```

## Limits

不要過度類比：

- destructor has side effects
- move is not algebraic inverse
- allocation can throw
- UB exists
- object lifetime matters
- representation and cost model matter

## Stepanov Connection

Stepanov / generic programming 的重點：

```text
algorithm 不問你是不是某個 concrete class
algorithm 問你是否滿足 operation requirements / concepts
```

Examples:

- regular type
- semiregular type
- iterator category
- strict weak ordering
- range requirements

## Regular / Semiregular As Calibrated Vocabulary

`regular` / `semiregular` 是把數學類比拉回 C++20 concepts 的好入口。

```text
semiregular:
    copyable + default initializable

regular:
    semiregular + equality comparable
```

這不是說所有 resource owner 都應該 regular。`unique_ptr` 這種 move-only owner 明確不 regular，因為 copy 沒有合理語義。重點是：generic algorithm 可以用 concepts 描述它需要的 operation set，而不是只說「template 可以吃任何 type」。

所以更精準的說法是：

```text
math analogy:
    use operations and laws to reason about types

C++ calibration:
    also include lifetime, mutation, exception safety, ownership, and cost model
```

## External Source Check

- [cppreference - constraints and concepts](https://en.cppreference.com/w/cpp/language/constraints.html): concepts as constraints.
- [cppreference - std::regular](https://en.cppreference.com/w/cpp/concepts/regular): regular vocabulary.
- [cppreference - std::semiregular](https://en.cppreference.com/w/cpp/concepts/semiregular): semiregular vocabulary.
- [C++ Core Guidelines - T.20](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines?lang=en#Rt-low): concepts need meaningful semantics.

## Final Mental Model

```text
Type =
    valid states
  + operations
  + invariants / laws
  + cost expectations
  + C++ lifetime / exception / representation constraints
```

## Source

- [[20-languages/cpp/conversation-notes/Conversation Note - Type Operations Mathematical Analogy and Stepanov]]
- [[20-languages/cpp/Source Map - ChatGPT CPP RVO 解釋|Source Map]]: source lines `25572-26750`, `28072-33829`
