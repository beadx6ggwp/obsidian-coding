# C++ Object Semantics Report - Outline

## Working Title

C++ Object 不只是 Data：從 Copy、Move 到 RVO 的物件交付模型

## Thesis

C++ 的很多特性不是單純語法糖，而是在 C 的 memory / pointer / convention 之上，把 object lifetime、ownership、copy、move、construction location 和 cost model 變成 type system、compiler rule 與 library abstraction 可以支援的工程模型。

> [!note] Slide candidate
> 這張圖可以當整份報告的總覽：從 naive return-by-value、RVO/NRVO、C++17 prvalue，到 factory / delayed construction，剛好對應「object 在哪裡出生」這條主線。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_22_10.png]]

## Recommended 40-Minute Structure

### 1. Opening Case: Why Return by Value Looks Expensive

Use a small `Image` / `Buffer` example.

Naive model:

```text
local object
-> return temporary
-> caller object
```

Corrected model:

```text
caller return slot
-> callee constructs result directly there
```

Related:

- [[20-languages/cpp/object-resource-semantics/Concept - RVO and NRVO]]

### 2. Object Delivery Strategies

Core frame:

- Copy: A has one, B wants another.
- Move: A no longer needs it, B takes ownership.
- In-place construction: directly construct at B.

> [!note] Slide candidate
> 這張圖可以接在三種 object delivery 之後，用來把抽象模型落回 API 選擇：return-by-value、emplace / factory，或 mutation API。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_22_01 (4).png]]

Related:

- [[20-languages/cpp/object-resource-semantics/Concept - Object Delivery - Copy Move In-place Construction]]

### 3. Value Category As Control Signal

Explain why `std::move(obj)` changes expression category, not the object itself.

Related:

- [[20-languages/cpp/object-resource-semantics/Concept - Value Categories - lvalue xvalue prvalue]]
- [[20-languages/cpp/object-resource-semantics/Concept - std move vs Move Constructor]]

### 4. C++17 prvalue and Copy Elision

Key message: guaranteed copy elision is not merely an optimization. The intermediate object may not exist in the language model.

Related:

- [[20-languages/cpp/object-resource-semantics/Concept - Copy Elision and C++17 prvalue]]

### 5. Ownership and Resource Types

Contrast:

- `Matrix4x4`: mostly value data, move similar to copy.
- `Buffer`, `unique_ptr`, GPU handle wrapper: ownership matters.

Related:

- [[20-languages/cpp/object-resource-semantics/Concept - Move Semantics and Ownership]]

### 6. Extensions

Use these to expand the report beyond RVO:

- [[20-languages/cpp/object-resource-semantics/Concept - Storage vs Object Lifetime]]
- [[20-languages/cpp/object-resource-semantics/Concept - Placement New and construct_at]]
- [[20-languages/cpp/object-resource-semantics/Concept - In-place Construction and emplace]]
- [[20-languages/cpp/object-resource-semantics/Concept - Factory Lambda and Delayed Construction]]
- [[20-languages/cpp/object-resource-semantics/Concept - RAII and Rule of 0 3 5]]
- [[20-languages/cpp/object-resource-semantics/Concept - noexcept Move and Container Reallocation]]
- [[20-languages/cpp/object-resource-semantics/Concept - C vs C++ Semantic Lifting]]
- [[20-languages/cpp/object-resource-semantics/Concept - C++ Type as Operations and Invariants]]

> [!note] Slide candidates
> 這組圖可以當 extensions 的素材：需求分析、實際改寫流程、前提條件、適合/不適合 cases、常見誤判。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_21_50 (1).png]]
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_21_50 (2).png]]
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_21_50 (3).png]]
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_21_50 (4).png]]
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_21_51 (5).png]]

### 7. Closing Frame

End with the larger thesis:

```text
C:
    memory + pointer + convention

C++:
    object + lifetime + ownership + type operations + cost model
```

RVO is one node in this model: it answers where a result object begins its lifetime.

## Source

- [[20-languages/cpp/Source Map - ChatGPT CPP RVO 解釋|Source Map]]: source lines `12395-21742`, `27101-28071`, `32663-33800`
