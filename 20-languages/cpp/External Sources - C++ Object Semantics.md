# External Sources - C++ Object Semantics

## Purpose

這份 note 是 `ChatGPT-CPP RVO 解釋` 這組 Deep Dive 的外部來源地圖。用途不是取代 Conversation Note，而是校準：

- 原始對話裡哪些說法符合標準 / reference。
- 哪些說法需要加上版本差異或限制。
- 哪些地方可以用 CppCon / Core Guidelines 補成工程實務。

## Source Priority

1. ISO draft / standard wording：用來校準語言規則。
2. cppreference：用來快速查語意、版本差異、例子。
3. C++ Core Guidelines：用來連到 API design / engineering rule。
4. CppCon talks / materials：用來補教學脈絡、實務思考方式。

## Core Language Sources

### Copy Elision / NRVO / Return Slot

- [C++ draft - class.copy.elision](https://eel.is/c%2B%2Bdraft/class.copy.elision)
  - 用途：校準 copy/move elision 的標準文字。
  - Key takeaways:
    - 在符合條件時，implementation 可以省略 source object 的建立。
    - source 和 target 會被視為同一個 object 的兩種 referring ways。
    - return named local 的 NRVO 條件明確排除 function parameter / handler parameter。
    - constant expression / constant initialization 不能套 copy elision。

- [cppreference - copy elision](https://en.cppreference.com/w/cpp/language/copy_elision.html)
  - 用途：整理 C++17 prvalue / guaranteed copy elision 的版本差異。
  - Key takeaways:
    - C++17 後 prvalue 不必先 materialize temporary，而是可直接建到 final destination。
    - `return T{}` 這類同型 prvalue return 不需要 copy/move constructor 可用。
    - cppreference 特別提醒：C++17 這種情況不只是普通 optimization，因為沒有 source temporary 可以 move from。

- [cppreference - return statement](https://en.cppreference.com/w/cpp/language/return.html)
  - 用途：校準 return statement、automatic move、guaranteed copy elision。
  - Key takeaways:
    - prvalue return 且型別符合時，result object 直接由該 expression 初始化。
    - C++23 對 move-eligible expression 有更簡化的 implicit move rule。

### Value Category / `std::move`

- [cppreference - value categories](https://en.cppreference.com/w/cpp/language/value_category)
  - 用途：校準 lvalue / xvalue / prvalue。
  - Key takeaways:
    - expression 有 type，也有 value category。
    - `std::move(x)` 這種回傳 rvalue reference 的 function call 是 xvalue。
    - C++17 之後 prvalue 和 temporary materialization 的關係改變，prvalue 不再是「一定先有 temporary object」。

- [cppreference - std::move](https://en.cppreference.com/w/cpp/utility/move)
  - 用途：校準 `std::move` 本質。
  - Key takeaways:
    - `std::move` 產生 xvalue expression。
    - 它等價於 cast 到 rvalue reference type。
    - 真正是否搬 resource 取決於被選到的 move constructor / move assignment / overload。

- [cppreference - std::move_if_noexcept](https://en.cppreference.com/w/cpp/utility/move_if_noexcept)
  - 用途：校準 standard library 在強 exception guarantee 下何時 move、何時 copy。
  - Key takeaways:
    - `std::move_if_noexcept(x)` 會在 move constructor 是 `noexcept` 或 type 不可 copy 時回傳 rvalue reference。
    - 否則它可能回傳 const lvalue reference，讓 container reallocation 等流程選擇 copy 以維持 exception guarantee。

### Object / Lifetime / Storage

- [cppreference - object](https://en.cppreference.com/w/cpp/language/objects)
  - 用途：校準 C++ object 的 formal properties。
  - Key takeaways:
    - C++ object 有 size、alignment、storage duration、lifetime、type、value，並且 optionally 有 name。
    - 這支持「object 不只是 data layout」，但 ownership / invariant 屬於 type design layer，不是每個 object 的 formal property。

- [cppreference - lifetime](https://en.cppreference.com/w/cpp/language/lifetime.html)
  - 用途：校準 storage 與 object lifetime 的差異。
  - Key takeaways:
    - object lifetime begins when proper storage is obtained and initialization completes。
    - class object 的 lifetime 通常在 destructor call starts 時結束。
    - object lifetime is equal to or nested within storage lifetime。
    - storage reuse / placement construction 有嚴格規則，不能把 raw bytes 和 active object 混成同一件事。

- [cppreference - new expression / placement new](https://en.cppreference.com/w/cpp/language/new)
  - 用途：校準 placement new 與 object creation。
  - Key takeaways:
    - placement form 可以在指定 placement arguments 對應的位置建立 object。
    - 這是 manual storage / lifetime control 的語言入口之一。

- [cppreference - std::construct_at](https://en.cppreference.com/w/cpp/memory/construct_at)
  - 用途：校準 C++20 library-level placement construction helper。
  - Key takeaways:
    - `construct_at(location, args...)` 在指定 address 建立 `T` object。
    - 它大致等價於 placement new 加上 perfect forwarding，並有 constexpr 相關規則。
    - 注意：若討論的是 factory 回傳完整 same-type `T` 且 `T` 不可 move，`construct_at(ptr, factory())` 可能不等同於直接 placement-new initializer；要檢查是否會先經過 function argument materialization。

- [cppreference - std::optional::emplace](https://en.cppreference.com/w/cpp/utility/optional/emplace)
  - 用途：支撐 in-place construction / wrapper-owned storage。
  - Key takeaways:
    - `emplace` 在 optional 的 contained value 位置 in-place construct。
    - 若 optional 原本有值，會先 destroy 原 contained value，再直接建新值。

- [cppreference - constructors](https://en.cppreference.com/w/cpp/language/constructor)
  - 用途：校準 constructor 的角色。
  - Key takeaways:
    - constructor 是 class type 的 non-static member function，用來 initialize object。

- [cppreference - destructors](https://en.cppreference.com/w/cpp/language/destructor)
  - 用途：校準 destructor 的角色。
  - Key takeaways:
    - destructor 在 object lifetime ends 時呼叫，目的通常是釋放 lifetime 期間取得的 resource。

### Concepts / Regular Types

- [cppreference - constraints and concepts](https://en.cppreference.com/w/cpp/language/constraints.html)
  - 用途：校準 concepts 作為 template argument requirements。
  - Key takeaways:
    - concept 是 compile-time predicate，會成為 template interface 的一部分。
    - 這支持「type requirements 不只是 syntax，而是 interface contract」。

- [cppreference - std::regular](https://en.cppreference.com/w/cpp/concepts/regular)
  - 用途：校準 regular type。
  - Key takeaways:
    - `std::regular<T>` = `std::semiregular<T> && std::equality_comparable<T>`。
    - 它描述像 built-in types 一樣可 copy、default construct、equality compare 的 type。

- [cppreference - std::semiregular](https://en.cppreference.com/w/cpp/concepts/semiregular)
  - 用途：校準 semiregular type。
  - Key takeaways:
    - `std::semiregular<T>` = `std::copyable<T> && std::default_initializable<T>`。
    - 這和 Stepanov / generic programming 的 operation requirement 思考方式直接相關。

### Library Semantic Wrappers

- [cppreference - std::unique_ptr](https://en.cppreference.com/w/cpp/memory/unique_ptr)
  - 用途：支撐 ownership 被 type 化的例子。
  - Key takeaways:
    - `unique_ptr` owns and manages another object through a pointer and disposes it when appropriate。
    - `unique_ptr` 是 MoveConstructible / MoveAssignable，但不是 CopyConstructible / CopyAssignable。

- [cppreference - std::span](https://en.cppreference.com/w/cpp/container/span)
  - 用途：支撐 pointer + length 被 type 化的例子。
  - Key takeaways:
    - `span` 是 object sequence 的 non-owning view，將 pointer + extent 的語意放進 library type。

- [cppreference - std::variant](https://en.cppreference.com/w/cpp/utility/variant)
  - 用途：支撐 active alternative / tagged union 語意。
  - Key takeaways:
    - `variant` 是 type-safe union，持有 alternatives 之一；這對應 C union active member convention 的 type 化。

## Engineering Guidelines

### Return Values / Out Parameters

- [C++ Core Guidelines - F.20](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines?lang=en#Rf-out)
  - 用途：支撐「return-by-value 是 API design，不只是 compiler trick」。
  - Key takeaways:
    - out value 優先用 return value，不優先用 output parameter。
    - 大 object 也包含在內，因為 standard containers 可依賴 implicit move / optimization 避免顯式 memory management。

### `return std::move(local)`

- [C++ Core Guidelines - F.48](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines?lang=en#Rf-return-move-local)
  - 用途：校準「不要 return std::move(local)」。
  - Key takeaways:
    - explicit `std::move` 會 prevent RVO，通常是 pessimization。
    - `return result;` 是 named RVO at best, move construction at worst。

- [C++ Core Guidelines - ES.56](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines?lang=en#Res-move)
  - 用途：校準何時該顯式 `std::move`。
  - Key takeaways:
    - 只有需要明確把 object 移到另一個 scope 時才寫 `std::move`。
    - `std::move` 是 disguised cast；它本身不搬東西。
    - 不要因為聽說比較有效率就亂寫 `std::move`。

### RAII / Rule of Zero / Move Noexcept

- [C++ Core Guidelines - R.1](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines?lang=en#Rr-raii)
  - 用途：校準 RAII / resource handle。
  - Key takeaways:
    - resource 應該交給 object handle 管理，透過 constructor / destructor 對稱處理 acquire / release。
    - 這讓 manual resource convention 進入 type semantics。

- [C++ Core Guidelines - C.20 / C.21 / C.22](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines?lang=en#Rc-zero)
  - 用途：校準 Rule of Zero / default operations。
  - Key takeaways:
    - 能避免自定義 default operations 就避免。
    - 如果定義或刪除 copy / move / destructor 任一個，應考慮整組操作。
    - default operations 是一組彼此相關的 lifecycle semantics。

- [C++ Core Guidelines - C.66](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines?lang=en#Rc-move-noexcept)
  - 用途：校準 `noexcept move` 與 library efficiency。
  - Key takeaways:
    - move operations 應該 `noexcept`。
    - non-throwing move 會被 standard library / language facilities 更有效地使用。

### Regular Types / Concepts

- [C++ Core Guidelines - C.11](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines?lang=en#Rc-regular)
  - 用途：支撐 concrete type 應盡量 regular 的設計方向。
  - Key takeaways:
    - regular types 比 irregular types 更容易理解與推理。
    - resource handle 可能是 move-only，因此不一定 regular；這點可避免數學類比過度套用。

- [C++ Core Guidelines - T.20](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines?lang=en#Rt-low)
  - 用途：校準 concepts 不只是 syntactic constraints。
  - Key takeaways:
    - concept 應表達 meaningful semantics，例如 number、range、totally ordered。
    - 只有 `has +` 這類 syntax check 不足以構成好的 concept。

## CppCon / Learning Sources

- [CppCon 2019 - Back to Basics Track Announced](https://cppcon.org/b2b2019/)
  - 用途：定位 Back to Basics 系列，包括 move semantics、value categories、RAII / Rule of Zero。

- [CppCon 2019 - Back to Basics: Move Semantics, Klaus Iglberger](https://www.youtube.com/watch?v=St0MNEU5b0o)
  - 用途：補 move semantics、rvalue reference、`std::move`、move constructor / move assignment 的教學脈絡。

- [CppCon 2019 materials repository](https://github.com/CppCon/CppCon2019)
  - 用途：找 talk slides / code material。

- [CppCon 2019 - Understanding Object Lifetime class archive](https://cppcon.org/class-2019-obj-lifetime/)
  - 用途：支撐 object lifetime 是 C++ 核心主線；課程涵蓋 RAII、standard wording、return values、lambda captures、C++17 變化等。

## Rust Sources

- [The Rust Programming Language - What Is Ownership?](https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html)
  - 用途：校準 Rust ownership / move / drop model，不把它簡化成「C++ 比 Rust 更嚴格」。
  - Key takeaways:
    - Rust ownership rules 用 compiler 檢查 single owner、scope exit drop、move invalidates previous binding 等規則。
    - 這和 C++ RAII 有對應關係，但 Rust 把更多 resource state transition 放進 type system 與 borrow checker。

- [The Rust Programming Language - References and Borrowing](https://doc.rust-lang.org/book/ch04-02-references-and-borrowing.html)
  - 用途：校準 borrow / mutable borrow 的限制，作為 C++ reference / pointer / lifetime convention 的對照。
  - Key takeaways:
    - borrowing 允許 reference 暫時使用 value 而不取得 ownership。
    - mutable reference 與 immutable reference 的 aliasing 規則是 Rust memory safety model 的核心之一。

- [Rust Reference - Pointer Types](https://doc.rust-lang.org/reference/types/pointer.html)
  - 用途：補 Rust references / raw pointers 的 formal reference，避免只用 book-level intuition。
  - Key takeaways:
    - shared reference 指向由其他 value owned 的 value。
    - mutable reference 允許 mutation，但 aliasing / validity 規則是 Rust memory model 的一部分。

## How To Use In Deep Dive

每篇 Deep Dive 應該新增：

- `External Source Check`: 外部來源支持哪些說法。
- `Corrections And Extensions`: 原始對話中哪些地方要更精準。
- `Further Reading`: 連到這份 source map 和具體來源。

Deep Dive 的角色：

```text
Conversation Note:
    原始問題與探索脈絡

Deep Dive:
    根據 Conversation Note 統整，再用外部來源校準與延伸

Concept:
    從校準後的 Deep Dive 抽成 quick lookup
```
