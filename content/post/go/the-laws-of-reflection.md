+++
categories = ["go", "translate"]
date = "2016-06-15T09:04:18+08:00"
tags = ["go", "translate"]
title = "The Laws of Reflection 中譯"
+++

# 權利聲明

### 原文

原作者：Rob Pike

原文位於 [https://blog.golang.org/laws-of-reflection](https://blog.golang.org/laws-of-reflection)，依 CC BY 3.0 及 BSD 授權，相關權利非屬改譯者所有，詳見原文最末之權利宣告部份。

### 改譯

改譯者：任仲平 Ronmi Ren

本文相關權利屬改譯者所有。除中譯外，為求通順易讀，略有改動，與原文不盡相同。本文以 CC BY 3.0 TW 公開授權，惟程式碼部份除中譯外皆由原文抄錄而來，故依原授權條款授權。

# 介紹

「反射」一詞，在程式設計領域中，特指程式藉由型別系統來偵測、驗證自身結構的能力，是 metaprogramming 的一種。它也非常容易讓人搞混。

不同程式語言中，對於「反射」的定義和實行方式不盡相同。本篇文章旨在解釋 Go 中的反射機制，不應與其他語言混為一談。

<!--more-->

# 型別與介面

既然反射機制是基於型別系統而來的，我們就從 Go 的型別系統講起。

Go 是靜態型別的語言：任何變數都有自己的靜態型別。靜態型別是指在編譯時，一個變數必定是某個已知的型別，而且不會臨時改變。比如

```go
type MyInt int

var i int
var j MyInt
```

`i` 是 `int` 型別而 `j` 則是 `MyInt` 型別。雖然 `i` 和 `j` 終究都是整數 (本質型別都是 `int`)，但它們的靜態型別分別是 `int` 和 `MyInt`，對於編譯器而言是完全不同的型別，所以必須強制轉型才能把值指定給另一個 (如 `i = int(j)`)。

介面型別則是特定方法的集合。如果你把變數宣告成某個介面型別，它將可以儲存任意實值 (concrete value，這裡指的是「不是介面型別的值」)，只要那個實值有完整實作介面指定的方法。常見的例子就是 `io.Reader` 和 `io.Writer`

```go
// Reader is the interface that wraps the basic Read method.
type Reader interface {
    Read(p []byte) (n int, err error)
}

// Writer is the interface that wraps the basic Write method.
type Writer interface {
    Write(p []byte) (n int, err error)
}
```

任何型別只要有一樣的 `Read` 方法，就稱之為「實作了 `io.Reader`」型別；`io.Writer` 亦同。所以如果把變數宣告成 `io.Reader` 型別，這個變數就能儲存各種有 `Read` 方法的實值。

```go
var r io.Reader
r = os.Stdin
r = bufio.NewReader(r)
r = new(bytes.Buffer)
// and so on
```

要特別注意一點：雖然上面 `r` 的值一再更動，但它的型別始終是 `io.Reader`。

空介面是更重要的例子

```go
interface{}
```

空介面沒有必要的方法，也就是說任何實值都實作了空介面。

千萬不要誤會介面是動態型別：上面 `r` 裡儲存的實值雖然各種型別都有，但它們都必須實作 `io.Reader` 介面，所以 `r` 的型別始終會是 `io.Reader`。

我們一再強調這些，是因為反射跟介面是息息相關的。

# 深入介面型別

Russ Cos 寫了一篇[詳細的文章](http://research.swtch.com/2009/12/go-data-structures-interfaces.html)介紹 Go 的介面型別。本節只提其精要。

宣告為介面型別的變數其實儲存了兩個值：指定給它的實值，以及該實值真正的型別。

```go
var r io.Reader
tty, err := os.OpenFile("/dev/tty", os.O_RDWR, 0)
if err != nil {
    return nil, err
}
r = tty
```

`r` 可以說是儲存了 `(tty, *os.File)`。由於 `r` 是 `io.Reader` 型別，我們透過 `r` 只能存取 `Read` 方法；但 `r` 也紀錄了實值的真正型別 `*os.File`，所以我們才能這樣

```go
var w io.Writer
w = r.(io.Writer)
```

這就是所謂的型別斷言 (type assertion)；這代表 `r` 裡的實值 (`tty`) 同時也實作了 `io.Writer` 介面，所以才能指定給 `io.Writer` 型別的變數。現在 `w` 儲存了 `(tty, *os.File)` 了，恰好跟 `r` 存的一樣。雖然實值提供了更多方法，但變數的型別 (`io.Reader`, `io.Writer`) 限制了這個變數可以使用的方法。

再來，我們也可以

```go
var empty interface{}
empty = w
```

現在 `empty` 也是 `(tty, *os.File)` 了。你可以把任何實值指定給空介面的變數，而它也會老老實實地把一切儲存起來。

上面我們並沒有使用型別斷言，因為任何值都必然實作空介面。而在指定給 `w` 的時候，有實作 `io.Reader` 的實值不一定會實作 `io.Writer`，所以需要型別斷言。

另外一個重要的點是：介面變數儲存的是 (實值, 實值的型別) 而非 (實值, 介面的型別)。介面變數只能存實值。

現在可以來談反射了。

# 反射第一法則

## 1. 反射可以把介面變數轉成對應的反射物件

基本上，反射就是去驗證介面變數裡的東西 `(實值, 實值的型別)`。讓我們從 `reflect.Type` 和 `reflect.Value` 開始。這兩個型別讓我們可以存取介面變數裡儲存的兩個東西。`reflect.ValueOf` 跟 `reflect.TypeOf` 可以把介面變數裡儲存的實值和型別轉成 `reflect.Value` 和 `reflect.Type` 傳出來。(雖然從 `reflect.Value` 也能取得 `reflect.Type`，不過我們先略過不提)

我們從 `reflect.TypeOf` 開始：

```go
package main

import (
    "fmt"
    "reflect"
)

func main() {
    var x float64 = 3.4
    fmt.Println("type:", reflect.TypeOf(x))
}
```

執行結果是

```
type: float64
```

你也許會懷疑介面在哪，程式裡只看到 `float64` 而已啊？不過來看看 [godoc](http://golang.org/pkg/reflect/#TypeOf)，`reflect.TypeOf` 接受的參數是空介面型別的：

```
// TypeOf returns the reflection Type of the value in the interface{}.
func TypeOf(i interface{}) Type
```

當執行 `reflect.TypeOf(x)` 的時候，`x` 會先存到一個暫時的空介面變數裡，也就是參數；`reflect.TypeOf` 再從這個空介面變數中提取型別資訊。

`reflect.ValueOf` 當然是提取實值出來

```go
var x float64 = 3.4
fmt.Println("value:", reflect.ValueOf(x))
```

結果是

```
value: <float64 Value>
```

`reflect.Type` 和 `reflect.Value` 都有許多方法可以幫助我們驗證和處理。`reflect.Value` 有個 `Type` 方法可以取得 `reflect.Value` 代表的實值的型別。而 `reflect.Type` 和 `reflect.Value` 都有 `Kind` 方法可以偵測實值的種類：`Uint`，`Float64`，`Slice` 等等。`reflect.Value` 也有些方法可以把真正的值取出來：

```go
var x float64 = 3.4
v := reflect.ValueOf(x)
fmt.Println("type:", v.Type())
fmt.Println("kind is float64:", v.Kind() == reflect.Float64)
fmt.Println("value:", v.Float())
```

結果是

```
type: float64
kind is float64: true
value: 3.4
```

此外也有可以賦值的方法，但這要等到介紹第三法則的時候才會提到。

為了簡化反射機制，取值和賦值的方法都只保留最大的型別 (比如取得任何整數實值都是用 `Int` 方法，而它會回傳 `int64`)，所以你得自己轉型：

```go
var x uint8 = 'x'
v := reflect.ValueOf(x)
fmt.Println("type:", v.Type())                            // uint8.
fmt.Println("kind is uint8: ", v.Kind() == reflect.Uint8) // true.
x = uint8(v.Uint())                                       // v.Uint returns a uint64.
```

而 `Kind` 方法偵測的是本質型別，而非靜態型別：

```go
type MyInt int
var x MyInt = 7
v := reflect.ValueOf(x)
```

`v.Kind()` 依然是 `reflect.Int` 而非 `MyInt`。換句話說，`Kind` 無法分辨 `MyInt` 和 `int`，但 `Type` 可以。

# 反射第二法則

## 2. 反射可以把反射物件轉成介面變數

既然是反射，自然也能反過來用。

`reflect.Value` 提供了 `Interface` 方法把代表的實值重新包成介面變數

```go
// Interface returns v's value as an interface{}.
func (v Value) Interface() interface{}
```

接續前一節的程式，我們可以

```go
y := v.Interface().(float64) // y will have type float64.
fmt.Println(y)
```

來印出實值。


更棒的是，因為 `fmt.Println` 和 `fmt.Printf` 等都接受空介面的參數 (`fmt` 套件會從這個介面參數取得正確的實值和型別來處理)，所以要印出實值還可以更簡單

```go
fmt.Println(v.Interface())
```

(因為 `v` 的型別是 `reflect.Type` 而非真正的實值 `float64`，所以這裡不可以用 `fmt.Println(v)`) 既然實值是 `float64` 型別，我們還可以指定浮點數專用的格式：

```go
fmt.Printf("value is %7.1e\n", v.Interface())
```

結果是

```
3.4e+00
```

再提一次，我們不需要做型別斷言，`Printf` 可以從介面變數取得正確的型別來處理它。

簡單來說，`Interface` 方法就是 `refelct.ValueOf` 的相反，只是它回傳的是空介面變數。

總而言之，反射可以在介面變數和反射物件間彼此轉換。

# 反射第三法則

## 3. 想賦值的話，反射物件必需是可賦值的

第三法則是最容易混淆的，不過從頭說起其實很簡單。

以下程式碼是錯誤的，但非常值得研究一番：

```go
var x float64 = 3.4
v := reflect.ValueOf(x)
v.SetFloat(7.1) // Error: will panic.
```

執行這些程式會噴錯

```
panic: reflect.Value.SetFloat using unaddressable value
```

問題不是 `7.1` 無法定位 (unaddressable)，而是 `v` 不可賦值。不是每個 `reflect.Value` 都可賦值的。

你可以用 `CanSet` 方法來偵測可賦值性

```go
var x float64 = 3.4
v := reflect.ValueOf(x)
fmt.Println("settability of v:", v.CanSet())
```

結果是

```
settability of v: false
```

對不可賦值的 `Value` 呼叫 `Set` 系列方法就會噴這種錯誤。

可賦值性有點像是變數定位問題，但更狹義。它定義了一個反射物件是否可以修改它代表的實值。如果反射物件代表的是實值的本體而非副本，那反射物件就是可賦值的。

```go
var x float64 = 3.4
v := reflect.ValueOf(x)
```

由於參數傳遞的是副本，所以 `v` 代表的其實不是真正的 `x`，而是 `x` 的副本，所以

```go
v.SetFloat(7.1)
```

必須噴錯，否則只有 `x` 的副本的值會改變，`x` 仍然會是 `3.4`。這顯然並非我們的原意，也會造成許多困擾。

如果你覺得這很詭異，想想一般的函式

```go
f(x)
```

在 `f` 裡更改 `x` 的值不應該影響真正的 `x`，`f` 接收到的是 `x` 的副本。如果想要在 `f` 裡影響真正的 `x`，應該要

```go
f(&x)
```

這應該比較好理解，而這正是反射的工作方式。如果想對某 `Value` 賦值，就要用指標來建立 `Value`。那麼我們來試看看

```go
var x float64 = 3.4
p := reflect.ValueOf(&x) // Note: take the address of x.
fmt.Println("type of p:", p.Type())
fmt.Println("settability of p:", p.CanSet())
```

結果是

```
type of p: *float64
settability of p: false
```

之所以 `p` 不可賦值，是因為 `p` 代表的是 `&x`，而我們想修改的是 `x`，這可以用 `Elem` 方法取得

```go
v := p.Elem()
fmt.Println("settability of v:", v.CanSet())
```

現在 `v` 就是可賦值的了

```
settability of v: true
```

而且 `v` 代表的是 `x`，所以我們可以

```go
v.SetFloat(7.1)
fmt.Println(v.Interface())
fmt.Println(x)
```

結果當然是

```
7.1
7.1
```

反射可能不太好理解，但它十分忠實地反映了 Go 的工作模式。簡單來說，要用指標建麼的 `Value` 才可賦值。

# Struct

在前一節，`v` 本身不是指標，它只從某個指標轉化而來的。當你想修改某個 `struct` 的成員時就會用到這點：既然你能取得 `struct` 的位址，當然也可以修改它的成員。

以下範例分析某個 `struct` 變數 `t`。我們先用指標建立 `Value`，因為等下會去修改 `t` 的成員。然後把 `t` 的型別儲存到 `typeOfT` 裡，再呼叫一些相關的方法列舉出 `t` 的成員。要注意雖然我們是從 `struct` 的型別取得成員名稱，但成員的值本身仍然是 `Value`。

```go
type T struct {
    A int
    B string
}
t := T{23, "skidoo"}
s := reflect.ValueOf(&t).Elem()
typeOfT := s.Type()
for i := 0; i < s.NumField(); i++ {
    f := s.Field(i)
    fmt.Printf("%d: %s %s = %v\n", i,
        typeOfT.Field(i).Name, f.Type(), f.Interface())
}
```

結果是

```
0: A int = 23
1: B string = skidoo
```

有個重點剛剛沒有提到：`T` 型別的所有成員都是大寫開頭的，因為只有公開成員才是可賦值的。

既然 `s 是可賦值的，我們當然可以用 `s` 來修改 `t` 的 (可賦值的) 成員

```go
s.Field(0).SetInt(77)
s.Field(1).SetString("Sunset Strip")
fmt.Println("t is now", t)
```

結果是

```
t is now {77 Sunset Strip}
```

如果 `s` 是用 `t` 建立的，那 `SetInt` 跟 `SetString` 就會噴錯了。

# 結論

再說一次反射三大法則

1. 反射可以把介面變數轉成對應的反射物件
2. 反射可以把反射物件轉成介面變數
3. 想賦值的話，反射物件必需是可賦值的

只要理解了這三大法則，反射雖然還是有點複雜，但會變得很好用了。反射機制極為強大，所以要小心使用，不到萬不得已最好別用。

還有很多反射相關的議題我們沒有討論到：傳送或接收 `channel` 的資料、配置變數、`slice` 和 `map` 的操作、呼叫函式及方法等，不過篇幅已經夠長了。我們會再後續的文章裡再探討一些相關的議題。

原作者：Rob Pike
改譯者：Ronmi Ren
