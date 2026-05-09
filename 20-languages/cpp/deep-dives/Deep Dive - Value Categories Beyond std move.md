# Deep Dive - Value Categories Beyond std move

## Derived From Conversation Notes

- [[20-languages/cpp/conversation-notes/Conversation Note - Move Ownership and Value Categories]]

## Rebuild Source Rule

這篇從 Conversation Note 裡 value category 的後半段生成，不再只是 `std::move` 的附屬說明。原始對話追問了 xvalue 來源、`T&&` return、lvalue / xvalue / prvalue 歷史。

## Original Questions

- source line `8908`: `xvalue的產生方式是不是只有std::move(lvalue) 還是有其他的`
- source line `9347`: `都是lvalue變成xvalue嗎`
- source line `9607`: `能不能畫個ASCII流程圖 告訴我 lvalue xvalue rvalue prvalue的由來`
- source line `10705`: `不知道什麼時候會用到回傳T&&的這種function 為什麼要這樣規定`
- source line `11159`: `lvalue是本來就存在 xvalue是為了move而生? pvalude則是因為move誕生出的copy而生?`

## Conversation Reconstruction

```text
起點：
    std::move 產生 xvalue 之後，你追問 xvalue 是否只有這一種來源。

擴展：
    你要求 value category 的整體圖，而不是單點定義。

修正：
    T&& type 與 expression category 不是同一件事。

收斂：
    value category 是 C++ 控制 object delivery / overload / materialization 的語義層。
```

## The Map

```text
expression
├── glvalue
│   ├── lvalue
│   └── xvalue
└── prvalue

rvalue = xvalue + prvalue
```

來源導向：

```text
named variable
-> lvalue

std::move(x)
-> xvalue

T{}
-> prvalue

function returns T
-> prvalue

function returns T&
-> lvalue

function returns T&&
-> xvalue
```

## `T&&` Type Is Not Automatically xvalue

```cpp
T&& ref = T{};
ref; // lvalue, because it has a name
```

這是對話中的重要修正。

```text
declared type:
    T&&

expression category:
    depends on expression form
```

## Why Return `T&&` Exists

回傳 `T&&` 的 function call 是 xvalue：

```cpp
T&& get();
get(); // xvalue
```

典型用途：

- `std::move`
- `std::forward`
- forwarding wrapper
- proxy / rvalue-qualified accessor

危險用途：

```cpp
T&& makeT(); // if returning local, dangling
```

## Historical Correction

```text
lvalue:
    old concept, identity / assignable location history

xvalue:
    introduced with C++11 move semantics to represent expiring objects

prvalue:
    not born from move
    C++17 changed materialization model significantly
```

prvalue 不是「因為 move 誕生出的 copy」。它是 pure rvalue；C++17 後更像「用來初始化目標 object 的 value computation」。

## External Source Check

- [cppreference - value categories](https://en.cppreference.com/w/cpp/language/value_category): value category taxonomy and C++17 prvalue changes.
- [cppreference - std::forward](https://en.cppreference.com/w/cpp/utility/forward): forwarding and conditional xvalue/lvalue behavior.

## Final Mental Model

```text
Value category is not trivia.
It is the layer that tells C++ whether an expression denotes identity, expiring object, or pure value computation.
```

## Source

- [[20-languages/cpp/conversation-notes/Conversation Note - Move Ownership and Value Categories]]
- [[20-languages/cpp/Source Map - ChatGPT CPP RVO 解釋|Source Map]]: source lines `8908-11514`

