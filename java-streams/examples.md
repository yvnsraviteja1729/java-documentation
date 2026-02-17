<p><a target="_blank" href="https://app.eraser.io/workspace/gSnLOIzpgNKHksvgulY9" id="edit-in-eraser-github-link"><img alt="Edit in Eraser" src="https://firebasestorage.googleapis.com/v0/b/second-petal-295822.appspot.com/o/images%2Fgithub%2FOpen%20in%20Eraser.svg?alt=media&amp;token=968381c8-a7e7-472a-8ed6-4a6626da5501"></a></p>

# java Streams
[﻿Java 8 Stream Tutorial - GeeksforGeeks](https://www.geeksforgeeks.org/java/java-8-stream-tutorial/) 

[﻿information](java%20Streams/information%2027226bcbdfe880d38c16e3b31af60add.md) 

[﻿Comparator functions](java%20Streams/Comparator%20functions%2027226bcbdfe880a48f63d220863bb0c7.md) 

[﻿Collectors](java%20Streams/Collectors%2027226bcbdfe880efb268fef4ab5d2b99.md) 

- Generate stream using stream.iterateStream.iterate(1, n -> n <= 100, n -> n * 2)
.forEach(System.out::println);
- Given a string `"hello world"` , create a stream of characters and print each.import java.util.Arrays;

class Main {
    public static void main(String[] args) {
        String str = "hello world";

        Arrays.stream(str.split(""))    // Stream<String>
              .forEach(System.out::println);
    }
}
- Convert an array of primitive `int[]`  to a stream and sum all numbers.import java.util.Arrays;

class Main {
    public static void main(String[] args) {
        int[] numbers = {1, 2, 3, 4, 5};

        int sum = Arrays.stream(numbers)   // creates IntStream
                        .sum();            // directly sums all elements

        System.out.println("Sum = " + sum);
    }
}
- Convert list of strings to uppercase// Online Java Compiler
// Use this editor to write, compile and run your Java code online
import java.util.*;
import java.util.stream.Collectors;

class Main {
    public static void main(String[] args) {
        System.out.println("Try programiz.pro");
        List<String> list=List.of("hello","world","new");
        List<String> ans=list.stream().map(str->str.toUpperCase())
            .peek(str->System.out.println(str))
            .collect(Collectors.toList());
        
    }
}
- Reverse elements in an arrayimport java.util.*;
import java.util.stream.*;

public class ReverseArray {
    public static void main(String[] args) {
        Integer[] arr = {1, 2, 3, 4, 5};

        // Reverse using IntStream
        Integer[] reversed = IntStream.rangeClosed(1, arr.length)
                                      .mapToObj(i -> arr[arr.length - i])
                                      .toArray(Integer[]::new);

        System.out.println(Arrays.toString(reversed)); // [5, 4, 3, 2, 1]
    }
}
- Reverse elements in a stringString reversed = str.chars()                        // IntStream of code points
.mapToObj(c -> (char)c)                          // convert int -> Character
.reduce("",                                      // identity (start with empty string)
    (s, c) -> c + s,                             // accumulator: prepend character
    (s1, s2) -> s2 + s1);                        // combiner (for parallel streams)
- Sorting the stringsimport java.util.*;
import java.util.stream.Collectors;

class Main {
    public static void main(String[] args) {
        List<String> list = List.of("hello", "world", "new", "java", "streams");

        List<String> ans = list.stream()
            .sorted((a, b) -> Integer.compare(a.length(), b.length())) // custom comparator
            .map(String::toUpperCase)
            .peek(System.out::println)
            .collect(Collectors.toList());

        System.out.println("Final list: " + ans);
    }
}
- Find the second highest number in a list using `sorted()`  and `skip()` .import java.util.*;
import java.util.stream.*;

class Main {
    public static void main(String[] args) {
        List<Integer> numbers = Arrays.asList(10, 20, 35, 50, 50, 40, 25);

        Integer secondHighest = numbers.stream()
                .distinct() // remove duplicates
                .sorted(Comparator.reverseOrder()) // sort descending
                .skip(1) // skip the highest
                .findFirst() // pick the next one
                .orElse(null); // return null if not found

        System.out.println("Second Highest: " + secondHighest);
    }
}
- Find the maximum and minimum numbers in a list using `reduce()` .

 `java ## 1️⃣ Find Maximum ` java
 import java.util._;_
_ import java.util.stream._;

 class Main {
 public static void main(String[] args) {
 List numbers = Arrays.asList(5, 10, 15, 20, 25, 3, 8);

 // maximum using reduce
 int max = numbers.stream()
 .reduce(Integer::max) // or (a, b) -> a > b ? a : b
 .orElseThrow(); // handle empty list

 System.out.println("Maximum = " + max);
 }
 }
 ` **Output:** ` 
 Maximum = 25
 ``` // minimum using reduce
int min = numbers.stream()
                 .reduce(Integer::min)          // or (a, b) -> a < b ? a : b
                 .orElseThrow();

System.out.println("Minimum = " + min);2️⃣ Find Minimum **Output:**

 ` Minimum = 3 `  
- Check if all numbers in a list are positive using `allMatch()` .import java.util.*;
import java.util.stream.*;

class Main {
    public static void main(String[] args) {
        List<Integer> numbers = Arrays.asList(5, 10, 15, 20, 25);

        boolean allPositive = numbers.stream()
                                     .allMatch(n -> n > 0); // condition for positivity

        System.out.println("All numbers are positive? " + allPositive);
    }
}
- Find the average of numbers using `mapToInt().average()` .import java.util.*;
import java.util.stream.*;

class Main {
    public static void main(String[] args) {
        List<Integer> numbers = Arrays.asList(5, 10, 15, 20, 25);

        OptionalDouble average = numbers.stream()
                                        .mapToInt(Integer::intValue) // convert to IntStream
                                        .average();                  // compute average

        if (average.isPresent()) {
            System.out.println("Average = " + average.getAsDouble());
        } else {
            System.out.println("List is empty!");
        }
    }
}
- Map<Character, Long> countByFirstChar = names.stream()
        .collect(Collectors.groupingBy(name -> name.charAt(0), Collectors.counting()));
System.out.println(countByFirstChar);🔹 Notes:{A=[Alice, Adam], B=[Bob, Brian], C=[Charlie, Catherine]}✅ Sample Output:Group a list of names by their first character using `Collectors.groupingBy()` .

 `java import java.util.*; import java.util.stream.Collectors; class Main { public static void main(String[] args) { List<String> names = Arrays.asList("Alice", "Adam", "Bob", "Brian", "Charlie", "Catherine"); Map<Character, List<String>> groupedNames = names.stream() .collect(Collectors.groupingBy(name -> name.charAt(0))); // group by first character System.out.println(groupedNames); } } `   **Output:**

 ` {A=2, B=2, C=2} `  
    - `groupingBy()`  takes a **classifier function** (`name -> name.charAt(0)` ) which decides the key for grouping.
    - The result is a **Map<Key, List>** by default.
    - You can also do **downstream operations** like counting, summing, or mapping:

- Given a list of strings, find the frequency of each word using Java streams:List<String> words = Arrays.asList("apple", "banana", "apple", "cherry", 
                                    "banana", "apple");
Map<String, Long> wordFrequency = words
              .stream()
              .collect(Collectors
                    .groupingBy(Function.identity(), Collectors.counting())
                );
[﻿Java Stream Coding Interview Questions: Part 1](https://medium.com/@mehar.chand.cloud/java-stream-coding-interview-questions-part-1-dc39e3575727) 



<!--- Eraser file: https://app.eraser.io/workspace/gSnLOIzpgNKHksvgulY9 --->