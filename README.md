# GoTestExample [![Build Status](https://api.travis-ci.org/kevingo/GoTestExample.png?branch=master)](https://travis-ci.org/kevingo/GoTestExample) [![codecov.io](http://codecov.io/github/kevingo/GoTestExample/coverage.svg?branch=master)](http://codecov.io/github/kevingo/GoTestExample?branch=master)


這個專案主要用來記錄如何在 golang 裡面寫測試，並且整合 [Travis CI](https://travis-ci.org/) 和 [codecov](https://codecov.io)。

### 寫 function

首先，假設我們寫了一個 Division 的 function 用來處理兩個浮點數相除，並且判斷除數為零的時候要拋出 error，可以這樣寫，並把這個檔案存成 `calculator.go`：

```
package function

import "errors"

func Division(a, b float64) (float64, error) {
	if b == 0 {
		return 0, errors.New("除數不可為零")
	}

	return a / b, nil
}

```

### 寫測試

接著我們要針對這個 function 進行測試，產生一個 `calculator_test.go` 的檔案來寫我們的測試程式。在 go 中，每一個檔案只要加上 `_test` 就是對應的測試程式。

```
package function

import "testing"

func Test_Division(t *testing.T) {
	if i, e := Division(6, 2); i != 3 || e != nil {
		t.Error("Fail to pass")
	} else {
		t.Log("Pass")
	}
}

func Test_Division_Zero(t *testing.T) {
	if _, e := Division(6, 0); e == nil {
		t.Error("Should not go here.")
	} else {
		t.Log("Check zero correctlly.")
	}
}
```

在這裡我們要測試兩個情境：

- 測試 6 / 2 要等於 3
- 測試 6 / 0 會拋出 error

你可以先在 local 端使用 `go test -v ./function/` 指令進行測試：

```
$ go test -v ./...
=== RUN   Test_Division
--- PASS: Test_Division (0.00s)
       	calculator_test.go:9: Pass
=== RUN   Test_Division_Zero
--- PASS: Test_Division_Zero (0.00s)
       	calculator_test.go:17: Check zero correctlly.
PASS
ok     	GoTestExample/function 	0.021s
```

確定沒問題後，就完成基本的測試。`-v` 的參數代表開啟 debug mode，你可以看到每個測試運作的情形。

### 寫壓力測試

golang 測試的項目還可以包含性能的測試，只要在設測試 function 中，以 `Benchmark` 開頭的開頭的 function 就代表一個壓力測試的側項。針對針對 Division 的函式，我們可以寫一個很簡單的測試函式如下：

```
func BenchmarkDivision(b *testing.B) {
	for i:=0 ; i<b.N ; i++ {
		Division(6, 2);
	}
}
```

執行測試時，golang 會幫我們自動執行 `b.N` 次來計算測試的結果，你會看到類似如下的結果：

```
$ go test ./function/ -bench=. -cpu=1,2,4,8
BenchmarkDivision      	2000000000     	         0.70 ns/op
BenchmarkDivision-2    	2000000000     	         0.69 ns/op
BenchmarkDivision-4    	2000000000     	         0.82 ns/op
BenchmarkDivision-8    	2000000000     	         0.69 ns/op
```

## 使用 Travis CI 進行自動測試

你可以在每次的 commit 時，透過 Travis CI 進行測試，整合的方式很簡單，首先，Travis CI 是透過 yaml 進行設定，在你的專案根目錄下，新增一個 `.travis.yml` 的檔案，寫入：

```
language: go

sudo: false

go:
  - 1.6
  - 1.7

before_script:
  - go fmt ./...
  - go vet $(go list ./...)

script:
  - go test ./function/
  - go test ./function/ -bench=. -cpu=1,2,4,8

```

這裡的意思是，讓 Travis CI 幫你測試 1.6 和 1.7 版的 golang 是不是可以正確跑測試。

在跑測試前，我們做 `go fmt` 和 `go vet` 兩件事情。
- `go fmt` 會自動格式化好你的程式碼
- `go vet` 則是會針對程式碼進行靜態分析，找出一些可能的潛在錯誤。

這部分你不寫，也是可以的，只是加上去會讓你在跑測試前，先透過 golang 自帶的工具幫你處理掉一些低級錯誤。

接著是測試的部分，我們主要執行兩行指令：
- `go test ./function/`：我們主要測試 function 這個這個 package 裡面的函式。
- `go test ./function/ -bench=. -cpu=1,2,4,8`：跑壓力測試。

接著你在 Travis CI 裡面，當你有新的 commit 時，就會看到你的專案正在進行測試：

![image](https://github.com/kevingo/blog/raw/master/screenshot/travis.png)

## 整合 codecov

codecov 是用來檢查 code coverage 的一個線上服務，和 Travis CI 以及 github 都整合的很好，很容易就可以與你的專案整合起來了。

首先，你只需要將你的 `.travis.yml` 改成：

```
language: go

sudo: false

go:
  - 1.6
  - 1.7

before_script:
  - go fmt ./...
  - go vet $(go list ./...)

script:
  - go test -coverprofile=coverage.txt -covermode=atomic ./...

after_success:
  - bash <(curl -s https://codecov.io/bash)
```

在跑測試的時候，利用 go test 指令自帶的一個 `coverprofile` 多產生一個 `coverage.txt` 的檔案，並且指定 `covermode=atomic`。
`covermode` 共有 `set`、`count` 和 `atomic` 三種模式，`atomic` 指定每個測試測試 statement 在並發的狀況下跑了幾次。最後，在測試成功後增加一行 `bash <(curl -s https://codecov.io/bash)`，就這麼簡單！

如果你登入到 [codecov.io](https://codecov.io)，還可以看到 code coverage 的數據，也會顯示你的測試涵蓋了哪幾個部分，相當方便。

![image](https://github.com/kevingo/blog/raw/master/screenshot/codecov.png)

## References
- [codecov example-go](https://github.com/codecov/example-go)
- [The Go Blog - The cover story](https://blog.golang.org/cover)
- [golang test package](https://golang.org/cmd/go/#hdr-Test_packages)
