---
title: "在 Java 里如何让方法只执行一次？"
categories: [ "Java" ]
tags: ["Java","并发"]
draft: false
slug: "how-does-java-make-methods-execute-only-once"
date: "2023-04-02T13:18:00+08:00"
---

最近一年时间一直在写 Golang ，也算是对 Golang 有了初步的掌握，再次写 Java 的时候发现有点生疏了，写代码的时候也不自觉代入了写 Golang 的思维。

正如我想要在 Java 里面想让某一个方法只执行一次的时候，我第一时间想到了 Golang 里面的 `Once` 功能。

> `sync.Once` 是 Golang 的一个并发原语，它提供了一种安全地在多个 goroutine 中执行某个函数（或代码块）一次的机制。
> 
> `sync.Once` 类型有一个 `Do` 方法，该方法接收一个函数作为参数，并确保这个函数只会被执行一次，无论有多少个 goroutine 同时调用它。具体来说，第一个调用 Do 方法的 goroutine 会执行这个函数，而其他 goroutine 则会等待它完成，然后返回相同的结果。
>
> `sync.Once` 可以用于一些需要全局初始化的场景，比如初始化配置信息、数据库连接等。使用 sync.Once 可以确保这些初始化只会被执行一次，并且可以安全地被多个 goroutine 共享使用。
> 
> -- 来自 ChatGPT

其实和`单例模式`差不多，但我想要的是让方法只执行一次，我魔改了一下，直接上代码吧。

```java
package cn.typesafe.sync;

import lombok.SneakyThrows;
import java.util.concurrent.Callable;

public class Once<T> {
	private volatile T t = null;

	@SneakyThrows
	public T doOnce(Callable<T> action) {
		if (t == null) {
			synchronized (this) {
				if (t == null) {
					t = action.call();
				}
			}
		}
		return t;
	}
}
```

单元测试

```java
package cn.typesafe.sync;

import cn.typesafe.sync.Once;
import org.junit.jupiter.api.Test;
import org.wildfly.common.Assert;

import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicInteger;

public class OnceTest {

	@Test
	public void onceTest() {
		Once<Integer> once = new Once<>();
		AtomicInteger count = new AtomicInteger(0);

		Integer value1 = once.doOnce(count::incrementAndGet);

		Assert.assertTrue(value1.equals(1));

		Integer value2 = once.doOnce(count::incrementAndGet);

		Assert.assertTrue(value2.equals(1));
	}

	@Test
	public void onceConcurrentTest() throws InterruptedException {

		Once<Integer> once = new Once<>();
		AtomicInteger count = new AtomicInteger(0);

		int threadCount = Runtime.getRuntime().availableProcessors() * 2;
		ExecutorService executorService = Executors.newFixedThreadPool(threadCount);
		CountDownLatch countDownLatch = new CountDownLatch(threadCount);

		for (int i = 0; i < threadCount; i++) {
			executorService.execute(() -> {
				Integer value1 = once.doOnce(count::incrementAndGet);

				Assert.assertTrue(value1.equals(1));
				countDownLatch.countDown();
			});
		}


		countDownLatch.await();
	}
}
```