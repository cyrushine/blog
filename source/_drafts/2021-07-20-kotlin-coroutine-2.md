---
title: 深入分析 Kotlin Coroutines 是如何实现的（二）
date: 2021-07-20 12:00:00 +0800
categories: [Kotlin]
tags: [kotlin, coroutine, 协程]
---

# async / await - 结构化并发编程 (Structured Concurrency)

```kotlin
fun main() = runBlocking {
    val values = (500L..1000L)

    val cost = measureTimeMillis {
        val job1 = async {
            val value = values.random()
            println("job1 delay: $value")
            delay(value)
            value
        }

        val job2 = async {
            val value = values.random()
            println("job2 delay: $value")
            delay(value)
            value
        }

        val job3 = async {
            val value = values.random()
            println("job3 delay: $value")
            delay(value)
            value
        }

        println("result: ${job3.await() + job2.await() + job1.await()}")
    }
    println("measureTimeMillis: $cost")
}

// output:
// job1 delay: 758
// job2 delay: 822
// job3 delay: 873
// result: 2453
// measureTimeMillis: 961
```

# withContext - 线程切换

