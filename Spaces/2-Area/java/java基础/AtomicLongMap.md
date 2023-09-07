---
title: AtomicLongMap
date created: 2023-09-07
date modified: 2023-09-07
---
> [!TIP] 技巧💡  
> 建议使用guava的AtomicLongMap。底层是使用 ConcurrentHashMap + compute方法

统计文本中单词出现的次数，把单词出现的次数记录到一个Map中。

## ConcurrentHashMap<String, Long>

### 先get再put(不可用)

```java
// 如果多个线程并发调用这个increase()方法，increase()的实现就是错误的，因为多个线程用相同的word调用时，很可能会覆盖相互的结果，造成记录的次数比实际出现的次数少。
private final Map<String, Long> wordCounts = new ConcurrentHashMap<>();
 
public long increase(String word) {
    Long oldValue = wordCounts.get(word);
    Long newValue = (oldValue == null) ? 1L : oldValue + 1;
    wordCounts.put(word, newValue);
    return newValue;
}

```

这个主要的问题在于，get虽然是线程安全，但是get和put没有原子性。解决方案，要么加锁，要么cas

### 加锁

```java
private final Map<String, Long> wordCounts = new ConcurrentHashMap<>();
 
public synchronized long increase(String word) {
    Long oldValue = wordCounts.get(word);
    Long newValue = (oldValue == null) ? 1L : oldValue + 1;
    wordCounts.put(word, newValue);
    return newValue;
}
```

### cas

```java
private final ConcurrentMap<String, Long> wordCounts = new ConcurrentHashMap<>();
 
public long increase(String word) {
    Long oldValue, newValue;
    while (true) {
        oldValue = wordCounts.get(word);
        if (oldValue == null) {
            // Add the word firstly, initial the value as 1
            newValue = 1L;
            if (wordCounts.putIfAbsent(word, newValue) == null) {
                break;
            }
        } else {
            newValue = oldValue + 1;
            if (wordCounts.replace(word, oldValue, newValue)) {
                break;
            }
        }
    }
    return newValue;
}
```

实现每次调用都会涉及Long对象的拆箱和装箱操作，很明显，更好的实现方式是采用AtomicLong

### 使用compute方法 类似cas

```java
private final Map<String, Long> wordCounts = new ConcurrentHashMap<>();
 
public synchronized long increase(String word) {
    wordCounts.compute(wordCounts, (k, v) -> (v == null) ? 1 : v+1);
    return newValue;
}

```

底层也是用cas。

## ConcurrentMap<String, AtomicLong>

```java
private final ConcurrentMap<String, AtomicLong> wordCounts = new ConcurrentHashMap<>();
 
public long increase(String word) {
    AtomicLong number = wordCounts.get(word);
    if (number == null) {
        // 存在问题。但是单实例doublecheck的问题
        AtomicLong newNumber = new AtomicLong(0);
        number = wordCounts.putIfAbsent(word, newNumber);
        if (number == null) {
            number = newNumber;
        }
    }
    return number.incrementAndGet();
}
```

## AtomicLongMap

### 源码实现-早期版本(cas)

```java
@CanIgnoreReturnValue  
public long addAndGet(K key, long delta) {  
    AtomicLong atomic;  
    label23:  
    do {  
        atomic = (AtomicLong)this.map.get(key);  
        if (atomic == null) {  
            atomic = (AtomicLong)this.map.putIfAbsent(key, new AtomicLong(delta));  
            if (atomic == null) {  
                return delta;  
            }  
        }  
  
        long oldValue;  
        long newValue;  
        do {  
            oldValue = atomic.get();  
            if (oldValue == 0L) {  
                continue label23;  
            }  
  
            newValue = oldValue + delta;  
        } while(!atomic.compareAndSet(oldValue, newValue));  
  
        return newValue;  
    } while(!this.map.replace(key, atomic, new AtomicLong(delta)));  
  
    return delta;  
}
```

### 源码实现-最新版本(compute)

```java
@CanIgnoreReturnValue  
public long updateAndGet(K key, LongUnaryOperator updaterFunction) {  
  checkNotNull(updaterFunction);  
  return map.compute(  
      key, (k, value) -> updaterFunction.applyAsLong((value == null) ? 0L : value.longValue()));  
}
```

## 参考资料

[ConcurrentHashMap使用示例 - Rozdy - 博客园](https://www.cnblogs.com/Rozdy/p/4359264.html)
