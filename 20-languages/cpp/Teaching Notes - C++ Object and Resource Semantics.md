# Teaching Notes - C++ Object and Resource Semantics

## Purpose

這份 note 用來保存目前對「教學編排、表達方式、目錄重心」的想法。

它不是正式 `Learning Path`，也不是完整章節教材。它的用途是：

- 記錄目前對章節順序的判斷。
- 記錄哪些主題應該主線化、哪些應該降級成 extension。
- 記錄想參考 CppCon / Arthur O'Dwyer 風格的表達方式。
- 避免之後寫正式章節時又回到「名詞列表式」教學。

Related:

- [[20-languages/cpp/Teaching Map - C++ Object and Resource Semantics]]
- [[20-languages/cpp/Learning Path - C++ Object and Resource Semantics]]
- [[20-languages/cpp/Knowledge Audit - C++ Object and Resource Semantics]]

## Current Teaching Direction

目前比較合理的方向不是教科書式：

```text
先定義 RVO
再定義 prvalue
再定義 std::move
再定義 RAII
```

而是 talk-style：

```text
先丟 code
-> 讓 naive model 失敗
-> 觀察現象
-> 補 vocabulary
-> 再上升到設計原則
```

也就是：

```text
code first
question first
wrong intuition first
terminology later
```

## Audience Assumption

目標讀者不是完全沒寫過 C++ 的人。

比較像：

```text
懂基本 C++ 語法
知道 class / constructor / destructor / reference / STL
但不熟 object lifetime、value category、move semantics、copy elision、RAII 背後的統一模型
```

所以教學不該太慢地介紹語法，但也不能直接跳進 standard wording。

合理節奏：

```text
具體 code
-> 問它會發生什麼
-> 打破常見直覺
-> 給 mental model
-> 補正式名詞
-> 連回 standard/library/guideline
```

## Style Reference

想參考的方向是 Arthur O'Dwyer / CppCon Back to Basics 那種講法：

- 一開始給小 code，不先鋪大理論。
- 用一句清楚的 slogan 打掉錯誤模型。
- 用實例逼出語言規則，而不是先背名詞。
- 把 C++ 的坑講成「這個直覺哪裡不夠精準」。
- 保留精準語言，但讓 vocabulary 出現在讀者需要它的時候。

Useful references:

- [Arthur O'Dwyer - Value category is not lifetime](https://quuxplusone.github.io/blog/2019/03/11/value-category-is-not-lifetime/)
- [Arthur O'Dwyer - The Superconstructing Super Elider](https://quuxplusone.github.io/blog/2018/03/29/the-superconstructing-super-elider/)
- [CppCon 2019 Back to Basics Track](https://cppcon.org/b2b2019/)

## Current Decisions

## Narrative Review - 2026-05-10

目前整體教學脈絡應該採用「逐步冒出疑問」而不是「一口氣展示所有 return case」。

Key finding:

```text
讀者不是先想知道 prvalue / xvalue / NRVO。
讀者最自然的第一個問題是：
    我在 function 裡建立一個 local object，然後 return 它，
    這到底會發生幾次建構 / copy / move？
```

因此第一章只應該出現最自然的 named-local return：

```cpp
T make() {
    T x;
    return x;
}
```

第一章不應該一開始就出現：

- `return T{}`
- `return std::move(x)`
- `C++17 prvalue`
- `xvalue`
- `guaranteed copy elision`

這些都不是錯，但它們都需要前置動機。

Revised question flow:

```text
1. 我 return local object，會 copy 嗎？
2. 如果我不需要 local name，只是要直接產生 result，`return T{}` 又如何？
3. 如果我怕 `return x` 會 copy，能不能 `return std::move(x)` 保險？
4. 為什麼 `std::move` 不是 move？
5. 那 value category 到底在描述什麼？
6. move 真正有意義的情境是什麼？
```

這樣的順序比較符合「懂 C++ 但不熟深層語義」的聽眾。每個新名詞都不是硬塞，而是為了解釋上一個 code case。

Current macro-structure is sound:

```text
return local
-> return path variants
-> std::move misconception
-> value categories
-> move ownership
-> storage/lifetime
-> emplace
-> Buffer bug
-> RAII / Rule of 0/3/5
-> semantic lifting
-> type invariants / generic programming
```

Remaining caution:

- Ch1 must not try to fully teach RVO. It should only create the object-count / object-delivery question.
- Ch2 can name RVO / NRVO / prvalue, but only after the named-local case is already understood.
- Ch3 introduces `return std::move(x)` as a tempting wrong turn, not as an option the instructor recommends.
- Ch5 value categories must stay tied to the earlier examples; do not turn it into a standalone taxonomy lecture.
- Ch12 `C semantic gap` should remain the reveal, not be pulled forward again.

## Core Object Delivery Frame - 2026-05-10

這次重新抓到的核心是：RVO 不能被講成單一 compiler trick，move 也不能被講成「現代 C++ 就是到處 `std::move`」。

更好的 framing 是：

```text
A 要生成一個 T。
B 要怎麼拿到這個 T？
```

基本有三種策略：

```text
1. Copy
   A 有一份，B 也有一份。

2. Move
   A 不要了，由 B 接手 ownership。

3. In-place construction
   不要先在 A 製作再搬到 B；
   直接在 B 的 final storage 製作。
```

教學上的核心句：

```text
RVO 是「避免搬移」。
move 是「搬移不可避免時，便宜地轉移 ownership」。

現代 C++ 的重點不是到處 std::move，
而是讓物件盡量直接出生在它最後要待的位置。
```

這個 frame 應該放在 Learning Path 前段，但不能變成抽象 thesis lecture。它要被第一個 code case 逼出來：

```cpp
T make() {
    T x;
    return x;
}
```

然後每個後續主題都回來回答同一件事：

- `return T{}`：展示 direct result-object construction。
- `return x`：展示 named local / NRVO candidate / fallback move-copy。
- `return std::move(x)`：展示把 path 推向 move，可能破壞 in-place 機會。
- `std::move`：不是 move，而是允許 move operation 被選到。
- real move：source object 已存在、且 ownership 可以被轉移時才有意義。
- `emplace`：library/API 層面的 final-storage construction，但不是「保證沒有 move」。

## RVO Should Be A Hook, Not The Center

RVO 是原始學習入口，但正式教學不應該把它當成整個主軸。

Better framing:

```text
I only wanted to understand RVO.
But RVO exposed a deeper question:
How does C++ deliver objects?
```

RVO 在教學中的角色：

- cold open / motivating story;
- return-by-value case study;
- copy elision / object delivery 的局部案例。

RVO 不該承擔：

- 解釋整個 C++ object model；
- 當作 RAII / move / lifetime 的上位概念；
- 變成所有筆記的唯一入口。

## C Semantic Gap Should Be The Reveal, Not The Opening

一開始就講「C++ 解決 C 的語意缺口」會太抽象。

如果讀者還沒看過：

- shallow copy bug;
- moved-from object;
- storage vs lifetime;
- `std::move` 不搬東西;
- `emplace` 不是 magic;
- Rule of 0/3/5 的壓力；

那麼「C convention -> C++ semantic lifting」只會像 thesis statement。

比較好的方式：

```text
先讓讀者經歷局部問題
-> copy/move/lifetime/RAII 都看過
-> 最後回頭說：
   原來這整串是在把 C 裡靠 convention 維護的 resource semantics
   提升到 C++ object/type operations 裡。
```

所以 `C semantic gap` 應該放在後段，作為 Big Reveal。

## Value Categories Need A Main Track, But Not Too Early

`move`、`xvalue`、`prvalue`、`rvalue` 這條線很重要，但不能在一開始就用名詞壓過讀者。

尤其 `C++17 prvalue` 如果太早出現，會很突兀。

Better order:

```text
先看 code:

T make() {
    T x;
    return x;
}

先問:
    這段程式會產生幾個 T？
    return by value 一定 copy 嗎？
    local object `x` 會不會先在 callee stack 存在，再被搬到 caller？

再加入對照:

T make_direct() { return T{}; }

再問:
    如果沒有需要先命名 local object，
    `return T{}` 是否一定要先變成 temporary？

再自然引出:
    如果我怕 `return x` 會 copy，
    能不能寫 `return std::move(x)` 來保險？

再補 vocabulary:
    prvalue
    xvalue
    lvalue
    rvalue
    NRVO
    implicit move fallback
```

Key rule:

```text
phenomenon first,
terminology second.
```

Terminology correction:

```text
glvalue = lvalue or xvalue
rvalue  = prvalue or xvalue
```

There is no ordinary C++ value-category term `pvalue`; the intended term is usually `prvalue`.

## Factory Lambda Should Be Extension For Now

Factory lambda / delayed construction is valuable, but currently should not be in the main teaching path.

Reasons:

- It is an advanced application of destination-first construction.
- It depends on materialization timing, same-type prvalue, API boundary design, and sometimes address-stable operation state.
- If placed too early, it may blur the boundary between RVO, `emplace`, placement new, and factory APIs.
- The concept still needs more personal understanding before becoming teaching core.

Current placement:

```text
Extensions
  E1 - Factory lambda and delayed construction
```

Possible future promotion condition:

```text
promote to main path only after:
    non-movable object examples are clear
    same-type prvalue direct initialization is clear
    API boundary / materialization timing is clear
    factory result edge cases are understood
```

## Rust Should Be Appendix / Oral Aside

Rust comparison is useful, but it pulls the teaching away from the C++ object/resource main line.

Current placement:

```text
Appendix / oral aside:
    Rust ownership as another design point
```

Use Rust only to say:

```text
C++ puts many ownership/lifetime rules in RAII, type operations, and convention.
Rust moves more of that discipline into compiler-checked ownership and borrowing.
```

Do not make Rust a core chapter unless the goal changes from C++ teaching to cross-language ownership design.

## Current Talk-Style Outline

## Prologue - I Only Wanted To Understand RVO

Purpose:

```text
保留原始動機，但不要在這裡正式教完 RVO。
```

Opening line:

```text
我原本只是想知道：為什麼 return by value 不一定 copy？
但這個問題其實一路拉出了 C++ object/resource semantics。
```

## Part 1 - Objects In Transit

## Ch1 - How Many Objects Are In This Code?

Cold open:

```cpp
T make() {
    T x;
    return x;
}
```

Audience questions:

```text
這段程式會產生幾個 T？
return by value 一定 copy 嗎？
local object `x` 會不會先在 callee stack 存在，再被搬到 caller？
```

Teaching purpose:

```text
先建立 object delivery problem。
不要一開始就丟 C++17 prvalue / `return T{}` / NRVO / xvalue。
```

## Ch2 - Return Paths: Direct Construction, NRVO, Fallback Move

Core model:

```text
return T{}:
    direct construction of result object

return x:
    NRVO candidate
    fallback can move/copy
```

Vocabulary appears here:

- result object
- copy elision
- prvalue
- NRVO
- implicit move fallback

## Ch3 - Why `return std::move(local)` Can Be Worse

Core slogan:

```text
Do not move from a local return value just to be helpful.
```

Better opening question:

```text
如果我怕 `return x` 會 copy，
是不是可以寫 `return std::move(x)` 來避免 copy？
```

Purpose:

```text
把 return path 和 value category 接起來，
為 Part 2 的 std::move / xvalue 做鋪墊。
```

## Part 2 - Moving Without Moving

## Ch4 - `std::move` Does Not Move

Core slogan:

```text
std::move does not move.
```

Teaching path:

```text
std::move(x)
-> static_cast<T&&>(x)
-> xvalue expression
-> later operation may choose move constructor
```

## Ch5 - Value Categories: lvalue, xvalue, prvalue, rvalue

Core slogan:

```text
Value category is not lifetime.
```

Purpose:

```text
整理 expression category，不把它和 object lifetime 混在一起。
```

Need to explain:

- expression has type and value category;
- lvalue is not "long lifetime";
- rvalue is not "short lifetime";
- xvalue means expiring glvalue, not "already moved";
- prvalue is not simply "temporary object" after C++17.

## Ch6 - Move As Ownership Transfer

Cold comparison:

```cpp
struct Matrix4x4 {
    float m[16];
};

struct Buffer {
    char* ptr;
    size_t size;
};
```

Core slogan:

```text
Move is not faster copy.
Move is meaningful when ownership can be transferred.
```

## Part 3 - Where Objects Are Born

## Ch7 - Storage Is Not Lifetime

Core slogan:

```text
Raw storage is not an object.
```

Purpose:

```text
建立 storage / lifetime / object identity 的底層模型。
```

## Ch8 - In-place Construction And `emplace`

Core slogan:

```text
emplace is not magic.
It only helps when constructor arguments reach the final construction site.
```

Need boundaries:

- `emplace_back(args...)` can construct new element in-place;
- `emplace_back(T(args...))` already materialized a `T`;
- vector reallocation can still move/copy existing elements;
- `optional::emplace` has different storage behavior from `vector::emplace_back`.

## Part 4 - Resource Semantics

## Ch9 - The Buffer Bug

Cold open:

```cpp
class Buffer {
    char* ptr;
    size_t size;
public:
    ~Buffer() { delete[] ptr; }
};
```

Question:

```text
compiler-generated copy constructor 對嗎？
```

Purpose:

```text
讓讀者感受到 class 不是 fields + destructor。
copy/move/destroy 是 resource semantics。
```

## Ch10 - RAII And Rule Of 0/3/5

Core slogan:

```text
Rule of 0/3/5 is ownership pressure, not a checklist.
```

## Ch11 - `noexcept` Move And Containers

Core slogan:

```text
noexcept move is a promise to generic code.
```

Need to explain:

- vector reallocation;
- `std::move_if_noexcept`;
- strong exception guarantee;
- why resource-owning move constructor should often be `noexcept`.

## Part 5 - The Big Reveal

## Ch12 - From C Convention To C++ Semantic Lifting

This is the elevation point.

Core reveal:

```text
前面看起來是很多獨立規則：
    RVO
    move
    storage/lifetime
    RAII
    Rule of 0/3/5

但它們其實都在處理同一件事：
    把 C 裡靠 convention 維護的 resource semantics
    提升到 C++ object/type operations 裡。
```

C-side questions:

```text
這個 pointer 是 owner 還是 view？
誰 free？
copy 是 shallow 還是 deep？
這個 struct 能不能 bitwise copy？
```

C++-side mechanisms:

```text
constructor
destructor
copy constructor
move constructor
RAII
type invariant
```

## Ch13 - Type As Operations And Invariants

Core slogan:

```text
A type is not just a layout.
```

Model:

```text
Type =
    valid states
  + operations
  + invariants / laws
  + lifetime rules
  + exception safety
  + cost model
```

## Ch14 - Regularity, Concepts, And Generic Programming

Purpose:

```text
把 Stepanov / regular / semiregular 放在最後，
作為「type operations can be reasoned about」的延伸。
```

## Extensions

## E1 - Factory Lambda And Delayed Construction

Status:

```text
advanced / revisit later
```

Reason:

```text
目前還沒完全掌握，不應該強放主線。
```

## E2 - ABI Return Slot

Status:

```text
systems appendix
```

Boundary:

```text
useful implementation model,
not standard source transformation.
```

## E3 - Rust Ownership As Comparison

Status:

```text
oral aside / appendix
```

Purpose:

```text
用來對照 ownership enforcement layer，
不拉走 C++ 主線。
```

## E4 - CppCon Watchlist

Status:

```text
further reading
```

## Chapter Writing Rules

每章應該遵守：

1. Start with code or a concrete failure.
2. Ask the reader what they think happens.
3. State the naive model explicitly.
4. Break the naive model with one sharp example.
5. Only then introduce formal vocabulary.
6. End with three things the reader should now be able to answer.
7. Link back to the relevant Conversation Note so the original reasoning path is preserved.

## Slogan Bank

Potential chapter slogans:

- `Returning by value does not mean copying.`
- `std::move does not move.`
- `Value category is not lifetime.`
- `Move is not faster copy.`
- `Raw storage is not an object.`
- `emplace is not magic.`
- `Rule of 0/3/5 is ownership pressure, not a checklist.`
- `A type is not just a layout.`
- `C++ lifts resource conventions into type operations.`

## Open Decisions

These still need judgment before rewriting the official `Learning Path`:

- Whether Ch1 should be called `How Many Objects Are In This Code?` or `Objects In Transit`.
- Whether `Object Delivery` should appear as the first explicit term, or be revealed after the three return examples.
- How deep Ch5 should go into `glvalue` before overwhelming the reader.
- Whether factory lambda remains extension permanently or becomes an advanced chapter later.
- Whether the first full chapter should be `How Many Objects Are In This Code?` instead of `Object Delivery`.

## Next Action

The official [[20-languages/cpp/Learning Path - C++ Object and Resource Semantics|Learning Path]] now reflects the talk-style plan. The next step is to write Chapter 1, but keep it narrow:

```text
Chapter 1 should ask the object-count / return-local question.
It should not yet fully teach prvalue, xvalue, or std::move.
```
