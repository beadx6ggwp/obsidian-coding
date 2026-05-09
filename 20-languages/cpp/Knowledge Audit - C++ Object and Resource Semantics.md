# Knowledge Audit - C++ Object and Resource Semantics

## Purpose

這份 audit 是 `Deep Dive` / `Concept` 的校準層。

原始文章與 Conversation Notes 在這個階段不再更動，因為它們的任務是保存原始思考脈絡。需要繼續修正的是：

- Deep Dive：推理文章是否精準、是否需要補標準 / reference / guideline。
- Concept：lookup card 是否太簡化、是否缺條件、是否把 mental model 說成標準語義。
- Cross-area notes：Rust / ABI / GC / Stepanov 這些延伸是否過度簡化。

## Status Labels

- `Correct`: 目前說法可保留。
- `Needs Nuance`: 方向正確，但需要補版本、條件、例外或語氣。
- `Mental Model Only`: 可以保留作理解模型，但不能寫成 C++ standard guarantee。
- `Extension`: 值得延伸成獨立 note / reading note。
- `Fix Needed`: 需要直接修改 Deep Dive / Concept。

## Source Priority

1. C++ draft / standard wording
2. cppreference
3. C++ Core Guidelines
4. Rust official book / Rust reference
5. CppCon / talks / engineering material

## High-Risk Claim Audit

| Claim | Status | Calibration | Main Sources | Fix Targets |
| --- | --- | --- | --- | --- |
| `return T{}` in C++17 directly initializes the result object and does not require a move/copy constructor. | `Correct` | Keep distinction: this is prvalue / guaranteed copy elision; it is not merely an optional optimization. Destructor accessibility still matters. | [cppreference copy elision](https://en.cppreference.com/w/cpp/language/copy_elision.html), [stmt.return](https://eel.is/c%2B%2Bdraft/stmt.return) | [[20-languages/cpp/deep-dives/Deep Dive - RVO Return Slot and Naive Copy Model]], [[20-languages/cpp/object-resource-semantics/Concept - Copy Elision and C++17 prvalue]] |
| NRVO constructs a named local directly in the caller result object when it succeeds. | `Correct` | Must always say NRVO is permitted/optional, not guaranteed. It applies to a non-volatile automatic local object meeting the return-expression conditions. | [class.copy.elision](https://eel.is/c%2B%2Bdraft/class.copy.elision), [cppreference copy elision](https://en.cppreference.com/w/cpp/language/copy_elision.html) | [[20-languages/cpp/deep-dives/Deep Dive - RVO Return Slot and Naive Copy Model]], [[20-languages/cpp/object-resource-semantics/Concept - RVO and NRVO]] |
| When copy elision occurs, source and target can be treated as the same object. | `Needs Nuance` | Correct for standard copy elision wording, but do not over-apply it to C++17 prvalue direct initialization where there is no separate source temporary. | [class.copy.elision](https://eel.is/c%2B%2Bdraft/class.copy.elision), [cppreference copy elision](https://en.cppreference.com/w/cpp/language/copy_elision.html) | [[20-languages/cpp/deep-dives/Deep Dive - Half RVO Misconception Storage and Lifetime]], [[20-languages/cpp/object-resource-semantics/Concept - Storage vs Object Lifetime]] |
| `return std::move(local)` usually prevents NRVO and is a pessimization. | `Needs Nuance` | Direction is right and matches Core Guidelines F.48. Add C++23 nuance: move-eligible return expressions are treated as xvalues for overload resolution when elision does not happen; explicit `std::move` still blocks NRVO candidate shape. | [cppreference return](https://en.cppreference.com/w/cpp/language/return.html), [Core Guidelines F.48](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines?lang=en#Rf-return-move-local), [MSVC C26479](https://learn.microsoft.com/en-us/cpp/code-quality/c26479?view=msvc-170) | [[20-languages/cpp/deep-dives/Deep Dive - std move xvalue and Return Path Selection]], [[20-languages/cpp/object-resource-semantics/Concept - std move vs Move Constructor]] |
| `std::move` moves an object. | `Fix Needed if phrased this way` | Correct phrasing: `std::move` produces an xvalue expression; actual resource transfer happens only if a move operation is selected and implemented. Current notes mostly say this correctly. | [cppreference std::move](https://en.cppreference.com/w/cpp/utility/move), [cppreference value categories](https://en.cppreference.com/w/cpp/language/value_category) | [[20-languages/cpp/deep-dives/Deep Dive - std move xvalue and Return Path Selection]], [[20-languages/cpp/object-resource-semantics/Concept - std move vs Move Constructor]] |
| Moved-from objects are valid but can be used normally. | `Needs Nuance` | For standard library types, moved-from state is valid but unspecified. Operations without preconditions are safe; operations with preconditions still require checking. For user types, the type author must define and preserve invariants. | [cppreference std::move](https://en.cppreference.com/w/cpp/utility/move), [cppreference move constructor](https://en.cppreference.com/w/cpp/language/move_constructor.html) | [[20-languages/cpp/deep-dives/Deep Dive - Move Ownership Matrix4x4 and Resource Transfer]], [[20-languages/cpp/object-resource-semantics/Concept - Move Semantics and Ownership]] |
| A `Matrix4x4` made of `float[16]` does not benefit much from move. | `Correct` | Good teaching counterexample. Add condition: if `Matrix4x4` owns heap/GPU/native handles, move can become meaningful. | [cppreference move constructor](https://en.cppreference.com/w/cpp/language/move_constructor.html) | [[20-languages/cpp/deep-dives/Deep Dive - Move Ownership Matrix4x4 and Resource Transfer]], [[20-languages/cpp/object-resource-semantics/Concept - Move Semantics and Ownership]] |
| Storage and object lifetime are separate. | `Correct` | Strongly supported. Lifetime begins when suitable storage is obtained and initialization is complete; lifetime can end before storage is released or reused. | [basic.life](https://eel.is/c%2B%2Bdraft/basic.life), [cppreference lifetime](https://en.cppreference.com/w/cpp/language/lifetime.html), [intro.object](https://eel.is/c%2B%2Bdraft/intro.object) | [[20-languages/cpp/deep-dives/Deep Dive - Half RVO Misconception Storage and Lifetime]], [[20-languages/cpp/object-resource-semantics/Concept - Storage vs Object Lifetime]] |
| Placement new / `construct_at` are the real syntax behind return slot. | `Mental Model Only` | Placement new and `construct_at` are real ways to start lifetime in chosen storage. Return slot is a compiler/ABI/codegen model, not source-level access to a standard object named `return_slot`. | [cppreference new expression](https://en.cppreference.com/w/cpp/language/new), [cppreference construct_at](https://en.cppreference.com/w/cpp/memory/construct_at), [stmt.return](https://eel.is/c%2B%2Bdraft/stmt.return) | [[20-languages/cpp/deep-dives/Deep Dive - Half RVO Misconception Storage and Lifetime]], [[20-languages/cpp/object-resource-semantics/Concept - Placement New and construct_at]], [[30-systems/abi-function-call/Concept - Function Call Stack and Return Object Storage]] |
| `emplace` guarantees no move. | `Needs Nuance` | `emplace(args...)` constructs in-place from constructor arguments, but `emplace(T{})` or reallocation of existing elements can still involve moves/copies. Optional `emplace` destroys old contained value first. | [optional::emplace](https://en.cppreference.com/w/cpp/utility/optional/emplace), [cppreference move_if_noexcept](https://en.cppreference.com/w/cpp/utility/move_if_noexcept) | [[20-languages/cpp/deep-dives/Deep Dive - Destination First Construction Across RVO Emplace and Placement New]], [[20-languages/cpp/object-resource-semantics/Concept - In-place Construction and emplace]] |
| Factory lambda is not syntax sugar because it delays materialization. | `Needs Nuance` | Correct as design pattern, but only if API truly accepts a recipe and constructs into final storage path. `T(factory())` can still require constructibility from the factory result; non-movable same-type cases need careful examples. | [cppreference construct_at](https://en.cppreference.com/w/cpp/memory/construct_at), [cppreference copy elision](https://en.cppreference.com/w/cpp/language/copy_elision.html) | [[20-languages/cpp/deep-dives/Deep Dive - Factory Lambda Is Not Syntax Sugar]], [[20-languages/cpp/object-resource-semantics/Concept - Factory Lambda and Delayed Construction]] |
| `noexcept` move affects vector reallocation decisions. | `Correct` | Use precise condition: `move_if_noexcept` returns `T&&` if nothrow move constructible or not copy constructible; otherwise it returns `const T&` so copy can preserve strong exception guarantee. | [cppreference move_if_noexcept](https://en.cppreference.com/w/cpp/utility/move_if_noexcept), [cppreference move constructor](https://en.cppreference.com/w/cpp/language/move_constructor.html), [Core Guidelines C.66](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines?lang=en#Rc-move-noexcept) | [[20-languages/cpp/deep-dives/Deep Dive - Buffer Bug RAII Rules and noexcept Move]], [[20-languages/cpp/object-resource-semantics/Concept - noexcept Move and Container Reallocation]] |
| Rule of 0 / 3 / 5 follows from resource ownership. | `Correct` | Add Core Guidelines framing: prefer Rule of Zero; if defining/deleting any default operation, consider the full set and consistency. | [Core Guidelines C.20/C.21/C.22](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines?lang=en#Rc-zero), [Core Guidelines R.1](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines?lang=en#Rr-raii) | [[20-languages/cpp/deep-dives/Deep Dive - Buffer Bug RAII Rules and noexcept Move]], [[20-languages/cpp/object-resource-semantics/Concept - RAII and Rule of 0 3 5]] |
| `object is not just data` is a good thesis. | `Needs Nuance` | Good teaching thesis, but formal C++ object properties are not identical to ownership/invariants. Phrase as: for non-trivial C++ types, object semantics include lifetime and operations; ownership/invariants are type-design semantics layered on top. | [intro.object](https://eel.is/c%2B%2Bdraft/intro.object), [basic.life](https://eel.is/c%2B%2Bdraft/basic.life), [Core Guidelines R.1](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines?lang=en#Rr-raii) | [[20-languages/cpp/deep-dives/Deep Dive - Object Not Just Data Report Thesis]], [[20-languages/cpp/object-resource-semantics/Concept - Object Delivery - Copy Move In-place Construction]] |
| C++ semantic lifting means C conventions become type/lifetime/operation semantics. | `Correct` | Keep concrete examples: owner pointer -> `unique_ptr`; pointer+length view -> `span`; tagged union -> `variant`; optional active object -> `optional`. Avoid presenting C++ as universally safer without tradeoff. | [Core Guidelines R.1](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines?lang=en#Rr-raii), [Core Guidelines R.20](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines?lang=en#Rr-owner), [cppreference optional](https://en.cppreference.com/w/cpp/utility/optional) | [[20-languages/cpp/deep-dives/Deep Dive - C Convention to Cpp Semantic Lifting]], [[20-languages/cpp/object-resource-semantics/Concept - C vs C++ Semantic Lifting]] |
| Type as operations/invariants maps well to math / algebra. | `Needs Nuance` | Useful as generic-programming lens, especially operation requirements and laws. Do not imply every C++ type literally forms an algebraic structure. Include cost model, mutation, lifetime, exception safety. | [Core Guidelines T.20](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines?lang=en#Rt-low), [cppreference regular](https://en.cppreference.com/w/cpp/concepts/regular), [cppreference semiregular](https://en.cppreference.com/w/cpp/concepts/semiregular) | [[20-languages/cpp/deep-dives/Deep Dive - Type Operations Laws Invariants and Stepanov]], [[20-languages/cpp/object-resource-semantics/Concept - C++ Type as Operations and Invariants]] |
| Rust is C++ ownership made stricter by compiler. | `Needs Nuance` | Good bridge, but incomplete. Rust ownership/borrow rules reshape API signatures and aliasing/mutation discipline; Rust move is not C++ user-defined move constructor. | [Rust Book - ownership](https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html), [Rust Book - borrowing](https://doc.rust-lang.org/book/ch04-02-references-and-borrowing.html), [Rust Reference - references](https://doc.rust-lang.org/reference/types/pointer.html) | [[20-languages/cpp/deep-dives/Deep Dive - Ownership Design Space Cpp Rust GC]], [[20-languages/rust/Concept - Rust Ownership Compared With C++ Move]] |
| ABI return slot explains how RVO really works. | `Mental Model Only` | Useful systems model, but actual calling convention is implementation/ABI-specific. C++ standard specifies initialization of result object and copy elision permissions, not a concrete hidden pointer ABI. | [stmt.return](https://eel.is/c%2B%2Bdraft/stmt.return), [class.copy.elision](https://eel.is/c%2B%2Bdraft/class.copy.elision), [intro.object](https://eel.is/c%2B%2Bdraft/intro.object) | [[30-systems/abi-function-call/Concept - Function Call Stack and Return Object Storage]], [[20-languages/cpp/deep-dives/Deep Dive - Half RVO Misconception Storage and Lifetime]] |

## Immediate Fix Queue

Status: applied to Deep Dive / Concept layer on 2026-05-09.

1. Applied: Add C++23 return move-eligible nuance to:
   - [[20-languages/cpp/deep-dives/Deep Dive - std move xvalue and Return Path Selection]]
   - [[20-languages/cpp/object-resource-semantics/Concept - std move vs Move Constructor]]
2. Applied: Tighten `emplace` wording:
   - `emplace(args...)` can construct the new element in-place.
   - Existing elements can still move/copy on container reallocation.
   - Passing an already materialized `T` to `emplace` does not undo that materialization.
3. Applied: Tighten factory lambda examples:
   - Keep the principle: pass recipe, not premature `T`.
   - Add condition: same-type non-movable factory examples need direct-initialization path that does not require a move from an intermediate `T`.
4. Applied: Add moved-from object nuance:
   - valid but unspecified for standard library types.
   - functions with preconditions still require those preconditions.
5. Applied: Mark ABI return slot as implementation model:
   - useful for systems understanding.
   - not a standard-mandated source transformation.

## Extension Queue

| Extension | Why It Matters | Candidate Note |
| --- | --- | --- |
| C++23 implicit move from local variables | Prevent outdated explanation of `return local;` fallback path. | `Deep Dive - C++23 Return Move-Eligible Expressions` or section in `std move xvalue` Deep Dive |
| `std::move_if_noexcept` and exception guarantees | Makes `noexcept move` concrete instead of slogan. | `Concept - noexcept Move and Container Reallocation` extension |
| `std::optional::emplace` vs `vector::emplace_back` | Both say `emplace`, but storage and reallocation behavior differ. | `Deep Dive - Destination First Construction...` section |
| `construct_at(factory())` edge cases | Prevent overclaiming factory lambda with non-movable types. | `Deep Dive - Factory Lambda Is Not Syntax Sugar` section |
| Regular / semiregular / Stepanov | Turns math analogy into C++20 concepts and generic-programming requirements. | `Reading - Stepanov Generic Programming and Algebraic Structures` |
| Rust ownership vs C++ move | Keep design-space comparison precise. | Rust concept extension or separate reading note |

Applied extension updates on 2026-05-10:

- `std::move_if_noexcept` decision rule added to [[20-languages/cpp/object-resource-semantics/Concept - noexcept Move and Container Reallocation]] and [[20-languages/cpp/deep-dives/Deep Dive - Buffer Bug RAII Rules and noexcept Move]].
- Rust API surface comparison added to [[20-languages/rust/Concept - Rust Ownership Compared With C++ Move]] and enforcement map added to [[20-languages/cpp/deep-dives/Deep Dive - Ownership Design Space Cpp Rust GC]].
- Regular / semiregular vocabulary added to [[20-languages/cpp/deep-dives/Deep Dive - Type Operations Laws Invariants and Stepanov]] and [[20-languages/cpp/object-resource-semantics/Concept - C++ Type as Operations and Invariants]].

## Done Definition For Calibration

A Deep Dive / Concept is calibrated when:

- it marks whether a statement is standard semantics, library behavior, guideline, or implementation model;
- it includes version-sensitive notes for C++17 / C++20 / C++23 where relevant;
- it does not treat analogies such as return slot / placement new / factory lambda as identical mechanisms;
- it preserves the original Conversation Note question that motivated the claim;
- it links to at least one external source when the claim depends on standard/library behavior.

## Sources

- [[20-languages/cpp/External Sources - C++ Object Semantics]]
- [C++ draft - class.copy.elision](https://eel.is/c%2B%2Bdraft/class.copy.elision)
- [C++ draft - stmt.return](https://eel.is/c%2B%2Bdraft/stmt.return)
- [C++ draft - basic.life](https://eel.is/c%2B%2Bdraft/basic.life)
- [C++ draft - intro.object](https://eel.is/c%2B%2Bdraft/intro.object)
- [cppreference - copy elision](https://en.cppreference.com/w/cpp/language/copy_elision.html)
- [cppreference - return statement](https://en.cppreference.com/w/cpp/language/return.html)
- [cppreference - value categories](https://en.cppreference.com/w/cpp/language/value_category)
- [cppreference - std::move](https://en.cppreference.com/w/cpp/utility/move)
- [cppreference - std::move_if_noexcept](https://en.cppreference.com/w/cpp/utility/move_if_noexcept)
- [C++ Core Guidelines](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines?lang=en)
- [Rust Book - Ownership](https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html)
- [Rust Book - References and Borrowing](https://doc.rust-lang.org/book/ch04-02-references-and-borrowing.html)
- [Rust Reference - Pointer Types](https://doc.rust-lang.org/reference/types/pointer.html)
