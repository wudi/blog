---
slug: optimizations-of-go-compiler-while-function-return-structure-by-value
title: Go 语言中函数按照值返回结构体，编译器会做哪些优化？
---

## 大小分析 Size Analysis

Go 编译器分析结构体的大小。小型结构体(通常最多几个字大小)是优化的候选对象。
```go
type SmallToken struct {
    Type  byte
    Value uint16
}

type LargeToken struct {
    Type    byte
    Value   uint16
    Data    [1024]byte
}

func GetSmallToken() SmallToken { /* ... */ }  // 可能被优化
func GetLargeToken() LargeToken { /* ... */ }  // 不太可能被优化
```


## 栈分配 Stack Allocation

对于小型结构体,编译器通常在栈上而不是堆上分配它们。栈分配更快,不需要垃圾回收。

```go
func ProcessToken() {
    token := SmallToken{Type: 1, Value: 100}  // 很可能在栈上分配
    // 使用 token...
}
```

## 寄存器使用 Register Usage

非常小的结构体(1-2个字)通常可以完全在 CPU 寄存器中返回,完全避免内存操作。

```go
type Coordinate struct {
    X, Y int32
}

func GetCoordinate() Coordinate {
    return Coordinate{10, 20}  // 可能直接使用寄存器返回
}
```

## 内联 Inlining

编译器可能会内联小函数,包括返回小型结构体的方法。这意味着函数的代码直接插入到调用点,消除了函数调用开销。

```go
func IsPositive(c Coordinate) bool {
    return c.X > 0 && c.Y > 0
}

func Process() {
    coord := GetCoordinate()
    if IsPositive(coord) {  // IsPositive 可能被内联
        // ...
    }
}
```

 - 复制消除 Copy Elision

在某些情况下,编译器可以消除不必要的结构体复制。如果返回的结构体立即被使用然后丢弃,编译器可能会直接在使用它的位置构造它。

```go
func UseToken(t SmallToken) int {
    return int(t.Type) + int(t.Value)
}

func Process() {
    result := UseToken(GetSmallToken())  // 可能直接在 UseToken 的参数位置构造 token
    // ...
}
```

 - 逃逸分析 Escape Analysis

编译器执行逃逸分析以确定结构体是否需要在堆上分配。不逃逸出函数作用域的小型结构体可以留在栈上。

```go
func CreateToken() SmallToken {
    return SmallToken{Type: 1, Value: 100}  // 不会逃逸,留在栈上
}

func CreateTokenPtr() *SmallToken {
    return &SmallToken{Type: 1, Value: 100}  // 可能逃逸到堆上
}
```

 - 优化调用者代码 Optimizing Caller Code

编译器可以优化调用代码。如果调用者立即使用返回的结构体的字段,编译器可能会生成直接访问这些字段的代码,而不完全构造结构体。

```go
func Process() {
    token := GetSmallToken()
    if token.Type == 1 {  // 编译器可能优化掉完整的 token 构造
        // ...
    }
}
```

 - 向量化 Vectorization

对于涉及多个小型结构体的操作,编译器可能会使用 SIMD (单指令多数据)指令并行处理它们。

```go
func SumCoordinates(coords []Coordinate) int {
    sum := 0
    for i := 0; i < len(coords); i++ {
        sum += coords[i].X + coords[i].Y  // 可能使用 SIMD 指令并行处理
    }
    return sum
}
```

 - 填充优化 Padding Optimization

编译器可能会重新排序结构体中的字段以最小化填充并优化内存布局,改善缓存使用。

```go
type OptimizedStruct struct {
    a int64  // 8 字节
    b int32  // 4 字节
    c int16  // 2 字节
    d int8   // 1 字节
}  // 总共 15 字节,但会被填充到 16 字节
```

 - 常量传播 Constant Propagation

如果返回的结构体的某些字段是常量或可以在编译时确定,编译器可能会预先计算这些值。

```go
const TokenTypeID = 1

func GetIDToken() SmallToken {
    return SmallToken{Type: TokenTypeID, Value: 100}  // Type 字段可能在编译时就确定
}
```

参考文档：
 - https://stackoverflow.com/questions/1422149/what-is-vectorization
