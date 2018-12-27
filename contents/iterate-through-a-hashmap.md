# HashMap遍历

在 Java 中有多种遍历 HashMap 的方法。让我们回顾一下最常见的方法和它们各自的优缺点。由于所有的 Map 都实现了 Map 接口，所以接下来方法适用于所有 Map（如：HaspMap, TreeMap, LinkedMap, HashTable, etc）

## 方法#1 使用 For-Each 迭代 entries

这是最常见的方法，并在大多数情况下更可取的。当你在循环中需要使用Map的键和值时，就可以使用这个方法

```java
Map<Integer, Integer> map = new HashMap<Integer, Integer>();
for(Map.Entry<Integer, Integer> entry : map.entrySet()){
	System.out.println("key = " + entry.getKey() + ", value = " + entry.getValue())
}
```

注意：For-Each 循环是 Java5 新引入的，所以只能在 Java5 以上的版本中使用。如果你遍历的 map 是 null 的话，For-Each 循环会抛出 NullPointerException 异常，所以在遍历之前你应该判断是否为空引用。

## 方法#2 使用 For-Each 迭代 keys 和 values

如果你只需要用到 map 的 keys 或 values 时，你可以遍历 KeySet 或者 values 代替 entrySet

```java
Map<Integer, Integer> map = new HashMap<Integer, Integer>();

//iterating over keys only
for (Integer key : map.keySet()) {
	System.out.println("Key = " + key);
}

//iterating over values only
for (Integer value : map.values()) {
	System.out.println("Value = " + value);
}
```

这个方法比entrySet迭代具有轻微的性能优势(大约快10%)并且代码更简洁

## 方法#3 使用 Iterator 迭代

使用泛型

```java
Map<Integer, Integer> map = new HashMap<Integer, Integer>();
Iterator<Map.Entry<Integer, Integer>> entries = map.entrySet().iterator();
while (entries.hasNext()) {
	Map.Entry<Integer, Integer> entry = entries.next();
	System.out.println("Key = " + entry.getKey() + ", Value = " + entry.getValue());
}
```

不使用泛型

```java
Map map = new HashMap();
Iterator entries = map.entrySet().iterator();
while (entries.hasNext()) {
	Map.Entry entry = (Map.Entry) entries.next();
	Integer key = (Integer)entry.getKey();
	Integer value = (Integer)entry.getValue();
	System.out.println("Key = " + key + ", Value = " + value);
}
```

你可以使用同样的技术迭代 keyset 或者 values

这个似乎有点多余但它具有自己的优势。首先，它是遍历老 java 版本 map 的唯一方法。另外一个重要的特性是可以让你在迭代的时候从 map 中删除 entries 的(通过调用 iterator.remover() )唯一方法.如果你试图在 For-Each 迭代的时候删除 entries，你将会得到 unpredictable resultes 异常。

从性能方法看，这个方法等价于使用 For-Each 迭代

## 方法#4 迭代 keys 并搜索 values（低效的）

```java
Map<Integer, Integer> map = new HashMap<Integer, Integer>();
for (Integer key : map.keySet()) {
	Integer value = map.get(key);
	System.out.println("Key = " + key + ", Value = " + value);
}
```

这个方法看上去比方法#1更简洁，但是实际上它更慢更低效，通过 key 得到 value 值更耗时（这个方法在所有实现 map 接口的 map 中比方法#1慢20%-200%）。如果你安装了 FindBugs，它将检测并警告你这是一个低效的迭代。这个方法应该避免。

## 总结

如果你只需要使用 key 或者 value 使用方法#2，如果你坚持使用 java 的老版本（java 5 以前的版本）或者打算在迭代的时候移除 entries，使用方法#3。其他情况请使用#1方法。避免使用#4方法。

原文地址：[http://stackoverflow.com/questions/1066589/iterate-through-a-hashmap](http://stackoverflow.com/questions/1066589/iterate-through-a-hashmap)