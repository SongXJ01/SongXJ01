---
title: Java Apache Collection4 工具箱使用笔记
date: 2024-10-07 18:58:33
copyright: true
tags: [Java, Collection4, 工具箱]
categories:
- 技术笔记
- Java

top: 97
---

&emsp;&emsp;本篇学习笔记将介绍 Apache Commons Collections 4 等使用。Collections 4 是一个非常有用的库，它提供了许多额外的集合类和工具方法来扩展Java标准库中的集合框架。Apache Commons Collections 4 提供了对 Java 标准集合框架的强大补充，包括新的集合类型、工具方法以及一些算法实现。

<!--more-->

<br/>

&emsp;&emsp;首先，你需要在你的项目中添加 Apache Commons Collections 4 的依赖。如果你使用的是 Maven，可以在 pom.xml 文件中加入如下依赖：

```xml
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-collections4</artifactId>
    <version>4.4</version> <!-- 请检查最新的版本号 -->
</dependency>
```


## Map

### BoundedMap: 固定长度的映射

#### LRUMap

底层是LRU算法。LRU算法的设计原则是：如果一个数据在最近一段时间没有被访问到，那么在将来它被访问的可能性也很小。
LRU算法可以有效地减少内存的使用，并提高缓存的命中率。

```Java
import org.apache.commons.collections4.map.LRUMap;

LRUMap<String, String> lruMap = new LRUMap<>(2);
lruMap.put("key1", "value1");
lruMap.put("key2", "value2");
System.out.println("LRUMap before eviction: " + lruMap);
lruMap.put("key3", "value3"); // 添加新元素，淘汰最少使用的元素
System.out.println("LRUMap after adding key3: " + lruMap);
// LRUMap before eviction: {key1=value1, key2=value2}
// LRUMap after adding key3: {key2=value2, key3=value3}
```

#### FixedSizeMap

`FixedSizeMap` 是一种固定长度的映射，可以一键将一个 Map 转换为定长形式，从而有效防止误操作的发生。

```Java
import org.apache.commons.collections4.map.FixedSizeMap;

Map<String, String> hashMap = new HashMap<>();
hashMap.put("key1", "value1");
hashMap.put("key2", "value2");
FixedSizeMap<String, String> fixedSizeMap = FixedSizeMap.fixedSizeMap(hashMap);
```

类似的还有 `FixedSizeSortedMap`。区别是底层采用 `SortedMap`。

```Java
import org.apache.commons.collections4.map.FixedSizeSortedMap;

TreeMap<String, String> treeMap = new TreeMap<>();
treeMap.put("key1", "value1");
treeMap.put("key2", "value2");
FixedSizeSortedMap<String, String> fixedSizeSortedMap = FixedSizeSortedMap.fixedSizeSortedMap(treeMap);
```

#### SingletonMap

一个只包含一个元素的Map

```Java
import org.apache.commons.collections4.map.SingletonMap;

SingletonMap<String, String> singletonMap = new SingletonMap<>("key1", "value1");
```

### BidiMap: 双向映射

使用双向映射，可以使用值查找键，并且可以使用键轻松查找值。它可以根绝key移除，也可以根据value移除。

```Java
public interface BidiMap<K, V> extends IterableMap<K, V> {}
```

常用方法:

* put(K key, V value)：添加键值对
* getKey(V value)：根据值获取键
* removeValue(V value)：根据值移除键值对
* inverseBidiMap()：返回一个逆向的双向映射
* values()：返回一个Set，包含所有值

#### DualHashBidiMap & DualLinkedHashBidiMap

DualHashBidiMap 底层维护着两个 HashMap，分别用于正向和逆向映射。它实现了高效的插入和删除操作，非常适合对性能要求较高的场景。

```Java
import org.apache.commons.collections4.BidiMap;
import org.apache.commons.collections4.bidimap.DualHashBidiMap;

BidiMap<String, Integer> dualHashBidiMap = new DualHashBidiMap<>();
dualHashBidiMap.put("B", 2);
dualHashBidiMap.put("A", 1);
// DualHashBidiMap (sorted): {A=1, B=2}
```

类似的还有 `DualLinkedHashBidiMap`，它底层采用两个LinkedHashMap存储。可以保持元素的插入顺序，适合需要顺序访问的场景。

```Java
import org.apache.commons.collections4.BidiMap;
import org.apache.commons.collections4.bidimap.DualLinkedHashBidiMap;

BidiMap<String, Integer> dualLinkedHashBidiMap = new DualLinkedHashBidiMap<>();
dualLinkedHashBidiMap.put("B", 2);
dualLinkedHashBidiMap.put("A", 1);
// DualLinkedHashBidiMap: {B=2, A=1}
```

#### TreeBidiMap & DualTreeBidiMap

TreeBidiMap 同样保持元素的顺序，但并不保证双向映射的严格性，适合根据自然排序方式进行访问。
TreeBidiMap 采用两个有序树（通常基于红黑树的实现）来存储键值对：一个树负责存储正向映射（即键到值），另一个树则用于存储反向映射（即值到键）。
这种结构不仅维护了键和值的顺序，还支持高效的查找、插入和删除操作，其时间复杂度约为
\(O(\log n)\)。在排序性能方面，TreeBidiMap 具备优越的表现。

```Java
import org.apache.commons.collections4.BidiMap;
import org.apache.commons.collections4.bidimap.TreeBidiMap;

BidiMap<String, Integer> treeBidiMap = new TreeBidiMap<>();
treeBidiMap.put("B", 2);
treeBidiMap.put("A", 1);
// TreeBidiMap (sorted): {A=1, B=2}
```

类似的还有 `DualTreeBidiMap`。DualTreeBidiMap 实际上是将两个 TreeBidiMap 结合起来，提供一个“视图”，使得在不同的上下文中可以更方便地使用相同的映射关系。
它允许用户在一个 BidiMap 上进行操作，而实际上在内部维护两个 TreeBidiMap。
提供反向映射的功能，但不重复存储相同的键值对，提高了内存使用的效率。

```Java
import org.apache.commons.collections4.BidiMap;
import org.apache.commons.collections4.bidimap.DualTreeBidiMap;

BidiMap<String, Integer> dualTreeBidiMap = new DualTreeBidiMap<>();
dualTreeBidiMap.put("B", 2);
dualTreeBidiMap.put("A", 1);
// DualTreeBidiMap (sorted): {A=1, B=2}
```

### MultiKeyMap: 多键映射

MultiKeyMap 能够有效解决我们在日常开发中经常遇到的一个痛点。举例来说，当我们的 Map
的键由多个字段的值联合组成时（这有些类似于联合索引的概念），我们通常会选择手动拼接字符串来构建键并进行存储。然而，有了
MultiKeyMap，我们可以更加优雅地解决这一问题。

```Java
import org.apache.commons.collections4.map.MultiKeyMap;

MultiKeyMap<String, String> multiKeyMap = new MultiKeyMap<>();
multiKeyMap.put("User1", "Info1", "ProductA");
multiKeyMap.put("User1", "Info2", "ProductB");
```

### MultiValuedMap: 多值映射

MultiValuedMap 允许一个键对应多个值，其内部的数据结构逻辑由它自行维护。我们常用的 `Map<String, List<String>>`
这种数据结构可以被其所替代，使用起来极为便捷。

#### ArrayListValuedHashMap

允许重复的值，并且保持值的插入顺序。

```Java
import org.apache.commons.collections4.MultiValuedMap;
import org.apache.commons.collections4.multimap.ArrayListValuedHashMap;

MultiValuedMap<String, String> arrayListMap = new ArrayListValuedHashMap<>();
arrayListMap.put("keyA", "valueA1");
arrayListMap.put("keyA", "valueA1");
arrayListMap.put("keyA", "valueA2");
arrayListMap.put("keyB", "valueB1");
// ArrayListValuedHashMap: {keyA=[valueA1, valueA1, valueA2], keyB=[valueB1]}
```

#### HashSetValuedHashMap

对每个键只存储唯一的值。即使我们插入相同的值 valueA1 多次，它也只会存储一次。

```Java
import org.apache.commons.collections4.MultiValuedMap;
import org.apache.commons.collections4.multimap.HashSetValuedHashMap;

MultiValuedMap<String, String> hashSetMap = new HashSetValuedHashMap<>();
hashSetMap.put("keyA", "valueA1");
hashSetMap.put("keyA", "valueA1"); // 重复的值将被忽略
hashSetMap.put("keyA", "valueA2");
hashSetMap.put("keyB", "valueB1");
// HashSetValuedHashMap: {keyA=[valueA2, valueA1], keyB=[valueB1]}
```

### MapUtils: 工具类

#### debugPrint

打印Map的详细内容

```Java
Map<String, Object> map = new TreeMap<>();
map.put("key1", 1);
map.put("key2", 2);
map.put("key3", 3);
MapUtils.debugPrint(System.out, "map", map);
```

输出：

```
map = 
{
    key1 = 1 java.lang.Integer
    key2 = 2 java.lang.Integer
    key3 = 3 java.lang.Integer
} java.util.TreeMap
```

#### verbosePrint

打印Map的内容

```Java
Map<String, Object> map = new TreeMap<>();
map.put("key1", 1);
map.put("key2", 2);
map.put("key3", 3);
MapUtils.verbosePrint(System.out, "map", map);
```

输出：

```
map = 
{
    key1 = 1
    key2 = 2
    key3 = 3
}
```

#### pullAll

快速构建一个新的Map

```Java
Map<String, Object> map = MapUtils.putAll(new HashMap<>(), new String[]{
        "name", "Xiao Ming", "age", "1", "sex", "male"
});
```

```Java
Map<String, Object> map2 = MapUtils.putAll(new HashMap<>(), new String[][]{
        {"name", "Xiao Ming"},
        {"age", "1"},
        {"sex", "male"}
});
// map2 = {sex=male, name=Xiao Ming, age=1}
```

#### lazyMap

`lazyMap` 方法的主要目的是在需要的时候才生成 Map 的值，而不是在创建 Map 时就填充所有的值。这种方式可以有效地延迟计算，只有在真正需要访问某个值时，才进行初始化和计算。


```Java
IterableMap<String, Object> result = MapUtils.lazyMap(new HashMap<>(), (Factory<Object>) Date::new);
result.put("key1", "value1");
result.get("key2");
// map = {key1=value1, key2=Wed Sep 25 13:49:42 CST 2024}
```

```Java
Transformer<String, Integer> transformer = Integer::parseInt;
Map<String, Integer> map = MapUtils.lazyMap(new HashMap<>(), transformer);
map.get("1");
map.get("2");
// map = {1=1, 2=2}
```

#### populateMap

能很方便向 Map 里面放值，并且支持定制化 `key` 和 `value`。

```Java
List<String> names = List.of("1", "2", "3", "4", "5");
Map<String, Object> map = new HashMap<>();
MapUtils.populateMap(map, names, e -> "key" + e);
// map = {key1=1, key2=2, key5=5, key3=3, key4=4}
```

#### invertMap

对调 `key` 和 `value` 的值。

```Java
Map<String, String> invertMap = new HashMap<>();
invertMap.put("key1", "value1");
invertMap.put("key2", "value2");
Map<String, String> inverted = MapUtils.invertMap(invertMap);
// invertMap: {value2=key2, value1=key1}
```

#### iterableMap

构建一个iterableMap，然后方便遍历、删除等

```Java
Map<String, String> iterableMap = new HashMap<>();
iterableMap.put("key1", "value1");
iterableMap.put("key2", "value2");
for (String key : iterableMap.keySet()) {
    System.out.println(key + ": " + iterableMap.get(key));
}
```

---

## List

### GrowthList

LazyList，list自增长效果。GrowthList修饰另一个列表，可以使其在因set或add操作造成索引超出异常时无缝的增加列表长度，可以避免大多数的IndexOutOfBoundsException。

```Java
GrowthList<String> growthList = new GrowthList<>();
growthList.addAll(List.of("A", "B", "C", "D"));
growthList.add("E"); // growthList: [A, B, C, D, E]
```

### TreeList

TreeList实现了优化的快速插入和删除任何索引的列表。这个列表内部实现利用树结构,确保所有的插入和删除都是O(log n)。

```Java
TreeList<String> treeList = new TreeList<>();
treeList.addAll(List.of("D", "B", "A", "C"));
treeList.sort(String::compareTo);  // treeList: [A, B, C, D]
```

### ListUtils: 工具类

#### hashCodeForList

计算List的HashCode

```Java
List<String> list1 = List.of("A", "B", "C");
int hashCode = ListUtils.hashCodeForList(list1);
// hashCode: 94369
```

#### intersection

取交集，生成一个新的List

```Java
List<String> list1 = List.of("A", "B", "C");
List<String> list2 = List.of("B", "C", "D");
List<String> intersection = ListUtils.intersection(list1, list2);
// intersection: [B, C]
```

#### partition

分割操作，将一个大型列表分割成多个小列表，操作非常便捷。
常见场景：例如，当需要批量查询10000个ID时，可以将其分割为若干个200个ID的小组，逐个发起请求进行查询。此外，还可以利用多线程技术，并通过闭锁来实现更高效的处理。

```Java
List<String> list1 = List.of("A", "B", "C");
List<List<String>> partition = ListUtils.partition(list1, 2);
// Partition (size 2): [[A, B], [C]]
```

#### subtract

相当于做减法，用第一个List除去第二个list里含有的元素 ，然后生成一个新的list

```Java
List<String> list1 = List.of("A", "B", "C");
List<String> list2 = List.of("B", "C", "D");
List<String> subtract = ListUtils.subtract(list1, list2);
// Subtract (list1 - list2): [A]
```

#### sum

把两个List的元素相加起来 注意：相同的元素不会加两次 生成一个新的List。

```Java
List<String> list1 = List.of("A", "B", "C");
List<String> list2 = List.of("B", "C", "D");
List<String> sum = ListUtils.sum(list1, list2);
// Sum: [A, B, C, D]
```

#### union

`union` 方法与 `sum` 方法有所不同，它并不具备去重的功能。它内部调用的是 `addAll` 方法，并会生成一个新的 List。

```Java
List<String> list1 = List.of("A", "B", "C");
List<String> list2 = List.of("B", "C", "D");
List<String> union = ListUtils.union(list1, list2);
// Union: [A, B, C, B, C, D]
```

---

## Set

### MultiSet

Set 是无序的，并且不允许出现重复元素。然而在某些场景中，我们不需要保留顺序，但却需要统计特定键出现的次数（例如，每种产品 ID
对应的剩余库存量）。在这种情况下，使用 MultiSet 进行统计是一个很好的选择。

```Java
import org.apache.commons.collections4.MultiSet;
import org.apache.commons.collections4.multiset.HashMultiSet;

MultiSet<String> multiSet = new HashMultiSet<>();
multiSet.add("apple");
multiSet.add("banana");
multiSet.add("apple"); // 添加重复元素
```

### SetUtils: 工具类

#### difference

找出两个集合之间的不同元素，返回的是第一个集合中存在而第二个集合中不存在的元素。

```Java
Set<String> set1 = new HashSet<>(Set.of("A", "B", "C"));
Set<String> set2 = new HashSet<>(Set.of("B", "C", "D"));
Set<String> difference = SetUtils.difference(set1, set2);
// Difference (set1 - set2): [A]
```

#### disjunction

disjunction 是将两个集合中存在于其中一个但不同时存在于两个集合中的元素合并在一起。它与 difference 不同，difference
返回的是第一个集合中存在而第二个集合中不存在的元素。

```Java
Set<String> set1 = new HashSet<>(Set.of("A", "B", "C"));
Set<String> set2 = new HashSet<>(Set.of("B", "C", "D"));
Set<String> disjunction = SetUtils.disjunction(set1, set2);
// Disjunction: [A, D]
```

#### union

使用union合并两个集合，并生成一个新的集合。

```Java
Set<String> set1 = new HashSet<>(Set.of("A", "B", "C"));
Set<String> set2 = new HashSet<>(Set.of("B", "C", "D"));
Set<String> union = SetUtils.union(set1, set2);
// Union: [A, B, C, D]
```

#### emptyIfNull

`emptyIfNull` 函数会在参数为 `null` 时返回一个空的集合。如果参数不为 `null`，则返回参数本身。

```Java
Set<String> emptySet = SetUtils.emptyIfNull(null); // []
```

#### isEqualSet

`isEqualSet` 函数用于判断两个集合中的元素是否完全相同，包括长度和内容。这种判断在某些情况下非常实用。

```Java
Set<String> set1 = new HashSet<>(Set.of("A", "B", "C"));
boolean areEqualSets = SetUtils.isEqualSet(set1, new HashSet<>(Set.of("A", "B", "C"))); // true
```

## Bag

Bag 继承自 Collection 接口，定义了一个集合，该集合会记录对象在集合中出现的次数。

```Java
public interface Bag<E> extends Collection<E> {}
```

常用方法:

* add(E object, int count)：添加指定数量的对象到集合中
* getCount(Object object)：获取指定对象在集合中出现的次数
* uniqueSet()：返回一个Set，包含集合中所有唯一的对象
* size()：返回集合中所有对象的总数

#### HashBag

使用 HashMap 作为数据存储，是一个标准的 Bag 实现。

```Java
import org.apache.commons.collections4.Bag;
import org.apache.commons.collections4.bag.HashBag;

Bag<String> hashBag = new HashBag<>();
hashBag.add("apple", 3); // 添加 3 个 "apple"
hashBag.add("banana", 2); // 添加 2 个 "banana"
hashBag.add("orange", 1); // 添加 1 个 "orange"
// HashBag: [2:banana,1:orange,3:apple]
```

#### TreeBag

用法与`HashBag`类似，只是`TreeBag`会使用自然顺序对元素进行排序。

```Java
import org.apache.commons.collections4.Bag;
import org.apache.commons.collections4.bag.TreeBag;

Bag<String> treeBag = new TreeBag<>();
treeBag.add("banana", 2); // 添加 2 个 "banana"
treeBag.add("apple", 3); // 添加 3 个 "apple"
treeBag.add("orange", 1); // 添加 1 个 "orange"
// TreeBag (sorted): [3:apple,2:banana,1:orange]
```


<br/><br/><br/><br/>