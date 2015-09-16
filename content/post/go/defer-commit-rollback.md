+++
date = "2015-09-16"
title = "defer, Commit and Rollback"
tags = [ "go" ]
+++

使用 transaction 的時候，利用 defer 函式的特性 commit 或 rollback 比較合於 Go 的慣例。理論上應該有兩種形式，第一種是利用 `Commit()` 之後 `Rollback()` 不會真正執行的特性：

<!--more-->

```go
tx, err = db.Begin()
if err != nil {
	return err
}
defer tx.Rollback()

if _, err := tx.Exec(query); err != nil {
	return err
}

tx.Commit()
```

另外一種則是用 `panic` 和 `recover` 來確認在 defer 執行時的狀態是出錯還是正常

```go
tx, err = db.Begin()
if err != nil {
	return
}
defer func() {
	if e := recover(); e != nil {
		tx.Rollback()
		err = e
	} else {
		tx.Commit()
	}
}

if _, err = tx.Exec(query); err != nil {
	panic("Cannot execute sql query")
}
return
```

後者看起來比較潮，但實際測速的結果

```go
func BenchmarkDeferRollback(b *testing.B) {
	fn := func(i int) {
		tx, err := db.Begin()
		if err != nil {
			b.Fatalf("Cannot begin transaction: %s", err)
		}
		defer tx.Rollback()
		q := tx.Stmt(newPadQuery)
		q.Exec(u.ID, fmt.Sprintf("rollback%d", i), "content")
		tx.Commit()
	}
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		fn(i)
	}
}

func BenchmarkDeferIfElse(b *testing.B) {
	fn := func(i int) {
		tx, err := db.Begin()
		if err != nil {
			b.Fatalf("Cannot begin transaction: %s", err)
		}
		defer func () {
			if err := recover(); err != nil {
				tx.Rollback()
			} else {
				tx.Commit()
			}
		}()
		q := tx.Stmt(newPadQuery)
		q.Exec(u.ID, fmt.Sprintf("if-else%d", i), "content")
	}
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		fn(i)
	}
}
```

```
BenchmarkDeferRollback-8          100000             17449 ns/op
BenchmarkDeferIfElse-8            100000             17816 ns/op
```

我跑了大概十來次測試，都是直接 Rollback 比較快，雖然只有快幾 % 而已。猜測原因，可能是 panic 跟 recover 這兩個錯誤處理函式耗去了一些效能吧。
