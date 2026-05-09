# Conversation Note - RVO Verification Compile Flags and Placement New

## Source Range

- [[20-languages/cpp/Source Map - ChatGPT CPP RVO 解釋|Source Map]]: source lines `5385-6278`
- [[00-inbox/ChatGPT-CPP RVO 解釋|Original conversation]]

這段把前面的 RVO mental model 壓回可驗證層：constructor log、compiler flag、C++14 vs C++17、`-fno-elide-constructors`、placement new 是否是真語法。

## 1. 回到 naive 範例：constructor / move / destructor 順序

Original question:

- source line `5385`: 重新貼出 `Image` 範例，追問 naive 想像中的 move / destructor 數量。

原範例：

```cpp
struct Image {
    Image() { puts("ctor"); }
    Image(const Image&) { puts("copy"); }
    Image(Image&&) { puts("move"); }
    ~Image() { puts("dtor"); }
};

Image makeImage() {
    Image img;
    return img;
}

int main() {
    Image a = makeImage();
}
```

對話整理了三個層級：

```text
naive:
    img
    return temporary
    a
    可能看到 ctor / move / move / dtor / dtor / dtor

NRVO success:
    img 就是 a 的 return slot
    可能只看到 ctor / dtor

C++17 with NRVO disabled:
    named local return 仍可能 move
    prvalue return 仍有 guaranteed copy elision
```

> [!note] 圖解定位
> 這張圖對應這節的 naive model：在還沒引入 copy elision / return slot 之前，`return by value` 會被想成 callee local、return object、caller object 三段搬運。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_22_47 (1).png]]

## 2. 怎麼編譯看？

Original question:

- source line `5741`: `如果我想編譯看 是要怎麼編譯`

這段很重要，因為它把抽象規則變成可觀察實驗。

測試目標：

1. 正常現代 compiler 下 NRVO 是否發生。
2. 用 `-fno-elide-constructors` 人為關掉 optional elision。
3. 比較 C++14 和 C++17 差異。
4. 測 `return Image{};` 這種 guaranteed copy elision case。

提到的 commands 會真的編譯並產生 executable：

```bash
g++ -std=c++17 -O0 rvo.cpp -o rvo
clang++ -std=c++17 -O0 rvo.cpp -o rvo
g++ -std=c++14 -O0 -fno-elide-constructors rvo.cpp -o rvo14_no_elide
g++ -std=c++17 -O0 -fno-elide-constructors rvo.cpp -o rvo17_no_elide
g++ -std=c++17 -O0 -fno-elide-constructors rvo.cpp -o rvo17_prvalue
```

保留的重點：

```text
-fno-elide-constructors 可以幫你觀察 fallback，
但 C++17 prvalue guaranteed copy elision 不是單純 optional optimization。
```

## 3. `new (return_slot) T()` 是真語法嗎？

Original question:

- source line `6042`: `new (return_slot) std::vector<int>(); 這是真實的語法嗎 placement new? 還是舉例`

回答分清楚：

```cpp
new (return_slot) std::vector<int>();
```

這是 placement new 的真實語法，但前面用在 RVO 解釋時是 lowering mental model，不是說 compiler 真的生成這段 source code。

> [!note] 圖解定位
> 這張圖對應「`new (return_slot) T()` 到底是不是真語法」：placement new / `construct_at` 是 programmer 指定 storage 開始 object lifetime 的真實工具；RVO 的 return slot 是相同心智模型，但層級在 compiler / ABI。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_21_33 (2).png]]

## 4. 一般 `new` vs placement new

一般 `new`：

```cpp
T* p = new T(args...);
```

做兩件事：

```text
allocate storage
construct T in that storage
```

placement new：

```cpp
void* mem = ...;
T* p = new (mem) T(args...);
```

只做：

```text
construct T in already-provided storage
```

所以 placement new 和 RVO mental model 的共同點是：

```text
storage 已知
object lifetime 在指定 storage 開始
```

## 5. C++20 `std::construct_at`

這段也提到現代更 library-friendly 的寫法：

```cpp
std::construct_at(ptr, args...);
std::destroy_at(ptr);
```

這讓手動 lifetime management 的意圖更清楚。

## What Should Be Preserved

這段不只是「有個 flag 可以測 RVO」。

完整脈絡是：

```text
RVO mental model
-> 用 constructor log 驗證
-> 比較 C++14 / C++17 / no-elide flags
-> 看出 C++17 prvalue 不是普通 optional optimization
-> placement new 是真語法
-> RVO 的 return slot model 和 placement construction 同構，但層級不同
```

## Related Notes

- [[20-languages/cpp/object-resource-semantics/Concept - RVO and NRVO]]
- [[20-languages/cpp/deep-dives/Deep Dive - RVO Return Slot and Naive Copy Model]]
- [[20-languages/cpp/deep-dives/Deep Dive - Destination First Construction Across RVO Emplace and Placement New]]
- [[20-languages/cpp/object-resource-semantics/Concept - Placement New and construct_at]]
- [[30-systems/abi-function-call/Concept - Function Call Stack and Return Object Storage]]
