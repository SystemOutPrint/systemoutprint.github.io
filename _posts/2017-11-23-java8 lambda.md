---
layout: post
title:  "Java8 Lambda"
date:   2017-11-23
author:       "CaiJiahe"
header-img:   "img/tag-bg.jpg"
header-mask:  0.3
catalog:      true
multilingual: true
tags:
    - Java
    - lambda
---

## 0x01 foreach
遍历执行
```java
// output 123
List<Integer> list = Arrays.asList(1, 2, 3);
list.stream().forEach(System.out::print);
```


## 0x02 map
将流中的A元素转换成B
```java
// output 1, 2, 3, 
List<Integer> list = Arrays.asList(1, 2, 3);
list.stream().map(i -> String.valueOf(i) + ", ").forEach(System.out::print);
```

## 0x03 flatmap
将流中中嵌套的流展开。
```java
// output 123456
Map<Integer, List<Integer>> map = new HashMap<>();
map.put(1, Arrays.asList(1, 2, 3));
map.put(2, Arrays.asList(4, 5, 6));
map.entrySet().stream().flatMap(e -> e.getValue().stream()).forEach(System.out::print);
```

# 0x04 filter
过滤元素
```java
// output 12
List<Integer> list = Arrays.asList(1, 2, 3);
list.stream().filter(i -> i < 3).forEach(System.out::print);
```
## 0x05 count
统计流中元素个数
```java
// output 3
List<Integer> list = Arrays.asList(1, 2, 3);
long count = list.stream().count();
System.out.println(count);
```

## 0x06 max/min
返回流中的最大值和最小值
```java
// output 3
List<Integer> list = Arrays.asList(1, 2, 3);
Optional<Integer> max = list.stream().max((i, j) -> i - j);
if (max.isPresent()) {
	System.out.println(max.get());
}
```

## 0x07 findFirst
找到流中第一个匹配的元素
```java
// output 2
List<Integer> list = Arrays.asList(1, 2, 3);
Optional<Integer> o = list.stream().filter(i -> i > 1).findFirst();
if (o.isPresent()) {
	System.out.println(o.get());
}
```

## 0x08 findAny
找到流中任意一个匹配的元素
```java
// output NOT SURE
List<Integer> list = Arrays.asList(1, 2, 3, 4, 5, 6);
Optional<Integer> o = list.stream().parallel().filter(i -> i > 1).findAny();
if (o.isPresent()) {
	System.out.println(o.get());
}
```

## 0x09 anyMatch
filter + findAny的结合体

## 0x0A allMatch
判断流中所有元素都匹配条件
```java
// output true
List<Integer> list = Arrays.asList(1, 2, 3, 4, 5, 6, 7);
boolean isGtZero = list.stream().allMatch(i -> i > 0);
System.out.println(isGtZero);
```

## 0x0B noneMatch
判断流中元素都不匹配条件
```java
// output true
List<Integer> list = Arrays.asList(1, 2, 3, 4, 5, 6, 7);
boolean isGt8 = list.stream().noneMatch(i -> i > 8);
System.out.println(isGt8);
```

## 0x0C reduce
对流中所有元素依次执行某操作
```java
// output 28
List<Integer> list = Arrays.asList(1, 2, 3, 4, 5, 6, 7);
Integer sum = list.stream().reduce(0, (i, j) -> i + j);
System.out.println(sum);
```

## 0x0D collect
收集流中的元素

#### collect(Supplier<R>, BiComsumer<R>, BiComsumer<R, R>):R
* Supplier<R> 一个能创建目标类型实例的方法。
* BiComsumer<R> 一个将元素添加到目标实例的方法。
* BiComsumer<R, R> 将两个元素整合到一起的方法。

#### collect(Collector):R
* Collectors.toCollection(集合构造函数) 构造一个集合
* Collectors.toList() 构造一个list
* Collectors.toSet() 构造一个set
* Collectors.toMap(getKey(), getValue(), keyRepeat()) 构造一个map
* Collectors.joining() 将字符串连接起来
* Collectors.joining(分隔符) 将字符串和分隔符连接起来
* Collectors.groupingBy(组名) 根据组名将流分为N个组，返回值的key值是组名，value是一个List
* Collectors.partitioningBy(判断条件) 根据组名将流分为2个组，返回值的key值是true和false，value是一个List

```java
List<Integer> list = Arrays.asList(1, 2, 3, 4, 5, 6, 7);
HashSet<Integer> hs = list.stream().collect(Collectors.toCollection(HashSet::new));
```

## 0x0E distinct
不改变流中元素顺序，去除流中重复元素。
```java
// output aabb
Stream<String> words = Stream.of("aa", "aa", "bb");
words.distinct().forEach(System.out::print);
```

## 0x0F sorted
对流中元素进行排序。也可以提供一个Comparator。
```java
// output aabbcc
Stream<String> words = Stream.of("bb", "aa", "cc");
words.sorted().forEach(System.out::print);

// output aaabbc
words = Stream.of("bb", "aaa", "c");
words.sorted((str1, str2) -> str2.length() - str1.length()).forEach(System.out::print);
```