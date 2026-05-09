# Reading - CppCon Watchlist for C++ Object Semantics

## Source

- [[20-languages/cpp/Source Map - ChatGPT CPP RVO 解釋|Source Map]]

## Purpose

這份 watchlist 對應 `RVO -> object delivery -> ownership/lifetime -> value semantics -> generic programming` 這條學習線。這不是單純找 RVO talk，而是補完整 C++ object/resource semantics 的背景。

## Watch Order

### 1. Bjarne Stroustrup - The Essence of C++

Why:

- 建立 C++ 的大方向：abstraction mechanisms、classes、templates、resource safety、zero-overhead。
- 對應 [[20-languages/cpp/object-resource-semantics/Concept - C vs C++ Semantic Lifting]]。

### 2. Back to Basics - RAII in C++

Why:

- 把 resource lifetime 綁到 object lifetime。
- 對應 [[20-languages/cpp/object-resource-semantics/Concept - RAII and Rule of 0 3 5]]。

### 3. Back to Basics - C++ Move Semantics

Why:

- 補 `std::move`、move constructor、moved-from state、ownership transfer。
- 對應 [[20-languages/cpp/object-resource-semantics/Concept - Move Semantics and Ownership]]。

### 4. Back to Basics - Value Categories

Why:

- 補 lvalue / xvalue / prvalue / materialization。
- 對應 [[20-languages/cpp/object-resource-semantics/Concept - Value Categories - lvalue xvalue prvalue]]。

### 5. Arthur O'Dwyer - Return Value Optimization

Why:

- 深入 RVO / NRVO 的 tricky cases。
- 對應 [[20-languages/cpp/object-resource-semantics/Concept - RVO and NRVO]]。

### 6. Jason Turner - Surprises in Object Lifetime

Why:

- 補 object lifetime 和 storage 的邊界。
- 對應 [[20-languages/cpp/object-resource-semantics/Concept - Storage vs Object Lifetime]]。

### 7. Sean Parent - Value Semantics and Concepts-Based Polymorphism

Why:

- 把 object 從「data + methods」推到 value semantics、regular type、local reasoning。
- 對應 [[20-languages/cpp/object-resource-semantics/Concept - C++ Type as Operations and Invariants]]。

### 8. Alexander Stepanov - Generic Programming / Elements of Programming

Why:

- 補 concepts、regular types、algebraic structures、algorithm requirements。
- 對應「type as operations and laws」這條線。

## Verification Needed

這份是從原始對話整理的 watchlist。正式引用前應逐一驗證：

- talk title
- speaker
- year
- official video / slides URL
- 是否真的涵蓋此處標註的內容

## Related

- [[20-languages/cpp/MOC - C++ Object and Resource Semantics]]
- [[20-languages/cpp/Question Trail - ChatGPT CPP RVO 解釋]]

## Source

- [[20-languages/cpp/Source Map - ChatGPT CPP RVO 解釋|Source Map]]: source lines `34853-35237`

