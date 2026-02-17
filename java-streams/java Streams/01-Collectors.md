<p><a target="_blank" href="https://app.eraser.io/workspace/0ESPpfMVk8fSMviWl2aU" id="edit-in-eraser-github-link"><img alt="Edit in Eraser" src="https://firebasestorage.googleapis.com/v0/b/second-petal-295822.appspot.com/o/images%2Fgithub%2FOpen%20in%20Eraser.svg?alt=media&amp;token=968381c8-a7e7-472a-8ed6-4a6626da5501"></a></p>

# Collectors
---

# 🔹 1. **Basic Collection**
| Collector | Description | Example |
| ----- | ----- | ----- |
|  | Collect elements into a  |  |
|  | <p>Collect elements into a </p><p> (removes duplicates)</p> |  |
|  | <p>Collect into a specific collection (e.g., </p><p>)</p> |  |
---

# 🔹 2. **Counting / Summarizing**
| Collector | Description | Example |
| ----- | ----- | ----- |
|  | Count number of elements |  |
|  | Sum of int values |  |
|  | Sum of long values |  |
|  | Sum of double values |  |
|  | Average of int values |  |
|  | Average of long values |  |
|  | Average of double values |  |
|  | <p>Returns </p><p> (count, sum, min, max, avg)</p> |  |
|  | Returns  |  |
|  | Returns  |  |
---

# 🔹 3. **Joining Strings**
| Collector | Description | Example |
| ----- | ----- | ----- |
|  | Concatenate strings with no delimiter |  |
|  | Concatenate with delimiter |  |
|  | Concatenate with delimiter + prefix/suffix |  |
---

# 🔹 4. **Grouping & Partitioning**
| Collector | Description | Example |
| ----- | ----- | ----- |
|  | Group elements by a classifier function (key → list of values) |  |
|  | Group and apply downstream collector |  |
|  | Group with a specific map type |  |
|  | <p>Partition into </p><p> / </p><p> map</p> |  |
|  | Partition with downstream collector |  |
---

# 🔹 5. **Reducing / Mapping**
| Collector | Description | Example |
| ----- | ----- | ----- |
|  | Reduce elements with a binary operator |  |
|  | Reduce with identity value |  |
|  | Map then reduce |  |
|  | Apply a function then collect downstream |  |
---

# 🔹 6. **Min / Max**
| Collector | Description | Example |
| ----- | ----- | ----- |
|  | Find min element using comparator |  |
|  | Find max element using comparator |  |
---

# 🔹 7. **Miscellaneous**
| Collector | Description | Example |
| ----- | ----- | ----- |
|  | Collect into a  |  |
|  | Handle duplicate keys |  |
|  | Specify map type |  |
|  | Apply a finisher function after collection |  |
---

# ✅ Summary
- **Collection** → `toList()` , `toSet()` , `toCollection()` 
- **Counting / Summarizing** → `counting()` , `summingX()` , `averagingX()` , `summarizingX()` 
- **Joining** → `joining()` 
- **Grouping / Partitioning** → `groupingBy()` , `partitioningBy()` 
- **Reducing / Mapping** → `reducing()` , `mapping()` 
- **Min / Max** → `minBy()` , `maxBy()` 
- **To Map / Post-processing** → `toMap()` , `collectingAndThen()` 
---

If you want, I can create a **one-page cheat sheet table with all Collectors + examples + return types** that you can **keep handy for interviews**.

Do you want me to do that?



<!--- Eraser file: https://app.eraser.io/workspace/0ESPpfMVk8fSMviWl2aU --->