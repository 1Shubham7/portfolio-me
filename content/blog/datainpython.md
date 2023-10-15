---
title: "Datatypes in Python"
description: "Datatypes in Python in simplest yet most effective way. · Python includes 6 types of built-in datatypes - Numeric datatype Boolean datatype Sequence..."
dateString: October 2023
draft: false
tags: ["CNCF", "DevOps", "Open-source", "CI/CD"]
weight: 101
cover:
    image: "blog/python.png"
---

Python includes 6 types of built-in datatypes -

- Numeric datatype

- Boolean datatype

- Sequence datatype

- Sets

- Dictionaries

- None

## Numeric datatype

this datatype includes integer, float, and complex numbers. Unlike other languages, python allows you to reassign float value to a variable with integer or vice versa.

```
a = 10

a = 10.5

print(a) #this will output 10.5
```

## Boolean datatype

This is a special data type that represents one of two possible values: True or False. Boolean values are often used in conditional statements and loops. Remember the first letter in "True" and "False" must be capitalized.

```
a = True

if (a==True):

print("a is true")

else :

print("a is not true")
```

## Sequence

These include strings, lists, tuples, and byte sequences. Strings are used to represent textual data, while lists and tuples are used to store collections of items. Byte sequences are similar to strings but represent binary data.

```
my_tuple = ("a", "b","c","d",5,6,7,8)

print(my_tuple)
```

## Sets

These are unordered collections of unique items. Python also has a frozenset type that is similar to sets, but is immutable.

```
my_set = {1,2,3,4}

print(my_set)
```

## Dictionaries

These are unordered collections of key-value pairs. They are used to store and retrieve data based on a unique value.

```
my_dict = {"name":"Shubham" , "age":19}

print(my_dict['name'])

print(my_dict)
```

## None

This is a special data type that represents the absence of a value. In other languages like Java, C++, c, Javascript, etc null is used rather than None.

`a = null1`

Please Follow my coming blogs on related topics.

Thank you for your time.