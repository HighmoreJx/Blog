# Fibonacci （斐波那契数列实现）

## From
《剑指offer》Question No.9

## Description
Code implementation of fibonacci.

f(0) = 0

f(1) = 1

f(n) = f(n - 1) + f(n - 2)


## Solution

swift
```
func fibonacci(n: Int) -> Int {
    if n <= 0 {
        return 0
    }
    
    if n == 1 {
        return 1
    }
    
    return fibonacci(n: n - 1) + fibonacci(n: n - 2)
}
```

问题: 太多重复计算.

better solution
```
func fibonacci_levelup_1(n: Int) -> Int {
    let result = [0, 1]
    if (n < 2) {
        return result[n]
    }
    
    var fibNumberOne = 0
    var fibNumberTwo = 1
    for _ in 2..<n {
        let temp = fibNumberOne
        fibNumberOne = fibNumberTwo
        fibNumberTwo = temp + fibNumberTwo
    }
    return fibNumberOne + fibNumberTwo
}
```

