# CPU分析

```go
package main

func main() {
    iterateLong()
    iterateShort()
}

func iterateLong() {
    iterate(9_000_000_000)
}

func iterateShort() {
    iterate(1_000_000_000)
}

func iterate(iterations int) {
    for i := 0; i < iterations; i++ {
    }
}
```


## 参考文档
[1 cpu profiling](https://www.parca.dev/docs/profiling-101)