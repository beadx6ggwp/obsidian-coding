# Learning Path - C++ Object and Resource Semantics

## Purpose

這份 note 是目前正式採用的學習路線草稿。

它不是完整教材，而是用來測試整場教學的「帶入節奏」：

```text
先給一段普通 C++ code
-> 讓讀者真的產生疑問
-> naive model 失敗
-> 才補 vocabulary
-> 最後把 vocabulary 收束成 C++ object/resource semantics
```

目前採用的風格來自 [[20-languages/cpp/Teaching Notes - C++ Object and Resource Semantics|Teaching Notes]]：

```text
code first
question first
wrong intuition first
terminology later
```

## Current Judgment

上一版從 C `Buffer` shallow copy 開場，語意上是對的，但教學帶入太重：

```text
C Buffer
-> RAII
-> copy / move
-> RVO
-> C semantic gap
```

聽眾還沒形成一個主問題，就被丟進很多層概念。

所以這版改回更自然的入口：

```cpp
T make() {
    T x;
    return x;
}

T y = make();
```

第一個問題不是「什麼是 RVO」，而是：

```text
x 在 function 裡。
y 在 caller 裡。

那 y 到底怎麼拿到這個 T？
```

這個入口比較好，因為它先建立一條單一主線：

```text
一個 T 要從某個地方交到另一個地方時，
C++ 到底有哪些語意可能？
```

RVO 仍然是入口，但不是主題。

真正主題是：

```text
C++ object 不只是 data。

一個 object 的建立、交付、轉移、銷毀，
都必須保留 lifetime、ownership、valid state、invariant 和 cost model。
```

## Teaching Thesis

這條路線最後要講的是：

```text
C++ object/resource semantics is a system for making object lifetime,
ownership, operations, and invariants explicit enough for programmers,
libraries, and compilers to reason about them.
```

但這句不適合當開場。

正式教學應該讓讀者先經歷：

```text
return by value 好像會 copy
-> 但實際上不一定
-> 那 object 到底怎麼被交付？
-> copy / move / in-place 是三種不同語意
-> copy / move 不只是效能問題
-> resource ownership 會讓 copy 失去語意
-> RAII / Rule of 0/3/5 / type invariant 開始變得必要
-> 回頭看，C++ 是在把 C convention 提升成 type semantics
```

## Semantic Axes

RAII、copy / move、RVO 不是同一個概念層級。它們在同一個大框架裡，但分別處理不同軸：

```text
C++ object / resource semantics

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

這個分層要在中後段才完整攤開。開場只要讓讀者先感覺：

```text
copy:
    A 有一份，B 也要一份。

move:
    A 已經有，B 接手，A 放棄。

in-place / RVO:
    A 那邊根本不需要獨立 object，
    T 直接在 B 的 final storage 形成。
```

## Route Shape

新版主線：

```text
return local object
-> object delivery question
-> copy / move / in-place as three possible stories
-> `std::move` and value categories
-> move as ownership transfer
-> Buffer shows copy/move are semantic operations, not byte tricks
-> RAII / Rule of 0/3/5
-> return by value / RVO revisited with the correct model
-> C convention to C++ type semantics
-> type invariants
```

這版的關鍵調整：

```text
RVO / return by value:
    開場用來產生問題。

Buffer:
    不再當第一幕。
    改成用來證明 copy / move 背後有 resource semantics。

C semantic gap:
    不放開頭當抽象 thesis。
    放在最後當 reveal。
```

## Prologue - I Started From RVO

保留原始動機，但不要一開始完整教 RVO。

Opening tone:

```text
我原本只是想知道：
為什麼 return by value 不一定 copy？

但這個問題後來一路拉出：
copy / move / ownership / lifetime / RAII / type invariants。
```

RVO 在這條路線裡的角色：

- 原始 hook；
- return-by-value 的具體疑問；
- object delivery 裡的 in-place case；
- 最後用來說明「有時候連 move 都不需要」。

RVO 不是：

- 整場教學的中心；
- 一開始要背的 compiler trick；
- RAII / ownership / move 的上位概念。

Reading:

- [[20-languages/cpp/Question Trail - ChatGPT CPP RVO 解釋]]
- [[20-languages/cpp/conversation-notes/Conversation Note - RVO Basics Return Slot and In-place Analogies]]

## Reader Position

這條路線假設讀者：

```text
會寫基本 C / C++
知道 class / constructor / destructor / reference / STL
聽過 copy / move / RVO 這些詞
但沒有把 lifetime、ownership、value category、RAII、type invariant 串成一個模型
```

所以開場不能是：

- `prvalue` / `xvalue` / `glvalue` 表格；
- C++17 standard wording；
- C vs C++ 大哲學；
- RAII / Rule of Five checklist。

第一個問題要普通到像是初學 C++ 時真的會問：

```text
我在 function 裡建立 local object，然後 return。
caller 那邊到底拿到的是什麼？
```

## Part 1 - Return By Value As The Doorway

這一段的任務不是教完 RVO，而是建立主問題：

```text
一個 object 要從 function 交到 caller 時，
到底是 copy、move，還是直接在 caller 那邊形成？
```

## Ch1 - How Many Objects Are In This Code?

## Cold Open

```cpp
T make() {
    T x;
    return x;
}

T y = make();
```

## Questions To Ask

```text
這段 code 裡有幾個 T object？
`x` 和 `y` 是同一個 object 嗎？
return by value 一定 copy 嗎？
local object 會先在 callee stack 建好，再搬到 caller 嗎？
```

## Naive Model

```text
function 裡有 x。
caller 裡有 y。
所以應該是先 copy，或至少 move 一次。
```

## Correct Direction

先不要急著背 RVO。

這裡真正冒出的問題是：

```text
T 不是單純從 A 的 bytes 複製到 B 的 bytes。
C++ 需要定義「B 如何得到一個 valid T object」。
```

這就是 object delivery problem。

## Main Reading

- [[20-languages/cpp/deep-dives/Deep Dive - RVO Return Slot and Naive Copy Model]]
- [[20-languages/cpp/object-resource-semantics/Concept - RVO and NRVO]]

## Natural Next Question

```text
如果 y 要得到一個 T，
到底有哪些可能的故事？
```

這自然導向 copy / move / in-place。

## Ch2 - Three Stories For Getting A T

## Core Frame

假設 A 這邊要產生一個 `T`，B 那邊要拿到 `T`。

基本有三種故事：

```text
copy:
    A 有一份 T。
    B 也得到一份語意等價的新 T。

move:
    A 已經有一個 T。
    B 接手 A 背後的 resource / ownership。
    A 保持 valid moved-from state。

in-place construction:
    不要先讓 A 有一個獨立 T。
    T 直接在 B 的 final storage 形成。
```

這章只建立 vocabulary，不深入 standard 細節。

## Why This Works As Early Framing

它不是抽象大理論，而是在回答 Ch1 的普通問題：

```text
`y = make()` 到底怎麼拿到 T？
```

這樣 RVO 不會像孤立最佳化，move 也不會像「到處加 `std::move`」。

## Main Reading

- [[20-languages/cpp/object-resource-semantics/Concept - Object Delivery - Copy Move In-place Construction]]
- [[20-languages/cpp/deep-dives/Deep Dive - Object Not Just Data Report Thesis]]

## Natural Next Question

```text
如果我怕 copy 很貴，
是不是可以幫 compiler 寫 `std::move`？
```

這自然導向 `std::move`。

## Ch3 - Should I Help The Compiler Move?

## Cold Open

```cpp
T make() {
    T x;
    return std::move(x);
}
```

## Question

```text
為什麼要 std::move(x)？

直覺上是：
    x 是 local object。
    function 要結束了。
    反正 x 不用了。
    那我是不是應該幫 compiler move？
```

## Naive Model

```text
copy 很貴。
move 比 copy 便宜。
所以 return std::move(x) 比 return x 保險。
```

## Correct Direction

先不要急著回答 NRVO。

這裡先抓住更根本的問題：

```text
`std::move(x)` 到底做了什麼？
它真的 move 了 x 嗎？
```

這章的任務是把讀者帶到下一段，不是把 return path 一次講完。

## Main Reading

- [[20-languages/cpp/deep-dives/Deep Dive - std move xvalue and Return Path Selection]]
- [[20-languages/cpp/object-resource-semantics/Concept - std move vs Move Constructor]]

## Natural Next Question

```text
std::move 到底是不是 move？
```

這自然導向 value categories。

## Part 2 - Moving Without Moving

這一段處理 `std::move` 的錯覺。

教學重點：

```text
move 不是語法動作。
std::move 也不搬 object。
真正的 move 是 type operation。
```

## Ch4 - `std::move` Does Not Move

## Cold Open

```cpp
T x;
std::move(x);
```

## Question

```text
這行執行完，x 被搬走了嗎？
```

## Naive Model

```text
std::move(x)
-> moves x
-> x 變成 moved-from
```

## Correct Direction

```text
std::move(x)
-> static_cast<T&&>(x)
-> produces an xvalue expression
-> later operation may select move constructor / move assignment
```

`std::move` 是 expression cast / permission signal。

真正 transfer resource 的是 type 的 move constructor / move assignment。

## Main Reading

- [[20-languages/cpp/deep-dives/Deep Dive - std move xvalue and Return Path Selection]]
- [[20-languages/cpp/object-resource-semantics/Concept - std move vs Move Constructor]]

## Natural Next Question

```text
如果 std::move 只是改變 expression，
那 expression category 到底是什麼？
```

這自然導向 value category。

## Ch5 - Value Category Is Not Lifetime

## Why This Chapter Exists

前面已經看到：

```text
x
std::move(x)
T{}
```

它們的差異不能只用「有沒有 object」解釋，還需要 expression category。

## Correct Direction

Expression 有兩個基本面向：

```text
type
value category
```

這章只需要讓讀者抓住：

```text
value category describes expressions;
lifetime describes objects.
```

Examples:

- `x` 是 lvalue expression。
- `std::move(x)` 是 xvalue expression。
- `T{}` 是 prvalue expression。

## Main Reading

- [[20-languages/cpp/deep-dives/Deep Dive - Value Categories Beyond std move]]
- [[20-languages/cpp/object-resource-semantics/Concept - Value Categories - lvalue xvalue prvalue]]
- [Arthur O'Dwyer - Value category is not lifetime](https://quuxplusone.github.io/blog/2019/03/11/value-category-is-not-lifetime/)

## Natural Next Question

```text
如果 std::move 只是允許 move operation 被選到，
那 move operation 本身到底在移動什麼？
```

這自然導向 ownership transfer。

## Ch6 - Move Is Ownership Transfer, Not Faster Copy

## Cold Comparison

```cpp
struct Matrix4x4 {
    float m[16];
};

struct Buffer {
    char* ptr;
    size_t size;
};
```

## Naive Model

```text
move = faster copy
```

## Correct Direction

對 `Matrix4x4` 這種 inline value data，move 可能跟 copy 差不多。

對 `Buffer` 這種 resource owner，move 才有明確語意：

```text
destination 接手 resource ownership。
source 留在 valid moved-from state。
```

所以 move 的重點不是「一定比較快」，而是：

```text
copy 不合理或太貴時，
ownership 可以被轉移。
```

## Main Reading

- [[20-languages/cpp/deep-dives/Deep Dive - Move Ownership Matrix4x4 and Resource Transfer]]
- [[20-languages/cpp/object-resource-semantics/Concept - Move Semantics and Ownership]]
- [[20-languages/cpp/conversation-notes/Conversation Note - Move Ownership and Value Categories]]

## Natural Next Question

```text
為什麼 Buffer 的 copy 會不合理？
copy 不是只是複製一份 data 嗎？
```

這自然導向 C Buffer。

## Part 3 - When Copy Loses Meaning

這一段才引入 C `Buffer`。

它不是開場，而是用來回答 Part 2 留下的問題：

```text
為什麼 copy / move 不是單純效能問題？
為什麼 move 要講 ownership？
```

## Ch7 - C Buffer: Representation Copy Is Not Semantic Copy

## Cold Open

```c
typedef struct {
    char* ptr;
    size_t size;
} Buffer;

Buffer a = buffer_create(1024);
Buffer b = a;
```

## Questions To Ask

```text
這行 `Buffer b = a;` copy 了什麼？
Buffer 能不能 copy？
copy 是 shallow copy 還是 deep copy？
copy 之後誰 owns heap buffer？
誰負責 destroy？
```

## Naive Model

```text
struct assignment 就是複製一份 Buffer。
```

## Correct Direction

在 C 這裡，representation 被複製了：

```text
a.ptr == b.ptr
```

但 ownership semantics 沒有被複製：

```text
a.ptr ---+
         +--> same heap buffer
b.ptr ---+
```

如果最後：

```c
buffer_destroy(&a);
buffer_destroy(&b);
```

就可能 double free。

這不是 C 做不到，而是這些規則沒有被 type 本身保存。沒有完整上下文的人，只看 `typedef struct` 和 `Buffer b = a`，無法知道正確語意。

## Main Reading

- [[20-languages/cpp/conversation-notes/Conversation Note - C vs C++ Semantic Lifting and Hidden Semantics]]
- [[20-languages/cpp/deep-dives/Deep Dive - C Convention to Cpp Semantic Lifting]]
- [[20-languages/cpp/object-resource-semantics/Concept - C vs C++ Semantic Lifting]]

## Natural Next Question

```text
如果問題不是「C 能不能做」，
而是語意沒有被 type 保存，
那 C++ 會把這些語意放到哪裡？
```

這自然導向 C++ Buffer。

## Ch8 - C++ Buffer: Copy / Move / Destroy Become Semantics

## Cold Open

```cpp
class Buffer {
public:
    Buffer(size_t n) : ptr(new char[n]), size(n) {}
    ~Buffer() { delete[] ptr; }

private:
    char* ptr;
    size_t size;
};
```

## Question

```text
如果這個 type 有 destructor，
compiler-generated copy constructor 還對嗎？
```

## Naive Model

```text
class = fields + destructor
copy / move 是 compiler 幫我處理的小事
```

## Correct Direction

如果 type owns resource，copy / move / destroy 就不再是小細節，而是它的核心語義。

```text
RAII:
    constructor acquire resource
    destructor release resource
    resource lifetime follows object lifetime

copy / move:
    當 RAII object 被交付給另一個 object 時，
    ownership invariant 要怎麼保持？
```

C++ 的方向是讓 type operation 明確回答：

```text
copy:
    deep copy? shared ownership? deleted?

move:
    transfer ownership?
    moved-from source remains valid?

destructor:
    releases owned resource?
```

## Main Reading

- [[20-languages/cpp/deep-dives/Deep Dive - Buffer Bug RAII Rules and noexcept Move]]
- [[20-languages/cpp/deep-dives/Deep Dive - Object Not Just Data Report Thesis]]
- [[20-languages/cpp/object-resource-semantics/Concept - RAII and Rule of 0 3 5]]

## Natural Next Question

```text
如果 destructor 代表 release resource，
那 constructor / copy / move / assignment 是不是也都必須成為同一組語義？
```

這自然導向 RAII / Rule of 0/3/5。

## Ch9 - Rule Of 0/3/5 Is Ownership Pressure

## Naive Model

```text
Rule of 5 是 C++ 背誦規則。
```

## Correct Direction

Rule of 0/3/5 不是 checklist，而是 ownership pressure：

```text
if destructor frees resource,
then copy/move/assignment must define resource semantics too.
```

更精準地說：

```text
RAII 先讓 object 擁有 resource。
Rule of 0/3/5 是在問：
    這個擁有 resource 的 object
    被 copy、move、assign、destroy 時，
    同一個 ownership invariant 是否仍然成立？
```

Modern direction:

```text
put ownership into RAII handle types
let higher-level domain types follow Rule of Zero
```

## Main Reading

- [[20-languages/cpp/deep-dives/Deep Dive - Buffer Bug RAII Rules and noexcept Move]]
- [[20-languages/cpp/object-resource-semantics/Concept - RAII and Rule of 0 3 5]]

## Natural Next Question

```text
現在我們知道 copy / move 背後有 ownership semantics。
那回到一開始的 return by value，
哪些情況是 copy、move，哪些情況是連 move 都不需要？
```

這自然導向 RVO revisited。

## Part 4 - Return By Value Revisited

這一段回到開頭，但現在讀者已經有足夠 vocabulary。

現在可以精準區分：

```text
copy:
    duplicate a semantic T

move:
    transfer ownership from an existing T

RVO / copy elision:
    initialize result object directly
```

## Ch10 - `return T{}`: Sometimes There Is Nothing To Move

## Cold Open

```cpp
T make() {
    return T{};
}
```

## Question

```text
這裡一定要先建立 temporary，再 move 到 caller 嗎？
```

## Naive Model

```text
callee:
    建立 temporary T

return:
    move temporary 到 caller
```

## Correct Direction

`return T{}` 這類 same-type prvalue return 可以直接初始化 result object。

重點不是「compiler 很聰明幫你省掉 move」，而是：

```text
有些 path 根本不是先有 source object 再 transfer；
語言規則允許 result object 直接被初始化。
```

## Main Reading

- [[20-languages/cpp/deep-dives/Deep Dive - RVO Return Slot and Naive Copy Model]]
- [[20-languages/cpp/object-resource-semantics/Concept - Copy Elision and C++17 prvalue]]

## Natural Next Question

```text
那 named local return 呢？
`return x` 和 `return std::move(x)` 哪個比較好？
```

這自然導向 NRVO / return path selection。

## Ch11 - `return x` vs `return std::move(x)`

## Cold Open

```cpp
T make() {
    T x;
    return x;
}

T make_move() {
    T x;
    return std::move(x);
}
```

## Naive Model

```text
copy 很貴。
move 通常比 copy 便宜。
所以 `return std::move(x)` 應該比較保險。
```

## Correct Direction

```text
return x:
    keeps NRVO candidate shape
    fallback may move/copy

return std::move(x):
    changes the return expression to xvalue
    usually loses NRVO candidate shape
```

手動 `std::move` 不是 return local 的保險。

RVO / NRVO 在這條主線中的定位：

```text
move:
    source object 已經存在時，如何低成本轉移 ownership。

RVO / copy elision:
    如果 source object 不必獨立存在，
    就直接在 destination 建構。
```

## Main Reading

- [[20-languages/cpp/deep-dives/Deep Dive - std move xvalue and Return Path Selection]]
- [[20-languages/cpp/object-resource-semantics/Concept - RVO and NRVO]]

## Natural Next Question

```text
我們一路看了 return by value、RVO、std::move、move、Buffer、RAII。
這些真的只是 C++ 零散規則嗎？
還是它們在解同一個更大的問題？
```

這自然導向 Big Reveal。

## Part 5 - The Big Reveal: From C Convention To C++ Type Semantics

這段是昇華主題，不放在開頭。

因為如果一開始就說：

```text
C++ solves C's semantic gap.
```

讀者可能只會覺得抽象。

但現在讀者已經看過：

- return by value 不等於 copy；
- `std::move` 不 move；
- value category 描述 expression，不是 object lifetime；
- move 是 ownership transfer；
- C `Buffer` 展示 representation copy 可以失去 ownership semantics；
- C++ `Buffer` 展示 copy / move / destroy 必須成為 type operations；
- RAII / Rule of 0/3/5 來自 ownership pressure；
- RVO / copy elision 是有些情況下連 move 都不用。

這時才可以回頭說：

```text
原來這些不是零散規則。
它們都在把 resource semantics 從 convention 提升到 type / object model。
```

## Ch12 - From C Convention To C++ Semantic Lifting

## C-Side Questions

```text
這個 pointer 是 owner 還是 view？
誰 free？
copy 是 shallow 還是 deep？
這個 struct 能不能 bitwise copy？
哪個 function 會 invalidate 哪個 resource？
```

## C++-Side Mechanisms

```text
constructor
destructor
copy constructor
move constructor
deleted operations
RAII
type invariant
library vocabulary types
```

## Main Reading

- [[20-languages/cpp/deep-dives/Deep Dive - C Convention to Cpp Semantic Lifting]]
- [[20-languages/cpp/object-resource-semantics/Concept - C vs C++ Semantic Lifting]]
- [[20-languages/cpp/conversation-notes/Conversation Note - C vs C++ Semantic Lifting and Hidden Semantics]]

## Natural Next Question

```text
如果 C++ type 承載了這麼多語義，
那 type 到底是什麼？
```

這自然導向 type invariants。

## Ch13 - A Type Is Not Just A Layout

## Naive Model

```text
type = memory layout + member functions
```

## Correct Direction

```text
Type =
    valid states
  + operations
  + invariants / laws
  + lifetime rules
  + exception safety
  + cost model
```

這裡可以把前面的所有議題收束：

- constructor 建立 valid state；
- destructor 結束 lifetime；
- copy 定義 value/resource duplication；
- move 定義 ownership transfer；
- deleted operation 表示這個 type 不支援某種語義；
- value-like type 和 resource owner 需要不同 operation set。

## Main Reading

- [[20-languages/cpp/deep-dives/Deep Dive - Type Operations Laws Invariants and Stepanov]]
- [[20-languages/cpp/object-resource-semantics/Concept - C++ Type as Operations and Invariants]]
- [[20-languages/cpp/conversation-notes/Conversation Note - Type Operations Mathematical Analogy and Stepanov]]

## End Point

走到這裡，讀者應該能把整條路線收束成：

```text
RVO 告訴我們 object 不一定要被搬。
move 告訴我們 object 背後的 resource 不能只是被 copy。
RAII 告訴我們 resource lifetime 可以跟 object lifetime 綁定。
Rule of 0/3/5 告訴我們 ownership invariant 會壓迫整組 type operations。

真正的主題是：
    C++ 如何用 object lifetime、type operations、library vocabulary
    保留原本容易藏在 C convention 裡的 semantics。
```

## Extensions

這些主題都重要，但不放進主線。它們會把焦點從：

```text
object 如何被建立 / 交付 / 轉移 / 銷毀，
以及這些操作如何保留語意
```

拉到更專門的 library behavior、generic programming、low-level storage、ABI 或 cross-language design。

## E1 - `noexcept` Move And Generic Containers

Status:

```text
extension / library contract case study
```

Why it is not main path:

```text
它不是 object delivery 的核心問題，
而是 generic library 在 exception guarantee 下如何選 copy 或 move。
```

Core question:

```cpp
std::vector<Buffer> buffers;
buffers.push_back(Buffer(1024));
```

```text
vector reallocation 時一定會 move elements 嗎？
```

Correct direction:

```text
move operation 不只要存在，
generic container 還會在意它是否 noexcept。

noexcept move 是 type 對 generic code 的承諾。
```

Reading:

- [[20-languages/cpp/deep-dives/Deep Dive - Buffer Bug RAII Rules and noexcept Move]]
- [[20-languages/cpp/object-resource-semantics/Concept - noexcept Move and Container Reallocation]]

## E2 - Regularity, Concepts, And Generic Programming

Status:

```text
extension / generic programming endpoint
```

Why it is not main path:

```text
主線只需要讓讀者知道 type 不只是 layout。
regularity / concepts 是下一層：
generic algorithm 如何描述它需要哪些 operations。
```

Core question:

```text
algorithm 需要的不是某個 class name，
而是一組 operation requirements。
那 C++ 如何描述這件事？
```

Correct direction:

```text
algorithm asks for operation requirements
concepts name those requirements
regular / semiregular describe value-like behavior
```

Reading:

- [[20-languages/cpp/deep-dives/Deep Dive - Type Operations Laws Invariants and Stepanov]]
- [[20-languages/cpp/object-resource-semantics/Concept - C++ Type as Operations and Invariants]]

## E3 - Final Storage Beyond Return

Status:

```text
supporting / optional
```

Why it is not main path:

```text
Part 4 已經透過 return by value / RVO 建立 no-transfer case。
如果主線再開一整個 Part 講 final storage，會像重複 RVO。

這段只作為補充：
同一個 final-storage 思想在 container / wrapper / low-level storage 裡如何出現。
```

Topics:

- raw storage is not an object;
- storage vs object lifetime;
- `emplace` is not magic;
- `emplace_back(args...)` vs `emplace_back(T(args...))`;
- `vector` reallocation may still move/copy existing elements;
- placement new / `construct_at` as low-level lifetime control.

Reading:

- [[20-languages/cpp/deep-dives/Deep Dive - Destination First Construction Across RVO Emplace and Placement New]]
- [[20-languages/cpp/deep-dives/Deep Dive - Half RVO Misconception Storage and Lifetime]]
- [[20-languages/cpp/conversation-notes/Conversation Note - RVO Verification Compile Flags and Placement New]]
- [[20-languages/cpp/object-resource-semantics/Concept - Storage vs Object Lifetime]]
- [[20-languages/cpp/object-resource-semantics/Concept - In-place Construction and emplace]]
- [[20-languages/cpp/object-resource-semantics/Concept - Placement New and construct_at]]

## E4 - Factory Lambda And Delayed Construction

Status:

```text
advanced / revisit later
```

Why it is not main path yet:

- It depends on destination-first construction.
- It depends on materialization timing.
- It depends on same-type prvalue direct initialization.
- It often belongs to API boundary design.
- It can blur RVO / emplace / placement new / factory API if introduced too early.

Reading:

- [[20-languages/cpp/deep-dives/Deep Dive - Factory Lambda Is Not Syntax Sugar]]
- [[20-languages/cpp/object-resource-semantics/Concept - Factory Lambda and Delayed Construction]]
- [Arthur O'Dwyer - The Superconstructing Super Elider](https://quuxplusone.github.io/blog/2018/03/29/the-superconstructing-super-elider/)

## E5 - ABI Return Slot

Status:

```text
systems appendix
```

Boundary:

```text
useful implementation model,
not standard source transformation.
```

Reading:

- [[30-systems/abi-function-call/Concept - Function Call Stack and Return Object Storage]]

## E6 - Rust Ownership As Comparison

Status:

```text
oral aside / appendix
```

Role:

```text
C++:
    many ownership/lifetime rules live in RAII, type operations, convention, static analysis

Rust:
    more ownership/borrowing discipline is compiler-enforced
```

Rust is useful as contrast, but should not pull the main C++ teaching path sideways.

Reading:

- [[20-languages/rust/Concept - Rust Ownership Compared With C++ Move]]

## E7 - Other C Semantic Lifting Entry Points

Status:

```text
future expansion
```

Possible openings:

- raw pointer ownership unclear -> `unique_ptr` / `shared_ptr` / `span` / `string_view`;
- out parameter -> return-by-value / `optional` / `expected`;
- lock/unlock -> `lock_guard`;
- enum + union active member -> `variant`;
- `qsort` / `void*` -> templates / iterators / concepts;
- callback + `void* context` -> lambda / callable object / type erasure;
- data race -> mutex / atomic / memory model.

These are useful, but the current teaching route stays centered on return-by-value -> move -> Buffer -> type semantics.

## E8 - CppCon Watchlist

Status:

```text
further reading
```

Reading:

- [[90-reading-notes/cpp/Reading - CppCon Watchlist for C++ Object Semantics]]
- [CppCon 2019 Back to Basics Track](https://cppcon.org/b2b2019/)

## Chapter Writing Rules

正式寫章節時，每章都應該遵守：

1. Start with code or a concrete failure.
2. Ask the reader what they think happens.
3. State the naive model explicitly.
4. Break the naive model with one sharp example.
5. Only then introduce formal vocabulary.
6. Mark whether a claim is standard semantics, library behavior, guideline, or implementation model.
7. End with exactly three things the reader should now be able to answer.
8. Link back to the relevant Conversation Note.

## Slogan Bank

- `Returning by value does not mean copying.`
- `The first question is not RVO; it is how the caller gets a valid T.`
- `C++ object delivery has three stories: copy, move, or construct in place.`
- `std::move does not move.`
- `Value category is not lifetime.`
- `Move is not faster copy.`
- `Representation copy is not semantic copy.`
- `Object is not just data.`
- `Rule of 0/3/5 is ownership pressure, not a checklist.`
- `RVO means sometimes there is nothing to move.`
- `A type is not just a layout.`
- `C can do it; the question is where the meaning is stored.`
- `C++ lifts resource conventions into type operations.`

## Next Step

Do not add more topics yet.

Next useful action:

```text
Write Chapter 1 - How Many Objects Are In This Code?
```

Expected chapter goal:

```text
Use `T make() { T x; return x; }` to make the reader feel:
return by value is not automatically "copy local object into caller".

Do not fully teach prvalue / xvalue yet.
Do not start with `return T{}`.
Do not start with `return std::move(x)`.
Do not mention C Buffer yet.
```
