---
layout: post
title: "[翻译] Optimizing Concurrent Map Access in Go"
tags: [翻译, golang]
comments: true
date: 2018-04-17 22:00:00+08:00
---


翻译文章, 原文来自:
```
https://misfra.me/optimizing-concurrent-map-access-in-go/
```

## 优化go中的map并发读写

在我的时序存储引擎Catena中，最容易引起争议的代码是根据key获取一个metricSource对象的函数。每次插入操作会至少调用这个函数一次，但实际上，他有可能会调用成百上千次。这个操作在很多goroutine中会发生，所以我们需要做一些确保同步的操作。

这个函数的目的是根据名称去获取一个对象的指针。如果它不存在，它会立刻创建一个对象，以该key插入map，并且返回他的指针。这个数据库的主要结构是 map[string]*metricSource 。需要关注的重点是，对象只会往该map中插入，没有更新和删除。

下面是一段代码说明。为了省点空间看起来简洁，我去掉了函数头和返回语句。

```
var source *memorySource
var present bool

p.lock.Lock() // lock the mutex
defer p.lock.Unlock() // unlock the mutex at the end

if source, present = p.sources[name]; !present {
	// The source wasn't found, so we'll create it.
	source = &memorySource{
		name: name,
		metrics: map[string]*memoryMetric{},
	}

	// Insert the newly created *memorySource.
	p.sources[name] = source
}

```

我做了一个测试去往该数据库中插入指针，每次插入都会去调用该函数去获取它需要去更新的 metric source 的指针。

在4个goroutine并发 (即:GOMAXPROCS设置为4) 运行插入的情况下，可以达到  1,400,000 inserts / sec 。这看起来很快，但他却比只有一个goroutine运行时要慢。对，这时候，你应该会想到是锁的原因。

所以，问题原因是什么？让我们来考虑一个简单的情况，就是对于这个map没有插入操作。假设goroutine 1 想去获取 "a" 资源而goroutine 2 想去获取 "b" 资源，并且a和b都已经存在在map中。从引用的代码中来看，一个运行的goroutine会获取到map的锁、拿到指针、释放锁，这样的操作一直重复。而此时，另外一个goroutine会因为等待获取锁而被堵塞住。在等待持有锁上看起来会浪费很多时间。当你增加goroutine并行运行的时候，这种情况会越来越糟。

一种方法让这种情况变快就是把锁移除，同时保证只有一个goroutine会操作这个map。这种方式很简单，但是这意味着你放弃了程序的拓展性。下面来介绍一个替代方法，它同样足够简单并且保证了线程并发安全。

新的方法只增加了一行代码，但它会保证你在拓展goroutine数量时速度能够得到提升。

```
var source *memorySource
var present bool

if source, present = p.sources[name]; !present { // added this line
	// The source wasn't found, so we'll create it.

	p.lock.Lock() // lock the mutex
	defer p.lock.Unlock() // unlock at the end

	if source, present = p.sources[name]; !present {
		source = &memorySource{
			name: name,
			metrics: map[string]*memoryMetric{},
		}

		// Insert the newly created *memorySource.
		p.sources[name] = source
	}
	// if present is true, then another goroutine has already inserted
	// the element we want, and source is set to what we want.

} // added this line

// Note that if the source was present, we avoid the lock completely!

```

测试结果 5,500,000 inserts / sec ，比原来快了3.93倍。我是用4个goroutine并发重复这个测试，所以这个速度提升符合预期。

这种方式有效是因为我们不会去删除map中的元素，元素的地址也不会改变。如果CPU缓存了指针地址，即使后面map会因为其他操作而改动，我们也可以安全的操作。需要注意的是，我们仍然需要mutex锁，因为如果没有的话，那么在一个goroutine插入数据的时候，另一个goroutine可能会在过程中进程另一次插入，这样就造成了并发冲突。因此，我们只需要在插入时去竞争锁，这样的话，竞争会小了很多。

我的同事建议我移除 defer 因为 defer 操作会增加栈的开销。我小改动了一下之后，测试结果令我很惊讶。
```
var source *memorySource
var present bool

if source, present = p.sources[name]; !present {
	// The source wasn't found, so we'll create it.

	p.lock.Lock() // lock the mutex
	if source, present = p.sources[name]; !present {
		source = &memorySource{
			name: name,
			metrics: map[string]*memoryMetric{},
		}

		// Insert the newly created *memorySource.
		p.sources[name] = source
	}
	p.lock.Unlock() // unlock the mutex
}

// Note that if the source was present, we avoid the lock completely!

```

这个版本的代码测试结果达到了 9,800,000 inserts / sec ，在我只改了4行代码的情况下，速度是第一版的7倍。


#### 补充：
这种方式真的对吗？很不幸，不对！仍然有并发数据竞争的存在，通过 race detector 可以很容易的发现。在map有被读的情况下，我们无法保证数据的完整性。

接下来是一个 无竞争、线程安全、并且“正确”的版本，使用了 RWMutex，读操作不会相互堵塞，但写操作仍然需要同步。

```
var source *memorySource
var present bool

p.lock.RLock()
if source, present = p.sources[name]; !present {
	// The source wasn't found, so we'll create it.
	p.lock.RUnlock()
	p.lock.Lock()
	if source, present = p.sources[name]; !present {
		source = &memorySource{
			name: name,
			metrics: map[string]*memoryMetric{},
		}

		// Insert the newly created *memorySource.
		p.sources[name] = source
	}
	p.lock.Unlock()
} else {
	p.lock.RUnlock()
}

```

这个版本是上一个版本性能的 93.8% ，效果仍然不错。当然，之前的版本不正确，所以不应该去和上个版本做对比。