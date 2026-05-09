# Concept - C vs C++ Semantic Lifting

## Derived From

- Conversation Note: [[20-languages/cpp/conversation-notes/Conversation Note - C vs C++ Semantic Lifting and Hidden Semantics]]
- Deep Dive: [[20-languages/cpp/deep-dives/Deep Dive - C Convention to Cpp Semantic Lifting]]
- Deep Dive: [[20-languages/cpp/deep-dives/Deep Dive - Ownership Design Space Cpp Rust GC]]

## Problem

C 可以用 `struct`、pointer、`malloc/free`、function pointer、out parameter 解決很多問題。那為什麼 C++ 要引入 constructor、destructor、reference、iterator、copy/move、RAII、template 等複雜機制？

核心不是「C++ 把 C 包裝得比較好看」，而是 C++ 試圖把 C 裡靠 programmer convention 維護的語義，提升成 type / object / compiler / library 可以處理的規則。

## Original Context To Keep

原始追問是：

```text
整體上明明 C 能解決，為什麼 CPP 這些要搞得這麼複雜？
CPP 跟 C 的本質差異是什麼？
```

這個問題不能用「C++ 比較安全」帶過。真正要保存的是：C 的很多語義不是不存在，而是藏在人腦、命名、文件、convention 裡。

## Why Naive Fails

naive view：

```text
C has struct + pointer + malloc
therefore C can express the same thing
```

問題是 compiler 不知道這些 pointer 的語義：

- owner 還是 borrower？
- 誰負責 free？
- copy 是 shallow copy 還是 deep copy？
- out parameter 是否 initialized？
- object 是否處於 valid state？

## Mental Model

```text
C:
    memory + pointer + convention

C++:
    object + lifetime + ownership + type operations
```

semantic lifting 就是把原本只有 programmer 知道的規則，提升成 code structure 可以表達、compiler 可以部分檢查、library 可以組合的模型。

## Details

C style：

```c
typedef struct {
    char* ptr;
    size_t size;
} Buffer;
```

C++ style 會嘗試把語義放進 type：

- constructor：如何建立 valid object。
- destructor：如何釋放 resource。
- copy constructor：copy 的真正語意。
- move constructor：ownership 如何轉移。
- `= delete`：禁止不合法 operation。
- `unique_ptr` / `shared_ptr` / `span`：用不同 type 表達不同 ownership / view。
- `optional` / `variant`：把 active state / object lifetime 管起來。

## Tradeoff

C++ 的複雜度來自它同時想要：

- 接近 C 的底層控制能力。
- 高階 abstraction。
- 不隱藏主要成本，也就是 zero-overhead abstraction。

所以語法和規則會複雜，但目標是讓大型系統裡的 lifetime / ownership / operation semantics 更可維護。

## Verification

拿一個 C `Buffer` API，問：

- 哪些 function 建立 valid state？
- 哪些 function 釋放 resource？
- caller 能不能 copy struct？
- double free 如何避免？

如果答案只能靠註解或團隊約定，代表語義還停在 convention 層。

## Links

- [[20-languages/cpp/object-resource-semantics/Concept - RAII and Rule of 0 3 5]]
- [[20-languages/cpp/object-resource-semantics/Concept - C++ Type as Operations and Invariants]]
- [[20-languages/cpp/object-resource-semantics/Concept - Object Delivery - Copy Move In-place Construction]]
- [[20-languages/rust/Concept - Rust Ownership Compared With C++ Move]]

## Source

- [[20-languages/cpp/Source Map - ChatGPT CPP RVO 解釋|Source Map]]: source lines `21778-23107`, `23317-25548`, `27101-28071`
