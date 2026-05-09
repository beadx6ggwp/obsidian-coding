# Concept - RVO and NRVO

## Derived From

- Conversation Note: [[20-languages/cpp/conversation-notes/Conversation Note - RVO Basics Return Slot and In-place Analogies]]
- Conversation Note: [[20-languages/cpp/conversation-notes/Conversation Note - RVO Verification Compile Flags and Placement New]]
- Deep Dive: [[20-languages/cpp/deep-dives/Deep Dive - RVO Return Slot and Naive Copy Model]]
- Deep Dive: [[20-languages/cpp/deep-dives/Deep Dive - Half RVO Misconception Storage and Lifetime]]

## Problem

直覺的 return-by-value model 會想成：

```text
callee local object
-> return temporary
-> caller variable
```

這讓人以為回傳大型 object 一定要 copy / move 多次。RVO / NRVO 修正的是這個 cost model：如果 result object 的 final storage 已經知道，C++ 可以讓 object 直接在那個 storage 開始 lifetime。

## Original Context To Keep

原始追問不是單純問「RVO 定義是什麼」，而是一路在釐清：

- return value 是不是先在 function 裡做好，再搬回 caller？
- caller 能不能先準備一塊 return slot？
- 同一個 object 的 lifetime 裡，會不會前半段一般 local、後半段才 RVO？
- 報告時應該講 compiler trick，還是講 C++ 如何交付 object？

> [!note] Quick visual
> RVO / NRVO / move fallback 的最小對照。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_22_01 (2).png]]

## Why Naive Fails

naive model 把 source code 裡的 local variable、實體 storage、object lifetime 混成同一件事。

如果 NRVO 成功，named local variable 不是先完整活在 callee stack frame，最後才被搬到 caller。比較好的 mental model 是：compiler 把那個 local object 的 construction 直接安排到 caller 提供的 result storage。

## Mental Model

RVO / NRVO 都是在避免「先有獨立 source object，再把它交給 destination」。

```text
RVO:
    return T{};
    -> directly initialize result object

NRVO:
    T obj;
    return obj;
    -> if chosen, construct obj in return slot from the start

NRVO fail:
    T obj in callee storage
    -> move/copy into return slot
```

## Details

RVO 常指 return expression 是 unnamed temporary / prvalue：

```cpp
Image makeImage() {
    return Image{};
}
```

C++17 之後，很多 prvalue return 不再是「先產生 temporary 再消掉」，而是語言語意上直接初始化 result object。

NRVO 是回傳 named local：

```cpp
Image makeImage() {
    Image img;
    return img;
}
```

NRVO 是允許但不保證的 optimization。它通常要求 return expression 是符合條件的 local object 名稱。

## Common Failure Pattern

```cpp
Image makeImage() {
    Image img;
    return std::move(img);
}
```

`std::move(img)` 把 expression 改成 xvalue。這常會讓 compiler 不能套用 NRVO。通常應該寫：

```cpp
Image makeImage() {
    Image img;
    return img;
}
```

能 NRVO 就直接建構；不能 NRVO 時再走 move fallback。

> [!note] Quick visual
> 多個 local object、手動 `std::move`、mutation API 等都可能讓 RVO / in-place 思路不成立或不適合。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_22_01 (6).png]]

## Verification

可用 constructor / copy / move / destructor print log 觀察。也可以用 `-fno-elide-constructors` 對比 C++14 和 C++17 行為。

注意：這類 command 會真的編譯並產生 executable，不只是 static analysis。

## Links

- [[20-languages/cpp/object-resource-semantics/Concept - Copy Elision and C++17 prvalue]]
- [[20-languages/cpp/object-resource-semantics/Concept - Value Categories - lvalue xvalue prvalue]]
- [[20-languages/cpp/object-resource-semantics/Concept - std move vs Move Constructor]]
- [[20-languages/cpp/object-resource-semantics/Concept - Storage vs Object Lifetime]]

## Source

- [[20-languages/cpp/Source Map - ChatGPT CPP RVO 解釋|Source Map]]: source lines `18-154`, `197-234`, `1071-1185`, `5769-6030`
