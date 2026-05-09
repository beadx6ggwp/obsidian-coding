# Conversation Note - Factory Lambda and Delayed Construction

## Source Range

- [[20-languages/cpp/Source Map - ChatGPT CPP RVO 解釋|Source Map]]: source lines `3609-5384`
- [[00-inbox/ChatGPT-CPP RVO 解釋|Original conversation]]

這篇保留原本關於 `factory lambda / delayed construction` 的追問脈絡。前面的 concept note 把它壓成「傳 recipe，不要太早 materialize」，但原本對話其實問了很多：是不是語法糖、實際場景、Unreal/Unity/Chrome/browser、manual lifetime、不可 move object、address stability。

## 1. 起點：看不懂 factory lambda / delayed construction

Original question:

- source line `3609`: `進階：factory lambda / delayed construction 看不懂 有示意圖嗎`

這代表你不是只要定義，而是要知道：

```text
這跟 RVO / in-place construction 到底有什麼同構？
它是語法技巧，還是真的改變 object 的出生時間？
```

> [!note] 圖解定位
> 這張圖對應這段對話的總問題：想把 `T` 放進指定 storage 時，差異不是「寫法比較漂亮」，而是 `T` 到底先在外面 materialize，還是等 final storage 已知後才出生。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_22_32.png]]

## 2. 核心追問：這只是語法糖嗎？

Original question:

- source line `3621`: `這一段是語法糖嗎?`

原本討論的對比 code：

```cpp
T& construct_from(T value) {
    return *new (&storage) T(std::move(value));
}
```

vs

```cpp
template<class Factory>
T& construct_from(Factory factory) {
    return *new (&storage) T(factory());
}
```

當時卡住的點是：

```text
兩者看起來都最後 construct 一個 T，
那 factory 版本是不是只是多包一層 lambda？
```

> [!note] 圖解定位
> 這張圖正好回答「是不是語法糖」：direct `T value` 版本已經有一個 `T`，factory 版本傳的是產生 `T` 的 recipe，真正改變的是結果物件的 materialization 時間與位置。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_22_16 (4).png]]

## 3. 傳 `T value` 版本實際發生什麼？

對話展開：

```cpp
T value
```

代表呼叫 `construct_from` 前，或進入 function parameter 時，已經有一個 `T` object materialized。

流程：

```text
caller side / parameter passing:
    construct T value

inside construct_from:
    final storage is known
    move value into storage
```

所以它還是有一個中間 `T value` object。

這對 movable object 可能可以接受，但對不可 move、address-sensitive、或 construction 很重的 object，就不理想。

> [!note] 圖解定位
> 這張圖保留原始對話裡對 `T value` 版本的拆解：呼叫端或參數傳遞階段已經先產生一個 `T`，進入函式後再 move 到 `storage_`，所以不可 move 的 `T` 會直接卡住。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_22_16 (2).png]]

## 4. Factory 版本真正改變什麼？

```cpp
template<class Factory>
T& construct_from(Factory factory) {
    return *new (&storage) T(factory());
}
```

這裡傳進來的不是 `T`，而是產生 `T` 的 recipe。

流程：

```text
caller:
    pass factory / lambda

inside construct_from:
    final storage is known
    call factory()
    use result to initialize T in storage
```

對話中的核心修正：

```text
lambda factory 本身只是 callable object；
真正重要的是 API 不太早要求一個 materialized T。
```

> [!note] 圖解定位
> 這張圖對應你後面修正出的核心：factory 先被傳入，但 `T` 沒有先出現；只有當 `construct_from` 已經拿到 `storage_`，才呼叫 factory 讓 prvalue 直接在目標位置初始化。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_22_16 (3).png]]

## 5. 它和 RVO 如何對齊？

RVO:

```text
caller provides return slot
callee directly constructs result there
```

Factory delayed construction:

```text
storage owner provides final storage
factory is invoked only after final storage is known
T is constructed there
```

同構點：

```text
不要先產生 T，再搬到目的地。
等目的地已知，再讓 T 出生。
```

差異：

```text
RVO 是 return-by-value 語意 + compiler/ABI 機制。
Factory 是 API design pattern，需要自己設計 boundary。
```

> [!note] 圖解定位
> 這張圖把 factory delayed construction 放回 RVO 的大模型：兩者共通點都是 destination-first；差別在 RVO 的 destination 來自 caller return slot，factory pattern 的 destination 來自 wrapper / pool / arena / manual storage。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_22_16 (5).png]]

## 6. 但 lambda 不是魔法

原對話中特別強調了三個前提：

1. `factory()` 要回傳可用來初始化 `T` 的 prvalue 或 compatible result。
2. `T(factory())` 必須能直接初始化 final object。
3. `storage` 必須是未初始化且 alignment 正確的 storage。

這點很重要，因為不是所有 factory 寫法都保證零 temporary。

如果 API 裡面又寫：

```cpp
T temp = factory();
new (&storage) T(std::move(temp));
```

那 delayed construction 的核心就被破壞了。

> [!note] 圖解定位
> 這張圖總結這一節的限制：factory 不是跳過語言規則的魔法，而是把「結果物件」改成「產生結果的方法」。最後仍然要看 initialization form、storage alignment、lifetime start 是否正確。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_22_16 (1).png]]

## 7. 你追問實際用法：Unreal / Unity / 渲染引擎

Original question:

- source line `4000`: `像是unreal、unity 、渲染引擎之類的有真的應用場景嗎`

對話展開不是說「Unreal 就在用這個 exact function」，而是分層類比：

### Unreal `TArray::Emplace`

`emplace` 是最直接的同構：

```text
container element slot known
-> construct element directly there
```

### Unreal Renderer / RHI / RDG

更大的類比不是 C++ object RVO，而是：

```text
render graph 知道 resource lifetime / usage
-> compile graph
-> plan transient resource allocation
```

精神相似：

```text
先知道真正目的地與 lifetime，
再安排 allocation / construction。
```

但這不是 C++ RVO 本身。

### Unity NativeArray / NativeContainer

Unity 是 C#，沒有 C++ RVO，但有相近的 data-oriented memory pattern：

```text
managed wrapper
-> unmanaged memory block
-> job / burst system 直接操作 data
```

類比點是避免不必要 copy，讓資料放在明確管理的 memory 中。

## 8. 渲染引擎實際 case

對話中列了幾個場景：

### Mesh / vertex buffer build

```text
先組出 temporary mesh data
-> move/copy 到 final GPU upload buffer
```

可改成：

```text
final buffer / arena known
-> builder 直接填入目標 storage
```

### Render pass parameter / descriptor

pass descriptor 若只是產生新 value，return-by-value / prvalue 很自然。

### Command buffer / render command queue

如果 command object 不適合 copy，queue 可以用 emplace 或 factory 建構 command。

### Frame graph / render graph transient resources

graph compile 後知道 resource lifetime，才配置 transient resource。這是更大尺度的 delayed allocation / lifetime planning。

## 9. 你追問 Chrome / Browser

Original question:

- source line `4549`: `那瀏覽器呢(chrome) factory lambda / delayed construction的場景嗎`

對話展開：

### Lazy global / `NoDestructor`

接近 delayed construction：

```text
不要在 static initialization 階段就建 object
等第一次需要時再建
避免 destruction order 問題
```

> [!note] 圖解定位
> 這張圖放在 `NoDestructor` 這裡，是因為原始對話的重點不是「Chrome 一定用了同一段 factory lambda」，而是 browser 會把 object 的建構延後到第一次需要、且 lifetime context 比較明確時。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_22_37 (1).png]]

### Task / callback delayed execution

Browser 裡 task/callback 很常見：

```text
先保存 operation recipe
晚點在正確 thread / sequence / lifetime context 執行
```

這不是直接 factory lambda for placement new，但思路相近：延後 materialization / execution 到 context 已知。

> [!note] 圖解定位
> 這張圖對應 browser task / callback 的類比邊界：callback 延後的是 action / recipe 的執行，不是單純延後一個 `T` object 的 placement construction；但它們都避免太早在錯誤 context 產生結果。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_22_37 (2).png]]

### Blink / DOM / GC allocation

`MakeGarbageCollected<T>` 類型 API 是另一種 controlled construction：

```text
object 必須在 GC-managed heap 中建立
不能普通 new/delete
```

這和 RVO 不同，但同樣是「object 不只是 data；它的出生地和 lifetime 管理很重要」。

> [!note] 圖解定位
> 這張圖對應 Blink / Oilpan 的 controlled construction：物件不是想在哪裡 `new` 都可以，而是要出生在 GC-managed heap，之後由 tracing / GC 管理生命週期。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_22_37 (3).png]]

### Address-stable / immovable operation object

最像 factory lambda 的 browser / async case 是：

```text
operation state 必須 stable address
不能先建 temporary 再 move
因為 callback / async machinery 可能保存 address 或 lifetime relation
```

> [!note] 圖解定位
> 這張圖是 browser / async 類比中最接近 factory delayed construction 的 case：operation state 一旦被 request、callback、scheduler 觀察，就不能靠「先 temporary 再 move」來處理。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_22_37 (4).png]]

> [!note] 圖解定位
> 這張圖收束 browser 這組場景：`NoDestructor`、callback、GC heap、operation state 的名字不同，但共同抽象是先決定 destination / lifetime context，再讓 construction 或 execution 發生。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_22_25 (5).png]]

### Browser 小版總覽圖（備查）

下面三張是 browser 場景的小版總覽。正文上面已經放了較大的逐場景圖，這裡保留它們是為了讓 `_assets/0509` 這批原始圖片都有對應語境，不讓之後回查時看不出它們為什麼存在。

> [!note] 圖解定位
> lazy initialization / `base::NoDestructor` 的小版總覽。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_22_24 (1).png]]

> [!note] 圖解定位
> task / callback delayed execution 的小版總覽。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_22_25 (2).png]]

> [!note] 圖解定位
> Blink / Oilpan / `MakeGarbageCollected` 的小版總覽。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_22_25 (3).png]]

## 10. 再次追問 exact 使用場景

Original question:

- source line `4948`: `template<class Factory> T& construct_from(Factory factory) {...}我比較好奇這個的使用場景`

這段最接近這個 pattern 的真實動機。

對話中的 cases：

### 自製 `optional<T>` / `ManualLifetime<T>`，且 `T` 不能 move

如果 `T` 不可 move：

```cpp
struct T {
    T(const T&) = delete;
    T(T&&) = delete;
};
```

那這種 API 不能用：

```cpp
construct_from(T value); // parameter itself needs T materialization / move
```

但可以傳 factory，讓 `T` 在內部 storage 直接被建立。

### Async operation state

operation state 可能需要：

- address stable
- lifetime covers async operation
- start 後不能移動

這時候「先建立 temporary 再 move」可能破壞設計。

> [!note] 圖解定位
> 這張圖對應 exact pattern 的第一個強需求：不可移動或 address-sensitive 的 operation state，必須直接在最終 storage 中開始生命週期。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_22_25 (4).png]]

### Object pool / arena allocator

arena 已經有 final storage。Factory 可以把 construction 延後到 arena slot 確定之後。

> [!note] 圖解定位
> 這張圖把 factory pattern 接到 engine / renderer 常見的 command pool：pool 或 arena 已經決定 slot，factory 的價值是讓 command 直接出生在那個 slot，而不是先做中間 command 再搬進去。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_22_37 (5).png]]

### `optional<T>` 放不可移動物件

`optional` 內部有 raw storage。只有 engaged 時才建 `T`。

```text
optional storage exists
T object not yet exists
emplace / factory
T lifetime begins inside optional storage
```

### 建構過程需要保持地址穩定

如果 constructor 或 registration 過程需要穩定 address，就不能先在 temporary address 建好，再 move 到 final address。

> [!note] 圖解定位
> 這張圖對應遊戲引擎 / 大型資源 / event registration 的場景：當 constructor 期間就會註冊 callback、handle 或外部引用時，final address 不是效能細節，而是語意條件。
>
> ![[_assets/0509/ChatGPT Image 2026年5月9日 上午08_22_37 (6).png]]

## 11. 什麼時候不需要這招？

對話也有講限制，這很重要：

不需要 factory lambda 的情境：

- `T` 很便宜 move。
- `T` 是普通 value type。
- 你本來就已經有一個 `T` object。
- API 語意就是 consume existing `T`。
- bottleneck 不在 object move/copy。

這避免把 delayed construction 變成過度工程化。

## What Should Be Preserved

這段不應該只整理成：

```text
傳 factory 可以延後建構。
```

完整脈絡是：

```text
factory lambda 是語法糖嗎？
-> 不是，重點是避免太早 materialize T
-> 和 RVO 同構：destination known before construction
-> 但不是魔法，要看 factory result、initialization form、storage alignment
-> 真實場景在 manual lifetime、optional、arena、async operation state、address-stable object
-> engine/browser 類比要分清楚 exact C++ mechanism 和更大尺度的 lifetime/resource planning
-> 不是所有 case 都該用，普通 movable value 不需要硬套
```

## Related Notes

- [[20-languages/cpp/object-resource-semantics/Concept - Factory Lambda and Delayed Construction]]
- [[20-languages/cpp/deep-dives/Deep Dive - Factory Lambda Is Not Syntax Sugar]]
- [[20-languages/cpp/object-resource-semantics/Concept - Placement New and construct_at]]
- [[20-languages/cpp/object-resource-semantics/Concept - In-place Construction and emplace]]
- [[20-languages/cpp/object-resource-semantics/Concept - Storage vs Object Lifetime]]
