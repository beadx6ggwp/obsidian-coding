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

Updated macro-structure:

```text
return local object
-> copy worry
-> copy means semantic duplication, not byte movement
-> C Buffer semantic loss
-> C++ Buffer type operations
-> RAII / Rule of 0/3/5
-> move appears when copy is wrong, expensive, or impossible
-> std::move / value categories
-> return by value / RVO revisited as no-transfer case
-> C convention to C++ type semantics
-> type invariants
```

Remaining caution:

- Ch1 should start from `T make() { T x; return x; }`, not from a named list of RVO rules.
- RVO should be the hook/question, not the whole subject.
- C `Buffer` should not be the first scene, but it should appear before move becomes the main answer.
- Move should be motivated by copy being wrong, expensive, or impossible; it should not be introduced as an early taxonomy item.
- Value categories must appear only after `std::move` becomes necessary.
- Raw storage / emplace should be a supporting interlude, not the main bridge after move ownership.
- RAII, copy/move, and RVO must not be presented as the same layer. They are lifetime, transfer, and construction-placement axes inside the same object/resource semantics frame.
- C semantic gap should remain the reveal, but the path must prepare it through return-by-value, copy semantics, Buffer, C++ Buffer, move, and RAII.

## Semantic Axes - 2026-05-11

RAII、copy / move、RVO 不是同一個概念層級。它們屬於同一個大框架：C++ object / resource semantics，但分別處理不同問題。

```text
Lifetime semantics:
    RAII
    constructor acquires
    destructor releases
    resource lifetime follows object lifetime

Transfer semantics:
    copy
        duplicate / share / view / delete
    move
        ownership transfer
        source remains valid moved-from object

Construction placement semantics:
    RVO / NRVO
    copy elision
    emplace / placement new

Validity / invariant:
    valid state
    moved-from state
    ownership invariant
```

Teaching consequence:

```text
RAII 先讓 object 擁有 resource。
copy / move 再回答這個 RAII object 被交付時，ownership 怎麼保持正確。
RVO / in-place construction 則回答 source object 能不能根本不獨立存在。
```

Rule of 0/3/5 的本質也應該放在這個分層下講：

```text
只要 type 進入 RAII resource-owning 狀態，
copy / move / destroy 就必須一起維護同一個 ownership invariant。
```

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

這個 frame 不應該放在教學最前面當抽象 thesis lecture。正式開場應該先用一段最普通的 return-by-value code，讓讀者自然問「caller 到底怎麼拿到 `T`」：

```cpp
T make() {
    T x;
    return x;
}

T y = make();
```

之後才把整條線收束成 copy / move / in-place construction 的 object delivery frame；不要在開場就把這個分類表攤開。

然後每個後續主題都回來回答同一件事：

- return local：caller 如何得到 valid `T`？
- copy semantics：如果真的 copy，必須產生語意正確的新 object。
- C `Buffer`：representation copy 不等於 semantic copy。
- C++ `Buffer`：copy / destroy 必須成為 type operations。
- real move：copy wrong / expensive / impossible，且 source 可被放棄時，ownership transfer 才有意義。
- `std::move`：不是 move，而是允許 move operation 被選到。
- `return T{}` / RVO：source object 不必獨立存在時，可以直接形成 result object。
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

Active route:

```text
Prologue:
    I started from RVO.
    Keep that as the emotional entry point, but do not teach RVO as an isolated trick.

Part 1:
    Return by value as copy worry.
    `T make() { T x; return x; }`
    The first question is: if this copies, what does copy mean?

Part 2:
    Copy is not just cost.
    C Buffer shows representation copy is not semantic copy.
    C++ Buffer shows copy / destroy become type operations.

Part 3:
    If copy is wrong, move appears.
    Move is ownership transfer, not faster copy.
    `std::move` does not move.

Part 4:
    Return by value revisited.
    `return T{}` / `return x` / `return std::move(x)` can now be explained with the right model.
    RVO is the no-transfer case: sometimes there is nothing to move.

Part 5:
    Big reveal.
    C convention becomes C++ type semantics.
```

## Return-First Detailed Outline

Status:

```text
reactivated as current candidate
```

This route is now the better teaching candidate again. The C `Buffer`-first version was semantically clean, but it front-loaded too many concepts before the listener had one clear question to follow.

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

## Part 1 - Return By Value: Why Are We Worried About Copy?

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
先建立 copy 焦慮。
不要一開始就丟 C++17 prvalue / `return T{}` / NRVO / xvalue。
```

## Ch2 - Copy Is Not Just Moving Bytes

Core question:

```text
如果 B 從 A copy 一個 T，
B 得到的是什麼？
```

Teaching purpose:

```text
建立：
copy means semantic duplication,
not merely representation duplication.

這章不要講 move。
先讓讀者知道 copy 可能很貴，也可能不合理。
```

## Part 2 - Copy Is Not Just Cost

## Ch3 - C Buffer: Meaning Lives In Convention

Cold open:

```c
typedef struct {
    char* ptr;
    size_t size;
} Buffer;

Buffer a = buffer_create(1024);
Buffer b = a;
```

Purpose:

```text
展示 C 可以完成 resource 管理，
但 ownership / copy / lifetime 語意常常只存在 programmer convention / 文件 / 命名裡。

Buffer b = a 複製了 representation，
但沒有告訴讀者 ownership semantics 是否被保留。
```

## Ch4 - Deep Copy, Deleted Copy, And Type Design Pressure

Purpose:

```text
補上 move 出現前的真正壓力：

shallow copy:
    cheap, but wrong for unique ownership.

deep copy:
    semantically correct if both objects need independent buffers,
    but may be expensive.

deleted copy:
    semantically honest if duplication should not exist.
```

## Ch5 - C++ Buffer: Copy / Destroy Become Type Operations

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
copy / destroy 是 resource semantics。
```

## Part 3 - If Copy Is Wrong, Move Appears

## Ch6 - Why Move Exists

Core slogan:

```text
copy means duplication.
move means transfer.
```

Need to explain:

- move is not just because deep copy is expensive;
- copy may be expensive, semantically wrong, or impossible;
- move only makes sense when the source can be abandoned.

## Ch7 - Move Is Ownership Transfer, Not Faster Copy

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

## Ch8 - `std::move` Does Not Move

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

## Ch9 - Value Categories: lvalue, xvalue, prvalue, rvalue

Core slogan:

```text
Value category is not lifetime.
```

Need to explain:

- expression has type and value category;
- lvalue is not "long lifetime";
- rvalue is not "short lifetime";
- xvalue means expiring glvalue, not "already moved";
- prvalue is not simply "temporary object" after C++17.

## Ch10 - RAII And Rule Of 0/3/5

Core slogan:

```text
Rule of 0/3/5 is ownership pressure, not a checklist.
```

## Part 4 - Return By Value Revisited

```text
回到開頭，但現在讀者已經有 copy / move / in-place / ownership 的模型。
這時才細分 `return T{}`、`return x`、`return std::move(x)`。
```

## Ch11 - `return T{}`: Sometimes There Is Nothing To Move

```text
RVO means sometimes there is nothing to move.
```

## Ch12 - `return x` vs `return std::move(x)`

Core slogan:

```text
Do not move from a local return value just to be helpful.
```

## Part 5 - The Big Reveal: From C Convention To C++ Type Semantics

## Ch13 - From C Convention To C++ Semantic Lifting

This is the elevation point.

Core reveal:

```text
前面看起來是很多獨立規則：
    RVO
    copy
    move
    C Buffer semantic loss
    C++ Buffer type operations
    RAII
    Rule of 0/3/5

但它們其實都在處理同一件事：
    把 C 裡靠 convention 維護的 resource semantics
    提升到 C++ object/type operations 裡。
```

## Ch14 - Type As Operations And Invariants

```text
A type is not just a layout.
```

## Extensions

## E1 - `noexcept` Move And Containers

Status:

```text
extension / library contract case study
```

Reason:

```text
它很重要，但會把主線從 object delivery / ownership semantics
拉到 generic containers 和 exception guarantee。
```

Need to explain:

- vector reallocation;
- `std::move_if_noexcept`;
- strong exception guarantee;
- why resource-owning move constructor should often be `noexcept`.

## E2 - Regularity, Concepts, And Generic Programming

Status:

```text
extension / generic programming endpoint
```

Reason:

```text
主線只需要講到 type 不只是 layout。
regularity / concepts 是下一層：
generic algorithm 如何描述 operation requirements。
```

## E3 - Factory Lambda And Delayed Construction

Status:

```text
advanced / revisit later
```

Reason:

```text
目前還沒完全掌握，不應該強放主線。
```

## E4 - ABI Return Slot

Status:

```text
systems appendix
```

Boundary:

```text
useful implementation model,
not standard source transformation.
```

## E5 - Rust Ownership As Comparison

Status:

```text
oral aside / appendix
```

Purpose:

```text
用來對照 ownership enforcement layer，
不拉走 C++ 主線。
```

## E6 - CppCon Watchlist

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
- `Rule of 0/3/5 is ownership pressure, not a checklist.`
- `A type is not just a layout.`
- `C++ lifts resource conventions into type operations.`

## Open Decisions

These still need judgment before writing the first full chapter:

- Whether Ch1 should be called `How Many Objects Are In This Code?` or `Objects In Transit`.
- Whether `Object Delivery` should appear as the first explicit term, or be revealed after the three return examples.
- How deep Ch5 should go into `glvalue` before overwhelming the reader.
- Whether `return T{}` belongs in Ch1 or waits until the RVO revisited chapter.
- Whether factory lambda remains extension permanently or becomes an advanced chapter later.
- Whether the first full chapter should be `How Many Objects Are In This Code?` instead of `Object Delivery`.

## Next Action

The official [[20-languages/cpp/Learning Path - C++ Object and Resource Semantics|Learning Path]] now reflects the talk-style plan. The next step is to write Chapter 1, but keep it narrow:

```text
Chapter 1 should ask the object-count / return-local question.
It should not yet fully teach prvalue, xvalue, or std::move.
```
