---
title: java8流相关Stream
date: 2018-07-24 23:46:58
tags: java基础
categories: JAVA学习
---

最近去了新公司，发现里面的循环什么的基本都是用stream来写的。就是java8那个流，感觉这种写法优雅，易懂，非常方便，所以在此学习并且记录下来。希望对Stream有一个大体上的认识。

<!-- more --> 

简介:
一个宏观上的介绍：流在管道中传输，并且可以在管道的节点上进行处理，比如筛选，排序，聚合等。
元素流在管道中经过中间操作(intermediate operation)的处理，最后由最终操作(terminal operation)得到前面处理的结果。

流管道示意图:
![流管道示意图](http://pcatcseys.bkt.clouddn.com/18-7-23/75871215.jpg)
> 流的操作分为两种：
> 中间操作方法：像filter这种，返回流对象本身，最终不产生新集合的方法。
> 最终操作方法：像count这种最终会从stream产生值的方法。

两个基础特性：
- **Pipelining** ：中间操作都会返回流对象本身。这样多个操作可以串联成一个管道，如同流式风格。这样可以对操作进行优化，比如延迟执行和短路。
- **内部迭代** ：以前对集合的遍历都是通过Iterator或者For-each的方式，显示的在集合外部进行迭代，这叫做外部迭代，Stream提供了内部迭代的方式，通过访问者模式实现。

为什么不在集合类实现这些操作，而是定义了全新的Stream API？Oracle官方给出了几个重要原因：
一是集合类持有的所有元素都是存储在内存中的，非常巨大的集合类会占用大量的内存，而Stream的元素却是在访问的时候才被计算出来，这种“延迟计算”的特性有点类似Clojure的lazy-seq，占用内存很少。
二是集合类的迭代逻辑是调用者负责，通常是for循环，而Stream的迭代是隐含在对Stream的各种操作中，例如map()。

#### 一、创建Stream

**数据源**
流的来源。可以是集合，数组，I/O channel，产生器generator等。(Map不能作为Stream的源)

```java
// 1.Individual values
Stream<String> s = Stream.of("a", "b", "c");
// 2. Arrays
String [] strArray = new String[] {"a", "b", "c"};
Stream<String> arrs = Stream.of(strArray);
// 3. Collections
List<String> list = Arrays.asList(strArray);
Stream<String> lists = list.stream();
// 4. iterate 
Stream.iterate(0, n -> n + 3).limit(10).forEach(x -> System.out.print(x + " "));  // 0 3 6 9 12 15 18 21 24 27
// 5. generate
//将一个无限的stream限制在10个"test"字符串
String joinStr = Stream.generate(() -> "test").limit(10).collect(Collectors.joining(","));
```
**stream**:集合创建串行流

```java
List<String> strings = Arrays.asList("abc", "", "bc", "efg", "abcd","", "jkl");
List<String> filtered = strings.stream().filter(string -> !string.isEmpty()).collect(Collectors.toList());
```
**parallelStream**:集合创建并行流

```java
List<String> strings = Arrays.asList("abc", "", "bc", "efg", "abcd","", "jkl");
List<String> filtered = strings.parallelStream().filter(string -> !string.isEmpty()).collect(Collectors.toList());
```
#### 二、中间操作

**distinct**:去除重复
```java
List<String> lists = Arrays.asList("1", "2", "3", "3", "follow","wind", "followwwind");
lists.stream().distinct().forEach(p -> System.out.print(p + "\t")); 
//lists.stream().distinct().forEach(p -> System.out.print(System.out::println); 
```
**filter**:过滤元素，返回过滤条件通过的流。
```java
lists.stream().filter(p -> p.length() > 1).forEach(p -> System.out.print(p + "\t")); 
```
**sorted**:流排序，中间操作返回流本身。
```java
lists.stream().filter(str -> str.contains("w"))
      .sorted((str1, str2) -> {
         if (str1.length() == str2.length()) {
            return 0;  
         } else if (str1.length() > str2.length()) {
            return 1;
         } else {
            return -1;
         }
      }).forEach(System.out::println); 
```
**limint**:获取截取的前N个元素，如果原Stream中包含的元素个数小于N，那就获取其所有的元素。
```java
lists.stream().limit(5).forEach(p -> System.out.print(p + "\t"));
```
**skip**:返回一个丢弃原Stream的前N个元素后剩下元素组成的新Stream。
```java
lists.stream().skip(5).forEach(p -> System.out.print(p + "\t")); 
```
**peek**:生成一个包含原Stream的所有元素的新Stream，同时会提供一个消费函数（Consumer实例），新Stream每个元素被消费的时候都会执行给定的消费函数。
```java
lists.stream().peek(p -> {p = p.toUpperCase(); System.out.println(p);}).forEach(System.out::println); 
```
**map**:接受lambda ,将元素转换成其他形式或提取信息。接受一个函数作为参数，该函数会被应用到每个元素上，并将其映射成一个新的元素
```java
lists.stream().map(p -> p + "-->").forEach(System.out::print); 
lists.stream().map(p -> p.split("")).map(p -> {
         String tmp = "";
         if(p.length > 1){
            tmp = p[1];
         }else{
            tmp = p[0];
         }
         return tmp + "\t";
      }).forEach(System.out::print);
```
**flatMap**:和map类似，不同的是其每个元素转换得到的是Stream对象，会把子Stream中的元素压缩到父集合中
```java
lists.stream().flatMap(p -> Stream.of(p.split("www"))).forEach(p -> System.out.print(p + "\t"));
//看看使用flatMap和不用faltMap输出结果的异同。
Stream<List<Integer>> inputStream = Stream.of(
          Arrays.asList(1),
          Arrays.asList(2, 3),
          Arrays.asList(4, 5, 6)
       );
      System.out.println();
      Stream<Integer> outputStream = inputStream.
      flatMap((childList) -> childList.stream());
      outputStream.forEach(p -> System.out.print(p + "\t"));
```
#### 三、最终操作

**forEach**:每个元素匹配
```java
lists.stream().forEach(System.out::print);
```
**match**:流匹配，终结操作
```java
System.out.println(lists.stream().allMatch(str -> str.length() == 3));
System.out.println(lists.stream().anyMatch(str -> str.length() > 5));
System.out.println(lists.stream().noneMatch(str -> str.length() > 5));
```
**count**:
```java
System.out.println(lists.stream().count());
```
**reduce**:可以将流中元素反复结合起来，得到一个值。可以设置一个初始值。
```java
Optional<String> reOptional = lists.stream().reduce((str, str2) -> str + "-->" + str2);
reOptional.ifPresent(System.out::println); //
lists.stream().filter(p -> p.matches("\\d+")).mapToInt(p -> Integer.valueOf(p)).reduce(Integer::sum).ifPresent(System.out::println);
```
**collect**:收集。将流转换为其他形式。接收一个Collector接口的实现，用于给Stream中元素做汇总的方法
```java
List<String> ll = lists.stream().collect(Collectors.toList());
lists.stream().collect(Collectors.maxBy((p1, p2) -> p1.compareTo(p2))).ifPresent(System.out::println); 
lists.stream().collect(Collectors.minBy((p1, p2) -> p1.compareTo(p2))).ifPresent(System.out::println); 
int s = lists.stream().filter(p -> p.matches("\\d+")).collect(Collectors.summingInt(p -> Integer.valueOf(p)));
String liString = lists.stream().collect(Collectors.joining(","));
```
**sum**:做求和汇总
```java
lists.stream().filter(p -> p.matches("\\d+")).mapToInt(p -> Integer.valueOf(p)).sum(); 
```
**findFirst**:查找第一个
```java
Optional<String> firstOptional = lists.stream().findFirst();
firstOptional.ifPresent(System.out::println);
Stream.of().findFirst().ifPresent(System.out::println); 
```
**findAny**:查找随机一个
```java
lists.stream().findAny().ifPresent(System.out::println);
```
**Collectors**:将流转化为集合和聚合元素。可以返回列表或者字符串
```java
List<String>strings = Arrays.asList("abc", "", "bc", "efg", "abcd","", "jkl");
List<String> filtered = strings.stream().filter(string -> !string.isEmpty()).collect(Collectors.toList());
 
System.out.println("筛选列表: " + filtered);
String mergedString = strings.stream().filter(string -> !string.isEmpty()).collect(Collectors.joining(", "));
System.out.println("合并字符串: " + mergedString);
```
**summaryStatistics**:统计，主要作用于int、double、long等类型上，产生各种统计结果
```java
List<Integer> numbers = Arrays.asList(3, 2, 2, 3, 7, 3, 5);
IntSummaryStatistics stats = integers.stream().mapToInt((x) -> x).summaryStatistics();
System.out.println("列表中最大的数 : " + stats.getMax());
System.out.println("列表中最小的数 : " + stats.getMin());
System.out.println("所有数之和 : " + stats.getSum());
System.out.println("平均数 : " + stats.getAverage());
```
分区，分组，多级分组:进阶，高级，复杂操作，暂且略去。

参考文献：
1.https://blog.csdn.net/dongyuancaizi/article/details/78795945
2.http://www.runoob.com/java/java8-streams.html
3.https://blog.csdn.net/followwwind/article/details/78211395
4.https://www.liaoxuefeng.com/article/001411309538536a1455df20d284b81a7bfa2f91db0f223000




