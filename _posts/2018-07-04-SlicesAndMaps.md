---
title:  Slices and Maps in Go
author: Will Andrews
date: 2018-07-04
order: 19
---

This post will go through the use of Slices and Maps in Go. (Is it me that always thinks of pizza when I hear slice!)

### What's a slice?
In Go, a slice is a better array. It uses an array in the background to store the values, and the slice that you use in code, contains references to the array segments. This makes it lightweight and easier to manipulate.

An array has a fixed length, where as a slice can be increased and decreased in size if needed. 

It's also worth noting, that as slices reference an underlying array, if you have 2 slices that reference the same array, changing the value of 1 segment in the slice, or underlying array, will change the value in the other slice. More on this later.

### Creating a slice

There are 2 ways to create a slice.

1: Use the 'make' function and spec out the slice, by saying what type of data will be held in the slice, the length of the slice and the capacity of the underlying array.

This slice will contain string values, have a length of 2, but the array storing the data will be have a capacity of 5, allowing the slice to grow. We then add in the values we want to the segments. However, if we try and add in a 3rd, we get an error. To add a new one we will need to append, which will be covered later.
``` go
shoppingList := make([]string, 2, 5)
shoppingList[0] = "Milk"
shoppingList[1] = "Bread
```

2: Directly create a slice and let Go work out the capacity etc.
```go
shoppingList := []string {"Milk", "Bread"}
```

### Creating a slice of a slice
Let's say we created our shopping list, but we want to make a copy of it. We can create a copy of it's references. We can either copy all of it, or just part of it. 
```go
shoppingList1 := []string {"Milk", "Bread", "Coffee", "Sugar"}

shoppingList2 := shoppingList1[0:] // Start at the first value (inclusive) and copy everything after

shoppingList3 := shoppingList1[1:3] // Start at the second value (inclusive) and copy up until the 4th value (non inclusive)

fmt.Println(shoppingList1)
fmt.Println(shoppingList2)
fmt.Println(shoppingList3)
```

This will output the following:
```
[Milk Bread Coffee Sugar]
[Milk Bread Coffee Sugar]
[Bread Coffee]
```
Remember, to use zero based when referencing a location in a slice.

### Change the value of a slice reference
Let's say I want to change bread and instead get some biscuits instead. Nice and easy.
```go
shoppingList[1] = "Biscuits"
```

However, as mentioned earlier, doing this will change the value for any slices sharing the same array. Here we will create a slice, then create a copy of that slice. Then we will change the value of one slices reference and see that it changes in the second slice as well.
```go
shoppingList1 := []string{"Milk", "Bread", "Coffee", "Sugar"}

shoppingList2 := shoppingList1[0:]

shoppingList1[1] = "Biscuits"

fmt.Println(shoppingList1)
fmt.Println(shoppingList2)
```

This will output the following:
```
[Milk Biscuits Coffee Sugar]
[Milk Biscuits Coffee Sugar]
```
Note how bread has changed to biscuits in both slices.

### Appending to a slice
Let's say we wanted to add some beer to our list. To do this we append the slice.
```go
shoppingList1 := []string{"Milk", "Bread", "Coffee", "Sugar"}

shoppingList1 = append(shoppingList1, "beer")

fmt.Println(shoppingList1)
```
This will output the following:
```
[Milk Bread Coffee Sugar beer]
```

Hang on, we created a slice with 4 segments, which Go would have created an underlying array with 4 segments. How do we add another segment in? Won't that cause an overflow exception? No. When doing this, if Go thinks this is going to happen, it will create a new array for the slice to use, doubling the capacity and then change the references over. 

BUT!!! Be careful, because if you created a slice off of this slice, once the first slice has been increased in size, the second slice will no longer be referencing the same underlying array. So changing the value in slice 1, will no longer effect slice 2.
```go
shoppingList1 := []string{"Milk", "Bread", "Coffee", "Sugar"}
shoppingList2 := shoppingList1[0:]

fmt.Println("\nshoppingList1Capacity is: ", cap(shoppingList1))
fmt.Println("\nshoppingList2Capacity is: ", cap(shoppingList2))

shoppingList1 = append(shoppingList1, "beer")

fmt.Println("\nshoppingList1Capacity is: ", cap(shoppingList1))
fmt.Println("\nshoppingList2Capacity is: ", cap(shoppingList2))

shoppingList1[1] = "Biscuits"

fmt.Println(shoppingList1)
fmt.Println(shoppingList2)
```
This will output the following:
```
shoppingList1Capacity is:  4

shoppingList2Capacity is:  4

shoppingList1Capacity is:  8

shoppingList2Capacity is:  4
[Milk Biscuits Coffee Sugar beer]
[Milk Bread Coffee Sugar]
```
Note how the second shopping list contains different values after the first shopping list is expanded and edited.

### So what's a map?
Maps are a the equivalent of a dictionary in .Net

### Creating a map
They are created in a similar way to slices.

1: Use the make function to create a new map that contains no keys or values and then add some keys and values after.

This map will have a key of type string and a value of type int. We will then add in a few keys with values.
```go
testMap := make(map(string)int)
testMap["a"] = 1
testMap["b"] = 2
testMap["c"] = 3
```

2: Directly create the map and initialize it with values. 

This map will also have a key of type string and a value of type int, but it will have some keys and values assigned straight away.

```go
testMap := map[string]int{
    "a":1,
    "b":2,
    "c":3
}
```

### Getting a value from a map

To get a value, you just use the key to get a value:
```go
testMap := map[string]int{
    "a":1,
    "b":2,
    "c":3
}
fmt.Println(testMap["b"])
```
This will give the result:
```
2
```

### Edit a map value
To edit a value in a map, use the key and then set the value: 
```go
testMap := map[string]int{
    "a":1,
    "b":2,
    "c":3
}
fmt.Println(testMap["b"])
testMap["b"] = 99
fmt.Println(testMap["b"])
```
This will output the following:
```
2
99
```
If you try and edit a value of a key that doesn't exist, it will simply add it.

### Adding a value
To do this, you use the same method as editing a value, the only difference is that you are using a key that doesn't already exist.
```go
testMap := map[string]int{
    "a":1,
    "b":2,
    "c":3
}
testMap["d"] = 99
fmt.Println(testMap["d"])
```
This will output the following:
```
99
```

### Deleting a value from a map
To do this, there is a delete function:
```go
testMap := map[string]int{
    "a":1,
    "b":2,
    "c":3
}
fmt.Println(testMap)
delete(testMap, "c")
fmt.Println(testMap)
```
This will output the following:
```
map[a:1 b:2 c:3]
map[a:1 b:2]
```