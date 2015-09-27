+++
date = "2015-09-27T17:11:23+08:00"
title = "list slice map"
tags = ["go","programming"]
categories = ["go"]
+++

# 實測 Go 語言中 List, slice 和 map 的效能

一般來說，slice 和 map 各有所長，linked list 最慢，這應該算是常識，但究竟差多少？為什麼會有這種情況？本文針對 Go 語言使用內建的 `testing` 套件實際測試。

<!--more-->

## Test code

在實際使用上，刪除特定元件會伴隨著「定位」的動作：你得先找到你要刪除的元素在哪，我把這點也反應在測試碼中。測試代碼會產生一個順序未定義的 linked list / slice / map，隨機刪除其中一個元素之後再加回到末端。

```go
package botgoram

import (
	"container/list"
	"math/rand"
	"testing"
)

var sz int = 100

func BenchmarkList(b *testing.B) {
	l := list.New()
	for i := 0; i < sz; i++ {
		l.PushBack(i)
	}
	rand.Seed(int64(sz))
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		digit := rand.Intn(sz)
		cur := l.Front()
		for ; cur != nil && cur.Value.(int) != digit; cur = cur.Next() {
		}
		if cur == nil {
			panic("FaQ")
		}
		l.Remove(cur)
		l.PushBack(digit)
	}
}

func BenchmarkSlice(b *testing.B) {
	l := make([]int, sz)
	for i := 0; i < sz; i++ {
		l[i] = i
	}
	del := func(l []int, k int) []int {
		ret := l[0:k]
		if k < len(l)-1 {
			ret = append(ret, l[k+1:]...)
		}
		return ret
	}
	rand.Seed(int64(sz))
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		digit := rand.Intn(sz)
		j := 0
		for ; j < len(l) && l[j] != digit; j++ {
		}
		if j > len(l) || l[j] != digit {
			panic("FaQ")
		}
		l = del(l, j)
		l = append(l, digit)
	}
}

func BenchmarkMap(b *testing.B) {
	l := make(map[int]bool)
	for i:=0; i < sz; i++ {
		l[i] = true
	}
	rand.Seed(int64(sz))
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		digit := rand.Intn(sz)
		delete(l, digit)
		l[digit] = true
	}
}
```

## 測試環境

* Go 1.5.1
* Debian jessie / sid
* AMD A6-6400K
* 8G ram

## 測試結果

長度 10: slice 最快

```
$ go1.5 test -bench .
testing: warning: no tests to run
PASS
BenchmarkList-2 	 5000000	       281 ns/op
BenchmarkSlice-2	20000000	        90.3 ns/op
BenchmarkMap-2  	10000000	       157 ns/op
ok  	github.com/Patrolavia/botgoram	5.334s
go1.5 test -bench .  7.16s user 0.16s system 108% cpu 6.770 total
```

長度 100： slice 和 map 差不多

```
$ go1.5 test -bench .
testing: warning: no tests to run
PASS
BenchmarkList-2 	 2000000	       874 ns/op
BenchmarkSlice-2	10000000	       169 ns/op
BenchmarkMap-2  	10000000	       173 ns/op
ok  	github.com/Patrolavia/botgoram	6.430s
go1.5 test -bench .  8.14s user 0.12s system 105% cpu 7.817 total
```

長度 1000： map 最快

```
$ go1.5 test -bench .
testing: warning: no tests to run
PASS
BenchmarkList-2 	  200000	      7833 ns/op
BenchmarkSlice-2	 2000000	       932 ns/op
BenchmarkMap-2  	10000000	       170 ns/op
ok  	github.com/Patrolavia/botgoram	6.351s
go1.5 test -bench .  7.91s user 0.16s system 103% cpu 7.769 total
```

## 分析與假說

三段測試碼實際的動作其實有微妙的不同:

#### map

1. 先對要刪除 key 做 hash 取得 hash value
2. 從內部的搜尋樹中刪去一個節點 (release memory)
3. 對要新增的 key 做 hash
4. 從內部的搜尋樹中加上一個節點 (allocate memory)

在沒有發生 hash collision 之前，理論上時間複雜度應該是 `O(1)`。總計 release 1 次、allocate 1 次、memcpy 0 次

#### slice

由於 Go 的 slice 可以理解成是某種 struct 的參考，所以動作是

1. 循序比對，找出要刪除的是第幾個元素 (這裡的時間複雜度是 `O(n)`)
2. 配置一個指向前半部份的 slice (allocate memory)
3. 複製剛剛那個 slice，調整複製品的容量 (allocate memory)
4. 把後半部份加到剛剛複製出來的 slice (memcpy) (這裡的時間複雜度也是 `O(n)`)
5. 釋放沒有用到的那兩個 slice (release memory)

所以時間複雜度應該是 `O(n)`。總計 release 2 次、allocate 2 次、memcpy 1 次

#### linked list

1. 循序比對，找到要刪除的元素
2. 把前一個元素的 next 指向自己的 next，下一個元素的 prev 指向自己的 prev
3. 必要時更新 list head 跟 tail 的資訊
4. 釋放元素 (release memory)
5. 配置新元素 (allocate memory)
6. 把 tail 的 next 指向新元素，新元素的 prev 指向 tail
7. 更新 list tail 的資訊

所以時間複雜度應該是 `O(n)`。總計 release 1 次、allocate 1 次、memcpy 0 次

### slice VS linked list

根據上面的分析，照理來說應該是 linked list 應該要比 slice 快才對，為什麼實測結果相反？

仔細觀察可以發現 linked list 有兩個問題：

1. 資料結構遠比 slice 複雜
2. 大量使用遠程的函式呼叫

第一點其實差距不大，畢竟 release 跟 allocate 都是在 `O(1)` 的地方執行，也就是固定損耗的概念。

但是第二點是發生在 `O(n)` 的部份，也就是差距會被放大 N 倍。而且函式呼叫的損耗遠比複雜結構配置大得多，甚至會有 context switch 的可能，速度就這樣拖慢了。

### map VS slice

map 的 release 跟 allocate 操作都比 slice 少，但在長度比較小的時候會慢，唯一的可能就是把時間花在計算 hash value 了。

## 結論

[When in doubt, use map.](https://s-media-cache-ak0.pinimg.com/736x/c2/50/19/c2501976d70198ffcbcf681589d94df9.jpg)
