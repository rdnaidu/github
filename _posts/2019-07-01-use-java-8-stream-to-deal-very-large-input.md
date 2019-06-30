---
layout: single
classes: wide
title:  "Use Java 8 streams for handling very large data"
date:   2019-06-03 23:40 -0500
tags:
  - Java
  - Java8
  - data
  - IO
 
categories: Java tricks
excerpt: "How would you deal with infinite of very large data in Java 8+ without breaking the CPU? Let's see the way I would do it."
comments: true
---

## Understanding the scenario

 Not so long, I saw a problem that says something like

 > Based on a very large list of `Integer` values as input, try to find all missing `Integer` values having in count that you have only left 2 Gbs of RAM.

Firstly, it intrigued me the fact that I had to have in count 2 Gbs of RAM, but probably that was just to bear in mind the size of Integer as a data structure and how many of them could I create before depleting the memory (Memory Overflow). To calculate the maximum amount of Integers we should be able to create I used the following constant:

```
private static final int MAX_CAPACITY = 2 * 1024 * 1024 * 1024 / Integer.BYTES;
```

- `Integer.BYTES` returns the amount of memory in bytes each `Integer` variable will occupy, which depends on the [Java Platform][1]. In Java 8 it is 32 bits, which is 4 bytes.
- 1024 bytes is 1 Kb, multiplied by 1024 again is the amount of Mb, which multiplied by 1024 is the amount of Gbs.
- I multiplied that number by 2 because to represent the maximum of 2 Gb to occupy. Afterwards I divided it by `Integer.BYTES` to determinate the amount of `Integer` values I can affore.

In order to find the missing `Integer` values that are not in the input list I had to implement a function called `findMissingIntegers` so I can print them later

```
final List<Integer> result = findMissingIntegers(Arrays.asList(
        2, 4, 6, 1, 14
)); 

result.forEach(System.out::println);
```

At the end I expect that the variable `result` contains a list of all those `Integer` I want to work with; which in the previous example it was just print to the console.

## Calculate a very large input and output using a List in Java
By instinct the first that will come to most developer's mind is to obtain a final definitive response, probably boxed into a List.
My version of that solution was something like

```
    static List findMissingIntegers(List<Integer> inputIntegers) {
        List result = new ArrayList<Integer>(MAX_CAPACITY);                                     #1

        Collections.sort(inputIntegers);                                                        #2    

        Integer previous = Integer.MIN_VALUE;
        Integer foundInts = 0;
        for (Integer i : inputIntegers) {
            if (i - previous > 1) {                                                             #3
                addNumbersRangeToList(result, previous + 1, i - 1);

                foundInts += i - previous;
                if (foundInts >= MAX_CAPACITY) {
                    System.out.printf("Maximum arrived %d of Integers%n", foundInts);
                    return result;
                }
            }
            previous=i;
        }

        final int MAX_NUMBERS_LEFT_TO_ADD = MAX_CAPACITY - foundInts;

        addNumbersRangeToList(result, previous +1, Math.max(previous + MAX_NUMBERS_LEFT_TO_ADD, Integer.MAX_VALUE));

        return result;
    }

    private static void addNumbersRangeToList(List<Integer> list, Integer start, Integer end) {
        if (start == end) {
            list.add(start);
        } else {
            IntStream.rangeClosed(start, end).forEach(list::add);
        }
    }
```

1. This is a big amount of RAM and it needs to be allocated.
2. I used the the java.util.Arrays.sort [which guarantees since Java 7 a O(n log n) performance][2], which which is much better than iterating over n elements and search if that element is contained in the input array, which will be unordered and is prone to actually have a BigO(n) complexity to find if an element is contained.
3. As the input is ordered, we can skip the stragegy of searching if an Integer `i` of the closed range `[Integer.MIN_VALUE, Integer.MAX_VALUE]`, is contained in the input array. Instead we can see add chunks of data based on the difference of the previous value gotten from the input to the current one. For instance, if the first value of the input is -2, then we can add easly from `Integer.MAX_MIN` to `-1` to the list of missing Integers, which saves a big amount of calculation for sure.




## Read more
[Primitives Data Types][1] in the Official website

[1]: https://docs.oracle.com/javase/tutorial/java/nutsandbolts/datatypes.html
[2]: https://bugs.openjdk.java.net/browse/JDK-6804124