+++
title="Go 1.21 slices.Contains()"
date=2023-11-03
updated=2023-11-03
description="Go 1.21 버전에 추가된 slices 패키지를 사용하여 slice에 특정 값이 있는지 확인하는 방법을 알아봅니다."
[taxonomies]
tags=["go", "slice"]
[extra]
giscus = true
quick_navigation_buttons = true
toc = true
+++

# `slices` package

go 1.21 버전에 슬라이스를 더 잘 사용하기 위해 slices 패키지가 추가됐습니다.

파이썬의 `in` 과 비슷한 기능이죠

```python
if "red" in ["red", "green", "blue"]:
	print("Found it!")
```

기존에는 slice를 순회하는 코드를 구현해야 했지만 다음과 같이 쓸 수 있습니다.

얼른 여러 코딩테스트 플랫폼들에 (특히 프로그래머스...) 1.21버전이 지원됐으면 좋겠습니다.

```go
if slices.Contains([]string{"red", "green", "blue"}, "red") {
	fmt.Println("Found it!")
}
```

`slices.Contains()` 의 동작 방식([깃허브](https://github.com/golang/go/blob/434af8537ea73f66f0d2b5a29806516b4b6207ab/src/slices/slices.go#L115))은 다음과 같습니다

```go
// 첫 value의 인덱스를 리턴합니다.
// 없으면 -1 리턴합니다
func Index[S ~[]E, E comparable](s S, v E) int {
	for i := range s {
		if v == s[i] {
			return i
		}
	}
	return -1
}

...

// value가 있으면 true를 리턴합니다
func Contains[S ~[]E, E comparable](s S, v E) bool {
	return Index(s, v) >= 0
}
```

그래서 이 기능이 효율적인지 궁금해서 벤치마킹을 돌려봤습니다.

```
➜  go-slice-benchmark go test -bench=. main_test.go -benchtime 100000000x

goos: darwin
goarch: arm64
BenchmarkContainsLen-10        	100000000	       116.3 ns/op
BenchmarkContainsRange_V-10    	100000000	        97.4 ns/op
BenchmarkContainsRange_I-10    	100000000	       128.3 ns/op
BenchmarkContainsMap-10        	100000000	        43.7 ns/op
BenchmarkContainsSlices-10     	100000000	       100.3 ns/op
PASS
ok  	command-line-arguments	49.241s
```

총 5가지 방식으로 진행했는데 자료 구조를 Map으로 변경해서 테스트한 케이스만 제외하면 `slices.Index()` 처럼 range로 value를 사용한 방식의 성능이 가장 좋았습니다.

다음은 벤치마킹을 진행한 코드입니다.

```go
package main

import (
	"log/slog"
	"math/rand"
	"slices"
	"testing"
)

var champions = []string{
	"Aatrox",
    ...
}

func BenchmarkContainsLen(b *testing.B) {
	for i := 0; i < b.N; i++ {
		c := champions[rand.Intn(len(champions)-1)]

		var ok bool = false
		for j := 0; j < len(champions); j++ {
			if champions[j] == c {
				ok = true
				break
			}
		}
		slog.Bool("", ok)
	}
}

func BenchmarkContainsRange_V(b *testing.B) {
	for i := 0; i < b.N; i++ {
		c := champions[rand.Intn(len(champions)-1)]

		var ok bool = false
		for _, champion := range champions {
			if champion == c {
				ok = true
				break
			}
		}
		slog.Bool("", ok)
	}
}

func BenchmarkContainsRange_I(b *testing.B) {
	for i := 0; i < b.N; i++ {
		c := champions[rand.Intn(len(champions)-1)]

		var ok bool = false
		for j, _ := range champions {
			if champions[j] == c {
				ok = true
				break
			}
		}
		slog.Bool("", ok)
	}
}

func BenchmarkContainsMap(b *testing.B) {
	m := make(map[string]bool, len(champions))

	for _, champion := range champions {
		m[champion] = true
	}

	for i := 0; i < b.N; i++ {
		c := champions[rand.Intn(len(champions)-1)]

		_, ok := m[c]
		slog.Bool("", ok)
	}
}

func BenchmarkContainsSlices(b *testing.B) {
	for i := 0; i < b.N; i++ {
		c := champions[rand.Intn(len(champions)-1)]

		ok := slices.Contains(champions, c)
		slog.Bool("", ok)
	}
}

```