# Concept - Function Call Stack and Return Object Storage

## Derived From

- Conversation Note: [[20-languages/cpp/conversation-notes/Conversation Note - RVO Basics Return Slot and In-place Analogies]]
- Conversation Note: [[20-languages/cpp/conversation-notes/Conversation Note - Type Operations Mathematical Analogy and Stepanov]]
- Deep Dive: [[20-languages/cpp/deep-dives/Deep Dive - Half RVO Misconception Storage and Lifetime]]
- Deep Dive: [[20-languages/cpp/deep-dives/Deep Dive - RVO Return Slot and Naive Copy Model]]

## Problem

`return by value` 表面上像是 callee 先做出一個 object，再把它交給 caller。但在 function call / ABI 層，尤其是大型 non-trivial object，常見模型是 caller 先準備 result storage，callee 直接把結果建構到那裡。

## Original Context To Keep

原始對話裡，你一直想把 RVO 放回 stack frame / memory diagram 裡理解。這篇是 systems-side lookup，不是 C++ semantic 的主文章。它服務的問題是：source code 的 `return obj;` 可能如何被 calling convention lowering？

## Why Naive Fails

naive stack model：

```text
main stack frame:
    x

make stack frame:
    local y
    return temporary

return:
    y -> return temporary -> x
```

這會把 RVO 想成「建好後再省掉搬運」。

## Mental Model

return slot model：

```text
main stack frame:
    x storage
        ^
        |
hidden return pointer / return slot
        |
make(...) constructs result here
```

source-level：

```cpp
T make() {
    T obj;
    return obj;
}

int main() {
    T x = make();
}
```

lowering mental model：

```cpp
void make(T* return_slot) {
    new (return_slot) T();
    // initialize result
}

int main() {
    alignas(T) unsigned char storage[sizeof(T)];
    make(reinterpret_cast<T*>(storage));
}
```

這不是標準規定的 source transformation，而是理解 ABI / codegen 常見做法的模型。

因此這篇的說法只能當 systems mental model：它解釋 compiler / ABI 可能怎麼做到 return-by-value without extra object。C++ 語言層的主張仍然要回到 return statement、copy/move elision、object lifetime，而不是假設每個平台都有同一個 hidden pointer 形式。

## Details

小型 primitive return 可能走 register；大型或 non-trivial object 常需要 memory return。實際規則依 ABI、calling convention、target architecture 而定。

這就是為什麼 RVO 不能只看 C++ source code 的變數名稱。source 上看見 `obj`，不代表 runtime 一定有一個獨立 callee-local object 再被搬出去。

## Relation To C++ Object Lifetime

RVO / NRVO 成功時，重點不是 bytes 被搬得少，而是 result object 的 lifetime 可以直接在 caller storage 開始。

```text
storage chosen by caller
-> constructor runs there
-> object lifetime begins there
```

## Verification

可以用 constructor / destructor log 和 address print 做觀察：

```cpp
struct T {
    T() { puts("ctor"); }
    T(const T&) { puts("copy"); }
    T(T&&) { puts("move"); }
    ~T() { puts("dtor"); }
};
```

編譯觀察 command 會產生 executable：

```bash
g++ -std=c++17 -O0 rvo.cpp -o rvo
g++ -std=c++17 -O0 -fno-elide-constructors rvo.cpp -o rvo_no_elide
```

`-fno-elide-constructors` 會影響可選 copy elision，但 C++17 prvalue guaranteed copy elision 的語意不是單純被這個 flag 完整取消。

## Links

- [[20-languages/cpp/object-resource-semantics/Concept - RVO and NRVO]]
- [[20-languages/cpp/object-resource-semantics/Concept - Storage vs Object Lifetime]]
- [[20-languages/cpp/object-resource-semantics/Concept - Placement New and construct_at]]

## Source

- [[20-languages/cpp/Source Map - ChatGPT CPP RVO 解釋|Source Map]]: source lines `28115-28582`, `28624-29017`, `35270-35581`, `36172-36342`
