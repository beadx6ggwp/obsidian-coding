# Concept - Object Delivery - Copy Move In-place Construction

## Derived From

- Conversation Note: [[20-languages/cpp/conversation-notes/Conversation Note - Report Topic Object Delivery RAII and Object Not Data]]
- Conversation Note: [[20-languages/cpp/conversation-notes/Conversation Note - RVO Basics Return Slot and In-place Analogies]]
- Deep Dive: [[20-languages/cpp/deep-dives/Deep Dive - Object Not Just Data Report Thesis]]
- Deep Dive: [[20-languages/cpp/deep-dives/Deep Dive - Destination First Construction Across RVO Emplace and Placement New]]

## Problem

RVO、move、copy 看起來是不同技巧，但原始對話後半段真正收斂到的上位問題是：

```text
How does C++ deliver an object?
```

當一個 function、container、factory 或 system component 要把一個 `T` 交給另一個地方，C++ 有多種 delivery strategy。

更具體地說：

```text
A 要生成一個 T。
B 要怎麼拿到這個 T？
```

## Original Context To Keep

你在對話中自己整理出三種直覺：

- A 有一份，B 也要一份：copy。
- A 不要了，B 接手：move / ownership transfer。
- 不要先在 A 做，直接在 B 製作：in-place construction / RVO-like direction。

後來教學主線應該把這句話講得更精準：

```text
RVO 是「避免搬移」。
move 是「搬移不可避免時，便宜地轉移 ownership」。

現代 C++ 的重點不是到處 std::move，
而是讓物件盡量直接出生在它最後要待的位置。
```

這個 Concept 應該保存這個從 RVO 報告轉向 object delivery 報告的過程。

## Why Naive Fails

只用「RVO 是最佳化」當報告主軸會太窄。它無法解釋：

- 為什麼 move 存在。
- 為什麼 `std::move` 可能讓 return 變差。
- 為什麼 RAII / Rule of 5 會進來。
- 為什麼 `emplace`、factory lambda、placement new 看起來跟 RVO 有共同方向。

## Mental Model

```text
Copy:
    source keeps value/resource meaning
    destination gets independent value/resource meaning

Move:
    source gives up resource ownership
    destination takes ownership

In-place:
    source object may never exist independently
    destination storage gets object lifetime directly
```

## Details

三種策略不是單純效能階層，而是語意不同：

- copy 需要 type 定義「複製後兩邊都 valid」。
- move 需要 type 定義「轉移後 source 仍 valid but moved-from」。
- in-place 需要 destination storage / lifetime control。

RVO 是 function return 上的 in-place / destination-first case。`emplace` 是 container 上的 destination-first case。factory lambda 是 API boundary 上延後 materialization 的 case。

因此 RVO 不應該被理解成孤立的 compiler trick。它是「直接在 destination storage 開始 object lifetime」這條設計方向在 function return 上的代表案例。

## Implementation

設計 API 時先問：

- caller 還需不需要 source object？
- `T` 是否擁有不可共享 resource？
- final storage 是否已知？
- `T` 是否 copyable / movable / non-movable？
- 是否需要 exception safety guarantee？

這些問題決定要用 `T`, `T&`, `T&&`, `unique_ptr<T>`, factory callable, or in-place API。

## Verification

對一個 type 寫出 copy / move / destructor log。測 function return、container insertion、factory construction 三條路，觀察到底是 copy、move，還是直接在 destination 建構。

## Links

- [[20-languages/cpp/object-resource-semantics/Concept - RVO and NRVO]]
- [[20-languages/cpp/object-resource-semantics/Concept - Move Semantics and Ownership]]
- [[20-languages/cpp/object-resource-semantics/Concept - In-place Construction and emplace]]
- [[20-languages/cpp/object-resource-semantics/Concept - Factory Lambda and Delayed Construction]]

## Source

- [[20-languages/cpp/Source Map - ChatGPT CPP RVO 解釋|Source Map]]: source lines `12974-14258`, `14068-14275`, `16409-21742`
