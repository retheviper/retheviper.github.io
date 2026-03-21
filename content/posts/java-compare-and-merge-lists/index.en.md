---
title: "Combine two Lists"
date: 2020-07-26
categories: 
  - java
image: "../../images/java.webp"
tags:
  - stream
  - java
---

I saw a question on a site I often visit. The question was how to merge two lists into one without duplicate elements. This may be easy to solve with SQL, but if you cannot change the query, or if you need to combine results from multiple APIs, you have no choice but to handle the merge in code. It was not a situation I had faced before, but I found it interesting, so I tried several approaches and wrote some sample code.

## the code in question

What the author of the question wants to do is that there are two `List<Map<String, Object>>`s, check the Map elements in the List, and if there are any duplicates, combine them into one List. Here, the duplication conditions were the Map's Key and Value.

Although the code currently exists, the logic for checking for duplicates has become too complex, resulting in high load and performance problems. The code he regretted was something like this:

```java
List<Object> list1 = ...
List<Object> list2 = ...

for (Object object : list1) {
    Map<String, String> map = (HashMap<String, String>) object;
    String a1 = map.get("a");
    for (Object object2 : list2) {
        Map<String, String> map2 = (HashMap<String, String>) object2;
        String a2 = map2.get("a");
        if (a1.equals(a2)) {
            Set<String> keys = map2.keySet();
            for (String key : keys) {
                String v = map2.get(key);    
                map.put(key, v);
            }
        }
    }
}
```

Looking at the code, it is a triple loop, checking each item one by one to see if there is a matching key and value. I'm not sure if this is the actual code, but since the keys are literals in this code, I think another loop should be added to cycle through all the keys. This will likely add another loop, making the code even more complex. I think if you were to run this for each List, it would be a lot of work. So I would like to make this code as short and simple as possible.

It may not be the best practice, but I will introduce the code based on my thoughts.

## Try using Stream

If I want to process the elements of a List one by one, I first thought about whether there was a way to use a Stream. When I searched on the internet, I found that there are several ways to merge multiple Lists using Stream. I tested it using them.

## Join and exclude duplicates

Using `Stream.concat()`, you can connect two Streams. Stream also allows you to remove duplicates with `distinct()`. Using these combinations, you can combine two Lists without duplicate elements. First, using a simple example, the code would be as follows.

```java
// Lists to merge
List<String> list1 = List.of("a", "b", "d", "e");
List<String> list2 = List.of("b", "c", "d", "f");

// Merge and remove duplicates ("a", "b", "c", "d", "e", "f")
List<String> concat = Stream.concat(list1.stream(), list2.stream()).distinct().collect(Collectors.toList());
```

Only two arguments can be specified for `Stream.concat()`, so if you want to connect three or more Lists, you should consider using a loop. Also, in the case of `distinct()`, any object can be checked for duplicates as long as `equals()` is properly defined. Therefore, even classes with annotations such as Lombok's `@Data` can be stored in one List by excluding duplicates.

## limit

The author of the question wants to check for duplicates on Map elements in a List, but this method does not do that. This is because the map itself is compared with `equals()`, and each element inside is not checked. So, with code like this, you'll just end up with something that looks like two Lists connected.

## Try using a For loop

This time, I will modify the questioner's code to make it more efficient. The purpose of using a For loop is to execute `put()` only once if the conditions are met. `Stream` and `forEach()` are used to process all elements, so they have been removed.

Map has `putAll()` in addition to `put()`, so you can erase the loop one by one while going through the elements. After running `putAll()`, there is no need to check the next element, so run `continue` and skip the next loop to eliminate unnecessary processing. Then, you can return the code as follows.

```java
for (Object object : list1) {
    Map<String, String> map = (HashMap<String, String>) object;
    String a1 = map.get("a");
    for (Object object2 : list2) {
        Map<String, String> map2 = (HashMap<String, String>) object2;
        String a2 = map2.get("a");
        if (a1.equals(a2)) {
            // Remove one loop
            map.putAll(map2);
            continue;
        }
    }
}
```

Here, we will also fix the part where the Map key is specified. The number of loops will increase by one because we will compare Entries while cycling through them instead of specifying them with literals. Since Entry can be obtained with `Set`, it can be compared using `contains()`, a method of Collection. Therefore, all you have to do is cycle through the Entries of the Maps you want to compare and check whether the element is in a different Map. Below is the code that has been modified to reflect this.

```java
for (Object object : list1) {
    Map<String, String> map = (HashMap<String, String>) object;
    for (Object object2 : list2) {
        Map<String, String> map2 = (HashMap<String, String>) object2;
        // Compare using Entry
        for (Map.Entry<String, DataClass> entry : map2.entrySet()) {
            if (map1.entrySet().contains(entry)) {
                map1.putAll(map02);
                continue;
            }
        }
    }
}
```

The other thing to do is to convert the type of List from the beginning so that you don't have to convert the type in each loop. I don't think you can expect much performance improvement from this...

```java
// Convert the type in advance
List<Map<String, String>> convertedList1 = (List<Map<String, String>>) list1;
List<Map<String, String>> convertedList2 = (List<Map<String, String>>) list2;

for (Map<String, String> map : convertedList1) {
    for (Map<String, String> map2 : convertedList2) {
        for (Map.Entry<String, DataClass> entry : map2.entrySet()) {
            if (map1.entrySet().contains(entry)) {
                map1.putAll(map02);
                continue;
            }
        }
    }
}
```

Depending on the case, it may be necessary to perform processing such as sorting the keys of the Map, but I feel that the requirements have been met for now.

## If the conditions are different

I can only infer from the questioner's code, but if there is a condition to compare based on the index of List, the code can be further reduced. This means checking if there is a Map with the same element at the same index in list1 and list2. If this condition exists, the loop can be folded twice. Below is the code for that case.

```java
for (Map<String, String> map : list1) {
    for (Map.Entry<String, String> entry : map.entrySet()) {
        // Compare the element at the same index in List2
        Map<String, String> map2 = list2.get(list1.indexOf(map));
        if (map2.entrySet().contains(entry)) {
            map.putAll(map2);
            continue;
        }
    }
}
```

However, in order to use this method, it is necessary to assume that the two Lists have the same size, so you have to be careful about that.

## lastly

I think it's a good experience to pay attention to the community, as it gives you a chance to think about cases you haven't encountered yet. While you're doing your research, it's also a good opportunity to look up methods and APIs you've never used before, and to look back at the code you've been writing.

You can think of various other variations, such as checking only if there are duplicate values ​​or checking if the fields of an element are duplicates, regardless of the key. All of these topics are interesting, but since they are a little different from the current topic, I would like to post some code for such cases on my blog someday if I have a chance. See you soon!
