+++
date = "2015-09-16"
title = "為 function 加上方法來測試 http handler"
tags = [ "go" ]
+++

在寫測試的時候，發現自己一直重複使用 `NewRequest` 跟 `NewRecorder`，所以想到可以在 function 上加上方法來協助處理這些問題。

<!--more-->

```go
type Test func(http.ResponseWriter, *http.Request)

func (f Test) Get(uri, cookie string) (rec *httptest.ResponseRecorder, err error) {
	rec = httptest.NewRecorder()
	req, err := http.NewRequest("GET", uri, nil)
	if err != nil {
		return
	}
	req.Header.Add("Cookie", cookie)
	f(rec, req)
	return
}
```

使用起來像這樣

```go
func TestMyhandler(t *testing.T) {
	resp, err := Test(myhandler).Get("/api/action", "")
	if err != nil {
		t.Fatalf("Failed to create request for /api/action: %s", err)
	}

	if resp.Body.String() != "success" {
		t.Errorf("Request to /api/action failed: %s", resp.Body.String())
	}
}
```
