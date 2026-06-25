# MOVE 簡報逐頁原文抄寫

狀態：raw transcript

用途：先完整抄寫簡報截圖內容，之後再整理成文章。

規則：

- 一張截圖 = 一個段落。
- 先一字不漏抄寫原版。
- 不整理、不潤稿、不改寫。
- 不主動補充推論。
- 如果截圖中有看不清楚的字，先標記為 `[看不清]`。

---

## Image 1

此頁為內部模板頁，已移除，不納入文章整理。

## Image 2

```text
Copy in C/C++
How many objects are in this code?

• Consider following code
  – What’s the output?

T make() {
    return T{};
}

int main() {
    T y = make();
}

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

default
constructor
copy
constructor
destructor
destructor
```

## Image 3

簡報附註：

```text
但其實compiler 沒有做半個copy，但為啥compiler沒有做copy呢?
```

```text
Copy in C/C++
How many objects are in this code?

• Consider following code
  – What’s the output?

T make() {
    return T{};
}

int main() {
    T y = make();
}

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

default
constructor
destructor
```

## Image 4

簡報附註：

```text
所以我們就要先了解 什麼是copy?
先從一個簡單的struct 開始，這個Point裡面只有兩個int，
所以copy就會很直覺的直接將這兩個值複製到一個新的物件上
但是假如物件變得更複雜呢? 這種複製方式還能保證我們複製出來的b可以被正常使用嗎?
這個b是不是需要對某些資源進行管理?
所以我們就要先了解 什麼是copy?
先從一個簡單的struct 開始，這個Point裡面只有兩個int，
所以copy就會很直覺的直接將這兩個值複製到一個新的物件上
但是假如物件變得更複雜呢? 這種複製方式還能保證我們複製出來的b可以被正常使用嗎?
這個b是不是需要對某些資源進行管理?
```

```text
Copy in C/C++
What is Copy?

• Easy struct: Point
struct Point {
    int x;
    int y;
};

Point a{1, 2};
Point b = a;

b.x = a.x
b.y = a.y

• However, not all objects are that easy. An object may have
  – heap memory
  – file handle
  – mutex lock
  – GPU resource
  – reference-counted control block
  – ...

After a copy, we should consider whether the object b
• Can be used normally?
• Should release some resource?
• Should have a reference of A, or have a new one?
• Should affect A when it dies, and vice versa?
```

## Image 5

簡報附註：

```text
所以現在有一個更複雜的struct Buffer，它裡面有一個member pointer指向heap block
現在我們將a 以剛剛的方式複製到 b，發生了什麼事? 沒事
問題出在buffer_destroy的時候，由於a, b都指向同一塊heap block，因此會有double free的問題發生
```

```text
Copy in C/C++
What makes byte-copy error?

• Easy struct: Buffer

typedef struct {
    char* ptr; // Points to a heap
block
    size_t size;
} Buffer;

// helper function
Buffer buffer_create(size_t size);
void buffer_destroy(Buffer* buffer);

Buffer a = buffer_create(1024);
Buffer b = a;

buffer_destroy(&a);
buffer_destroy(&b);

b.ptr = a.ptr
b.size = a.size

After a copy, we should consider whether the object b
• Can be used normally?
• Should release some resource?
• Should have a reference of A, or have a new one?
• Should affect A when it dies, and vice versa?
```

## Image 6

簡報附註：

```text
我們要怎麼去避免剛剛的問題發生?
在C當中，我們可以去寫一個helper function，假設叫 buffer_clone
裡面就去定義我們理想中的複製行為
```

```text
Copy in C/C++
C can still do it

• The C-style struct Buffer does not tell user how to deal with ownership
  – We can define something like below to avoid shallow copy
    Buffer buffer_clone(const Buffer* src);
    Buffer b = buffer_clone(&a);

  – However, the correctness depends on
    – naming,
    – documentation,
    – team conventions,
    – code reviews,
    – users remembering not to make mistakes
```

## Image 7

簡報附註：

```text
剛剛提到的概念就是 shallow copy與deep copy
一開始，預設的copy會是像shallow copy這樣，他只複製表面上的值
所以她其實就是複製a.ptr的value本身 (而不是a 所指向的地方 (dereference))
導致a.ptr與b.ptr指向同一塊heap block

而剛剛說的解決辦法就是deep copy，他讓b.ptr指向一塊新的heap block，但是這塊空間有著與a的heap block一樣的內容
```

```text
Copy in C/C++
Shallow Copy / Deep Copy

• Shallow copy is cheap

a.ptr
heap block
b.ptr
same

  – It just copy the pointer itself
  – Make object a and b both think they own the heap block

• Deep copy is usually expensive

a.ptr
block A
heap

b.ptr
block B
heap

  – allocate new memory and copy the content in heap block A to heap block B

Buffer a = buffer_create(1024);
Buffer b = buffer_clone(&a);

buffer_destroy(&a);
buffer_destroy(&b);

Buffer buffer_clone(const Buffer* src) {
    Buffer dst = buffer_create(src->size);
    memcpy(dst.ptr, src->ptr, src->size);
    return dst;
}
```

## Image 8

簡報附註：

```text
但這樣的問題是什麼? 現在有一個新進來的菜雞，像我。埋著頭開始寫code，
可能並不知道當我複製一個Buffer的時候，需要去call buffer_clone()這個API
然後code就炸了
為什麼會有這種情況發生，本身就是因為C語言本身對code行為的語意表達得不夠清楚
C雖然能夠做到，但他做的卻不一定是你預期中的事情，這時候語意就產生分歧了
這就會讓使用者在實作上感到困惑: 像是剛剛的Buffer，在缺少外部文件輔助的情況下，我們很難知道它其實要透過buffer_clone來複製才不會有問題

所以C++希望compiler可以協助維護語意的完整性，他把一個物件的產生、複製、移動等等所有有關ownership的東西集成到一個class中
Ownership 可以簡單的想成是誰要負責free掉這塊記憶體
只要在class當中規範得足夠完整，我們處理ownership的轉移問題時，就可以更加直覺
```

```text
Copy in C/C++
What C++ does?

• It does not only just transfer C-style method into constructor and destructor

Buffer buffer_clone(const Buffer* src);
Buffer b = buffer_clone(&a);

Buffer(...)
~Buffer(…)

  – Copying, destruction, and eventual moving all become operations that must be expressed by the type itself
  – Copy / Destroy become type-semantic

Ownership:
  • Following code says that the type Buffer owns the ptr
    ~Buffer() {
        delete[] ptr;
    }

  • If Buffer is copyable, then which object has the ownership of ptr?
    • The copy constructor should define how the type itself deal with the ownership
```

## Image 9

簡報附註：

```text
只是有時候，我們不希望一個物件被複製
在C當中，我們很難去規範使用者不能去對一個物件進行複製，總不能說ㄟ你複製了，電你一下，或是禁假一天
在C++中，我們就可以直接去寫=delete，明確的禁止

但這也導致了一些新的問題，這種”禁止複製的物件”，我要怎麼對她進行一些操作
像是return from function，或是把它放進一個container?
就是透過std::move
Std::move做了什麼呢? 他會明確地告訴compiler說，這個value對於這個物件來說，已經不需要了
因此compiler就可以心安理得地把該物件的ownership給轉交出去
```

```text
Copy in C/C++
Deleted Copy

• Sometime, a type can only have one valid instance at a time
  – In C, it is hard to define something like “This type should not have duplicate”
  – In C++, we can define this in class explicitly

    Buffer(const Buffer&) = delete;
    Buffer& operator=(const Buffer&) = delete;

• It generate new questions
  – When a type cannot have duplicate, how to transfer it into a container?
  – How to return a non-copyable type from a function?
  – How to transfer a non-copyable type from one object to another?

    Buffer make_buffer() {
        Buffer b(1024);
        return b; // cannot copy
    }

    // vector cannot copy type Buffer
    std::vector<Buffer> buffers;

std::move
```

## Image 10

簡報附註：

```text
在C++11以前
聰明的前人想到另批蹊徑，搞了一個auto_ptr出來
這個ptr做了什麼事情呢? 假如不能複製，那我把ownership交出來總行吧?
可以看看這段template，他其實就是一個wrapper，把T給包起來
然後我定義我的copy constructor在轉移ownership的同時，將前一個auto_ptr指向的地方直接弄掉 (前一個auto_ptr變成指向nullptr)
```

```text
Move
Before C++11: auto_ptr

std::auto_ptr<Buffer> a(new Buffer);
std::auto_ptr<Buffer> b = a;    // looks like copy

• auto_ptr performs transfer through the copy channel
  – Looks like copying but actually consumes the source
  – a is still a lvalue, but it does not have the ownership of its heap block

template<class T>
class auto_ptr {
public:
  explicit auto_ptr(T* p = nullptr) : ptr(p) {}
  ~auto_ptr() { delete ptr; }
  auto_ptr(auto_ptr& other) : ptr(other.release()) {}
  T* release() {
    T* p = ptr;
    ptr = nullptr;
    return p;
  }
private:
  T* ptr = nullptr;
};
```

## Image 11

簡報附註：

```text
看起來挺正常的，我確實可以做到ownership的轉移啊，還只用copy constructor，不用什麼花禮胡邵的東西
但當我們想要將auto_ptr傳入一些像是std::sort這樣的algorithm，會發生什麼事情
上次建宏介紹了STL，大家都知道STL的全名是standard template library
哪裡出事了? 就是template出事了
Template 不會知道 auto_ptr的copy constructor是會轉移ownership的，
結果就是copy 之後繼續使用source物件，然後就炸了
```

```text
Move
Before C++11: auto_ptr (Cont.)

• What happens when we put it into STL?

std::vector<std::auto_ptr<T>> v(…);
std::sort(v.begin(), v.end()); // 😨

  – auto_ptr do ownership transfer in copy constructor
  – STL call copy constructor to do copy
    – However, STL does not know that it will make source loss its ownership
    – STL still uses source to do something
    – ERROR
```

## Image 12

簡報附註：

```text
經過剛剛的例子就可以看出來我們不應該在copy 裡面做任何有關ownership transfer的事情
因此，我們需要一個額外的通道來特別處理ownership的轉換
這個通道就是 Move，他也需要藉由一個新的value category 來啟動 (因為我們不能夠用跟copy constructor一樣的，否則一樣會bind 到 copy constructor)
而這個category就是 xvalue
這邊的example我們晚點再看
```

```text
Value Category
xvalue

• A language needs proxy tricks to express a common requirement
  – It indicates that the language itself is missing a semantic channel

Expression | Examples

lvalue
(Locator value)
• x (variable name)
• Functions returning T&
  • ++i
• *ptr (dereference)

xvalue
(eXpiring value)
• Functions returning T&&
  • std::move(x)
• Member variable of a prvalue
  • Point{}.name

prvalue
(Pure rvalue)
• 42, true (literals)
• std::string{}
• Functions returning T
  • i++
```

## Image 13

簡報附註：

```text
所以現在就先來教大家怎麼簡單的判斷 value category (值類別) 
判斷的話可以只看column 1,2 或是 1,3，column 3其實就是column 2的相反，但我比較喜歡col3的理解方式
我們看到一段expression，第一件事情就是去問說”這東西有沒有實體位置?”，如果有的話就會是l或xvalue，沒有就一定是prvalue
接下來就是問這東西允不允許轉移ownership? 不允許的話就是lvalue，允許的話就有可能是xvalue或是prvalue
這時候就會有一個疑問: 為什麼一個沒有實例化的prvalue可以轉移ownership?
這也是為什麼我會覺得col3比較好懂
我們應該要把它看為ownership是否需要被保護? Lvalue很明顯就是需要被保護，因為我們不希望她的ownership隨意變化
除非我們顯式告訴compiler我要轉移這個x，這時候他的ownership就不再被保護，就是xvalue
最後prvalue，他連實體都沒有，自然也沒有ownership，所以也沒東西需要被保護
現在再反過來，為什麼他會被稱作moveable? 因為它能 bind 到 T&& — 包含 move constructor 在內的所有 T&& parameter。

Point{} 沒有名字，因此是一個 prvalue (沒有construct object )
但 Point{}.name 因為寫了 .name，compiler 被迫在 stack 上實例化一個匿名 Point object（Temporary Materialization）。實例化後它有了 memory location（對應判斷 1：Yes），而member variable 會繼承其母體的moveable屬性，因此他仍是moveable (母體是prvalue)（對應判斷 2：Yes）— 所以是 xvalue。
重點：Point{}.name 是 xvalue 的根本原因是 .name 觸發了 materialization。Point{} 後面沒接 .name 時，它永遠停在 prvalue。
```

```text
Value Category
A 2-step Decision

• C++17

1. Has specific memory location? | 2. “Moveable”? | 3. Identity protected? | Examples

lvalue
(Locator value)
Yes
No
Yes
• x (named object)
• Functions returning T&
  • ++i

xvalue
(eXpiring value)
Yes
Yes
No
• Functions returning T&&
  • std::move(x)
• Member variable of a prvalue
  • Point{}.y
• TMC

prvalue
(Pure rvalue)
No
Yes
No
• 42, true (literals)
• std::string{}
• Functions returning T
  • i++
```

## Image 14

簡報附註：

```text
Point{} 沒有名字，因此是一個 prvalue (沒有construct object )
但 Point{}.name 因為寫了 .name，compiler 被迫在 stack 上實例化一個匿名 Point object（Temporary Materialization）。實例化後它有了 memory location（對應判斷 1：Yes），而member variable 會繼承其母體的moveable屬性，因此他仍是moveable (母體是prvalue)（對應判斷 2：Yes）— 所以是 xvalue。
重點：Point{}.name 是 xvalue 的根本原因是 .name 觸發了 materialization。Point{} 後面沒接 .name 時，它永遠停在 prvalue。
```

```text
Value Category
Point{}.y

• Why Point{}.y is xvalue?
  – Point{} is a prvalue
  – When we write .y, it forces compiler to materialize a temporary Point object on the stack
    – It now has memory location (check1: yes)
    – It inherits the moveability of its parent object (check2: yes)
    – Therefore, it’s a xvalue
```

## Image 15

```text
Value Category
Functions

• T f()
  – Returns a pure value -> prvalue

• T& f()
  – Returns an alias of a lvalue expression -> lvalue

• T&& f()
  – Returns an alias of a xvalue expression -> xvalue
```

## Image 16

簡報附註：

```text
前面有提到xvalue還有一種情況會被產生，那就是TMC
通常來說，prvalue在runtime不會實例化，除非不得已
那什麼是不得已的情況? 下面的例子就是一個不得已的情況，
我們在f裡面傳入了一個T{}，這個T很明顯是一個prvalue，
然後這個prvalue就會去binding到這個 void f(T&& ref)，目前還沒有問題。
現在compiler一看這段code，發現不對，怎麼有一個 rvalue reference ref 在reference這個prvalue啊?這樣在runtime的時候不就得去要一塊空間materialize這個prvalue，因此這個T{}在compile time就會被implicit的轉換為一個xvalue，xvalue在runtime會有實體位置，我們就能理所當然的reference在他身上
```

```text
Value Category
Temporary Materialization Conversion

• C++17
  – A prvalue usually don’t create objects -> It’s a recipe
  – The temporary object is only materialized when required (Temporary Materialization Conversion)
    – Upgrade prvalue to xvalue
  – An example of TMC:

struct T {
    T() { std::cout << "default\n"; }
    T(T&&) { std::cout << "move\n"; }
    T(const T&) { std::cout << "copy\n"; }
};

void f(T&& ref) {}

f(T{});
```

## Image 17

簡報附註：

```text
還有一個容易混淆的點就是，我們會覺得一個看起來快死的物件就是rvalue
但什麼是”快死了”? 物件的生命週期，這種概念是在runtime才有的
看到這個例子中，這個f 的 input是一個rvalue  有有有它看起來要死了
還return，沒人接住他就真的死了，有有有有，超級瀕死
所以她是rvalue嗎? 不是，他是lvalue
```

```text
Value Category
Named Object

• Consider following code

T f(T&& rref) {
    // … do something
    return rref;
}

  – What is the value category of rref?
```

## Image 18

簡報附註：

```text
這邊有一個容易混淆的點，就是declaration type 跟 value vategory 之間的關係
我們要將這兩件事區分開來

現在我們先宣告一個type 為 T&& 的rr
這個declaration type告訴我們 
1這個rr在runtime時候的type 
2這個rr 他的binding source只能夠是 xvalue或是prvalue 

至於 Expression就是另外一回事，但Expression的Type會是Declaration去掉reference的結果
而value category就是來補足移除reference所缺失的訊息，怎麼決定value category就是用前面幾頁提到的 2-step decision
```

```text
Value Category
Declaration Type is Not Value Category

• Consider following code

T&& rr = std::move(x);

• rr
  – Declaration
    – Type: T&&
  – Expression
    – Type: T
    – Value category: lvalue

The declaration do 2 things:
1. Define the type of rr
2. Define the binding source of rr

Reference | Binding Source Category
T&        | lvalue
T&&       | xvalue or prvalue
const T&  | All categories are valid

T x; // x: lvalue
T& r1 = x;                         // OK (lvalue)
T& r2 = T{};                       // FAIL (prvalue)
T& r3 = std::move(x);              // FAIL (xvalue)
T&& rr1 = T{};                     // OK (prvalue)
T&& rr2 = std::move(x);            // OK (xvalue)
T&& rr3 = x;                       // FAIL (lvalue)
const T& cr1 = x;                  // OK
const T& cr2 = T{};                // OK
const T& cr3 = std::move(x);       // OK
```

## Image 19

簡報附註：

```text
至於為什麼剛剛說的rref是 lvalue，其實是語言的設計
制定規則的時候，設計師必須考慮兩種可能性
Option A: 假如我們預設將Named objects 視為 non-protected ，也就是 moveable 會發生什麼事?
答案其實蠻類似前面提到的auto_ptr那樣，我們在運行的時候，可能不小心就將我們的variable binding到某個轉移ownership的function，結果執行完那個function，return 回來發現天塌了
所以設計師就選擇option B，他強迫工程師要寫出明確的 “我願意放棄對這個opject 的保護(也可以說成我同意讓這個object進行轉移)”之後，才允許讓任何ownership轉移發生
```

```text
Value Category
Named Object (Cont.)

• Consider following code

T f(T&& rref) {
    // … do something
    return rref;
}

  – What is the value category of rref? lvalue

• Why?
  – This is a design choice, not something logically required
  – Designers had to decide:
    – Option A: Named objects are movable by default
      – The objects may accidentally loss its ownership, just like auto_ptr
    – Option B: Named objects are protected by default (treated as lvalues)
      – Forces developers to explicitly say “I want to move from here.”
```

## Image 20

```text
Value Category
Exercise

• Consider following code

void use(const T&){};
void use(T&&){};

void process_data(T&& t){
    use(t);              // (A)
    use(std::move(t));   // (B)
    use(std::move(T{})); // (C)
}

• What is the value category of t?

• What is the value category of std::move(t)?

• What is the value category of std::move(T{})?
```

## Image 21

簡報附註：

```text
在condition (C)，我們可能會覺得有點奇怪，我把一個prvalue去做std::move之後，他會變什麼? 
我們可以回憶一下std::move，他的樣子是一個接受T&&，然後return T&&，
所以T{}這個prvalue在一開始就會觸發前面提到的TMC，產生了一個temporary object後，變成xvalue
然後就static_cast為T&&後 return
```

```text
Value Category
Exercise (Cont.)

• Consider following code

void use(const T&){};
void use(T&&){};

void process_data(T&& t){
    use(t);              // (A)
    use(std::move(t));   // (B)
    use(std::move(T{})); // (C)
}

• What is the value category of t?
  • Since t has name, so it’s a lvalue
  • Therefore, condition (A) will call use(const T&)

• What is the value category of std::move(t)?
  • We explicitly enable a lvalue t become moveable;
    Thus, std::move(t) is a xvalue
  • Therefore, condition (B) will bind to use(T&&)

• What is the value category of std::move(T{})?
  • Function that returns T&& is xvalue
  • Therefore, condition (C) will bind to use(T&&)
```

## Image 22

簡報附註：

```text
經過了前面的練習，我們都已經學會要怎麼去binding一個接收 T&&的function了
所以現在我們就可以來implement Move Constructor了
```

```text
Move
Move Constructor

Buffer(Buffer&& other)
    : ptr(other.ptr), size(other.size) {
    other.ptr = nullptr;
    other.size = 0;
}

before:
    a.ptr ----> heap block
    b.ptr ----> nothing

after:
    b.ptr ----> heap block
    a.ptr ----> empty / null / safe state

b’s destructor
    delete[]
heap block

a’s destructor
    delete[]
nullptr

• Note that after moving, a is still exists. We call a is in move-from state
  • Valid but unspecified
  • We can define our customed move-from state

    a.ptr ---->
    nullptr
    a.size ---->
    0

  • Or just not doing anything, but we should ensure that destructor will not crash
```

## Image 23

```text
Move
When to use “move”?

typedef struct {
    char* ptr; // Points to a heap block
    size_t size;
} Buffer;

• Sometimes, we don’t care about whether a is live or not
  – We just want b to get a.ptr’s ownership
  – That is, a is a temporary result. Then we can tell compiler explicitly

before:
    a.ptr ----> heap block
    b.ptr ----> nothing

after:
    b.ptr ----> heap block
    a.ptr ----> empty / null / safe state
```

## Image 24

簡報附註：

```text
我們前幾章已經知道
1move 是 transfer ownership 
2也知道 std::move 不真的 move 
3還知道 prvalue 是個 recipe 不是 object 
現在帶著這三個觀念, 回到一開始的問題

哪裡看起來怪怪的?
其實一開始就錯了，T{}真的initialize 了一個temporary object 嗎? 這邊甚至沒有人在 reference 他
```

```text
Move
Sometimes there are nothing to move

• So now, what will compiler do in following code?

T make() {
    return T{};
}

T y = make();

  – T{} initialize a temporary object 😳
  – make() returns the temporary object to caller
  – Since the temporary objects are about to die, it’s a prvalue
  – Caller uses prvalue to trigger Move Constructor to construct y
```

## Image 25

簡報附註：

```text
先看 T make() function 內部，T{} 本身是一個prvalue，沒什麼問題，很直覺
然後 caller 端看到了什麼? 一個會return type T的function，也是一個prvalue
而prvalue在C++17會被當成什麼? 一個initialize 物件的recipe，阿這個recipe就是這個T{}
所以最後 y就使用這個recipe 進行initialize
所以從caller的角度來看 T y = make();
其實跟 T y = T{}; 是一樣的，所以在這種expression type 相同的case中，除了y本身以外，沒有其他的實體存在過
自然也沒有東西需要被move

看到這邊會有一種，function邊界好像是透明的感覺?
這是因為打從一開始，caller 就已經要好一塊空間，等著recipe來對這塊空間進行initialize了
```

```text
Move
Sometimes there are nothing to move

• So now, what will compiler do in following code?

T make() {
    return T{};
}

T y = make();

  – T{} is a prvalue
  – The function T make() itself is a prvalue, with recipe T{}
  – The caller uses prvalue to initialize object y
```
