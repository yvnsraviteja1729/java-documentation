# Collectors

---

# 🔹 1. **Basic Collection**

| Collector | Description | Example |
| --- | --- | --- |
| `toList()` | Collect elements into a `List` | `stream.collect(Collectors.toList())` |
| `toSet()` | Collect elements into a `Set` (removes duplicates) | `stream.collect(Collectors.toSet())` |
| `toCollection(Supplier)` | Collect into a specific collection (e.g., `TreeSet`) | `stream.collect(Collectors.toCollection(TreeSet::new))` |

---

# 🔹 2. **Counting / Summarizing**

| Collector | Description | Example |
| --- | --- | --- |
| `counting()` | Count number of elements | `stream.collect(Collectors.counting())` |
| `summingInt(ToIntFunction)` | Sum of int values | `stream.collect(Collectors.summingInt(String::length))` |
| `summingLong(ToLongFunction)` | Sum of long values | `stream.collect(Collectors.summingLong(Employee::getSalary))` |
| `summingDouble(ToDoubleFunction)` | Sum of double values | `stream.collect(Collectors.summingDouble(Product::getPrice))` |
| `averagingInt(ToIntFunction)` | Average of int values | `stream.collect(Collectors.averagingInt(String::length))` |
| `averagingLong(ToLongFunction)` | Average of long values | `stream.collect(Collectors.averagingLong(Employee::getSalary))` |
| `averagingDouble(ToDoubleFunction)` | Average of double values | `stream.collect(Collectors.averagingDouble(Product::getPrice))` |
| `summarizingInt(ToIntFunction)` | Returns `IntSummaryStatistics` (count, sum, min, max, avg) | `stream.collect(Collectors.summarizingInt(String::length))` |
| `summarizingLong(ToLongFunction)` | Returns `LongSummaryStatistics` | `stream.collect(Collectors.summarizingLong(Employee::getSalary))` |
| `summarizingDouble(ToDoubleFunction)` | Returns `DoubleSummaryStatistics` | `stream.collect(Collectors.summarizingDouble(Product::getPrice))` |

---

# 🔹 3. **Joining Strings**

| Collector | Description | Example |
| --- | --- | --- |
| `joining()` | Concatenate strings with no delimiter | `stream.collect(Collectors.joining())` |
| `joining(CharSequence delimiter)` | Concatenate with delimiter | `stream.collect(Collectors.joining(","))` |
| `joining(CharSequence delimiter, CharSequence prefix, CharSequence suffix)` | Concatenate with delimiter + prefix/suffix | `stream.collect(Collectors.joining(", ", "[", "]"))` |

---

# 🔹 4. **Grouping & Partitioning**

| Collector | Description | Example |
| --- | --- | --- |
| `groupingBy(Function)` | Group elements by a classifier function (key → list of values) | `stream.collect(Collectors.groupingBy(String::length))` |
| `groupingBy(Function, Collector)` | Group and apply downstream collector | `stream.collect(Collectors.groupingBy(String::length, Collectors.counting()))` |
| `groupingBy(Function, Supplier, Collector)` | Group with a specific map type | `stream.collect(Collectors.groupingBy(String::length, TreeMap::new, Collectors.toList()))` |
| `partitioningBy(Predicate)` | Partition into `true` / `false` map | `stream.collect(Collectors.partitioningBy(s -> s.length() > 5))` |
| `partitioningBy(Predicate, Collector)` | Partition with downstream collector | `stream.collect(Collectors.partitioningBy(s -> s.length() > 5, Collectors.counting()))` |

---

# 🔹 5. **Reducing / Mapping**

| Collector | Description | Example |
| --- | --- | --- |
| `reducing(BinaryOperator)` | Reduce elements with a binary operator | `stream.collect(Collectors.reducing(Integer::sum))` |
| `reducing(identity, BinaryOperator)` | Reduce with identity value | `stream.collect(Collectors.reducing(0, Integer::sum))` |
| `reducing(identity, Function, BinaryOperator)` | Map then reduce | `stream.collect(Collectors.reducing(0, String::length, Integer::sum))` |
| `mapping(Function, Collector)` | Apply a function then collect downstream | `stream.collect(Collectors.mapping(String::toUpperCase, Collectors.toList()))` |

---

# 🔹 6. **Min / Max**

| Collector | Description | Example |
| --- | --- | --- |
| `minBy(Comparator)` | Find min element using comparator | `stream.collect(Collectors.minBy(String::compareTo))` |
| `maxBy(Comparator)` | Find max element using comparator | `stream.collect(Collectors.maxBy(String::compareTo))` |

---

# 🔹 7. **Miscellaneous**

| Collector | Description | Example |
| --- | --- | --- |
| `toMap(keyMapper, valueMapper)` | Collect into a `Map` | `stream.collect(Collectors.toMap(String::toUpperCase, String::length))` |
| `toMap(keyMapper, valueMapper, mergeFunction)` | Handle duplicate keys | `stream.collect(Collectors.toMap(String::length, s -> s, (s1, s2) -> s1 + "," + s2))` |
| `toMap(keyMapper, valueMapper, mergeFunction, mapSupplier)` | Specify map type | `stream.collect(Collectors.toMap(String::length, s -> s, (s1,s2)->s1+","+s2, TreeMap::new))` |
| `collectingAndThen(Collector, finisher)` | Apply a finisher function after collection | `stream.collect(Collectors.collectingAndThen(Collectors.toList(), list -> Collections.unmodifiableList(list)))` |

---

# ✅ Summary

- **Collection** → `toList()`, `toSet()`, `toCollection()`
- **Counting / Summarizing** → `counting()`, `summingX()`, `averagingX()`, `summarizingX()`
- **Joining** → `joining()`
- **Grouping / Partitioning** → `groupingBy()`, `partitioningBy()`
- **Reducing / Mapping** → `reducing()`, `mapping()`
- **Min / Max** → `minBy()`, `maxBy()`
- **To Map / Post-processing** → `toMap()`, `collectingAndThen()`

---

If you want, I can create a **one-page cheat sheet table with all Collectors + examples + return types** that you can **keep handy for interviews**.

Do you want me to do that?