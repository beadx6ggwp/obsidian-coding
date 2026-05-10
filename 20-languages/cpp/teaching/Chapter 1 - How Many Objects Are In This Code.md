# Chapter 1 - How Many Objects Are In This Code?

Related:

- [[20-languages/cpp/Learning Path - C++ Object and Resource Semantics]]
- [[20-languages/cpp/deep-dives/Deep Dive - RVO Return Slot and Naive Copy Model]]
- [[20-languages/cpp/conversation-notes/Conversation Note - RVO Basics Return Slot and In-place Analogies]]

## Goal

這章只處理一個很普通、但很容易想錯的問題：

```text
我在 function 裡建立一個 local object，然後 return。
caller 那邊到底怎麼拿到這個 object？
```

這章不急著完整解釋 RVO、NRVO、`std::move`、`prvalue`、`xvalue`。

它只要讓讀者感受到：

```text
return by value 不應該被簡化成：
    把 local object 的 bytes copy 到 caller。
```

## Cold Open

先看這段 code：

```cpp
T make() {
    T x;
    return x;
}

T y = make();
```

如果你已經寫過一點 C++，你可能會自然產生幾個問題：

```text
這裡到底有幾個 T object？
function 裡有一個 x，caller 裡有一個 y，所以是兩個嗎？
return by value 會不會 copy？
如果 copy 很貴，這樣寫會不會很糟？
```

這些問題都合理。

但第一個要修正的是：我們不應該太快把它想成「先在 function 裡做一個 `x`，再把 `x` 的 bytes 搬到 `y`」。

## The Naive Picture

最直覺的模型通常長這樣：

```text
callee:
    建立 x

return:
    copy x

caller:
    得到 y
```

也就是：

```text
T make() {
    T x;       // object 1
    return x; // copy object 1
}

T y = make(); // object 2
```

這個模型有一個好處：它很容易想。

但它也有一個問題：它太像 C-style memory movement。

它把 `return by value` 想成：

```text
function 裡有一包 bytes。
caller 那邊需要一包 bytes。
所以把 bytes 複製過去。
```

對一些很簡單的 trivial data，這種想法可能暫時不會害你。

但一旦 `T` 是一個真正的 C++ object，這個模型就不夠了。

## A Better First Question

這段 code 真正逼出的問題不是：

```text
compiler 有沒有偷偷幫我省掉 copy？
```

而是：

```text
caller 那邊如何出現一個可以正常使用的 T object？
```

這個問法比「copy 幾次」更基本。

因為 caller 最後需要的不是一包看起來像 `T` 的 bytes，而是一個真的可以被當成 `T` 使用的 object。

也就是：

```text
T y = make();
```

這行的結果應該是：

```text
y 可以正常使用。
y 可以正常銷毀。
y 的內容應該是 make() 想交給 caller 的結果。
```

現在先不要急著問它是 copy、move、還是 RVO。

先問更基本的：

```text
如果 return by value 真的需要 copy，
那 copy 到底要保證什麼？
```

## Why Counting Objects Is Tricky

我們可以用一個會印出 constructor / destructor 的 type 來觀察：

```cpp
#include <iostream>

struct T {
    T() {
        std::cout << "default constructor\n";
    }

    T(const T&) {
        std::cout << "copy constructor\n";
    }

    ~T() {
        std::cout << "destructor\n";
    }
};

T make() {
    T x;
    return x;
}

int main() {
    T y = make();
}
```

你可能期待看到：

```text
default constructor
copy constructor
destructor
destructor
```

但實際上，在現代 compiler 的一般設定下，你很可能只看到：

```text
default constructor
destructor
```

這不代表 constructor / destructor 不重要。

也不代表 C++ object 只是被 compiler 偷偷「最佳化掉」。

比較好的理解是：

```text
C++ 不要求 return by value 一定要先做出一個獨立 local object，
再把它 copy 到 caller。

語言和 compiler 可以讓 caller 需要的 result object
直接在正確的位置形成。
```

這就是為什麼「這段 code 到底有幾個 object？」不是最穩定的第一個問題。

更好的第一個問題是：

```text
caller 需要一個 T。
這個 T 是怎麼被建立出來的？
如果不能直接建立，它需要什麼操作才能被交付？
```

## What This Chapter Does Not Explain Yet

這章故意不展開這些名詞：

```text
RVO
NRVO
copy elision
C++17 prvalue
implicit move
std::move
xvalue
```

這些都會回來。

但如果太早把這些詞全部攤開，讀者會變成在背規則，而不是先抓住問題。

現在只需要記住：

```text
return by value 不是一句「會 copy」就能描述完的事。
```

## The First Pressure: What If It Copies?

假設暫時沒有任何 copy elision。

假設真的需要 copy。

那下一個問題就是：

```text
copy 到底是什麼？
```

最 naive 的答案是：

```text
copy = 把 A 的 bytes 複製到 B
```

但對 C++ object 來說，這通常不夠。

比較接近我們需要的說法是：

```text
copy 之後，
B 應該是一個可以正常使用、正常銷毀的 T。
```

這句話現在看起來好像很平凡。

下一章會看到，它其實會直接導向 C++ object/resource semantics 的核心問題：

```text
如果一個 object 擁有 resource，
copy 它到底代表什麼？
```

## Three Things To Take Away

1. `return by value` 不應該被簡化成「把 local object 的 bytes copy 到 caller」。
2. `T y = make();` 真正要求的是：caller 那邊出現一個可以正常使用、正常銷毀的 `T`。
3. 如果真的需要 copy，下一個問題不是「copy 快不快」，而是「copy 之後的 object 是否仍然能正確使用與銷毀」。

## Next Question

```text
如果 copy 不是單純搬 bytes，
那 copy 到底應該做什麼？
```

Next:

- [[20-languages/cpp/teaching/Chapter 2 - Copy Is Not Just Moving Bytes]]
