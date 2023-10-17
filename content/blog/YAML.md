---
title: "YAML - from basics to advanced"
description: "explaining YAML from basics to advanced · What is YAML? YAML (which stands for 'YAML Ain't Markup Language') is a human-readable data serialization..."
dateString: October 2023
draft: false
tags: ["YAML", "markdown", "DevOps", "Open-source", "GitHub"]
weight: 101
cover:
    image: "blog/yaml.avif"
---

## What is YAML?

YAML (which stands for "YAML Ain't Markup Language") is a human-readable data serialization language. We will talk later about what a serialization language is.

YAML is used for configuration files, exchanging data between programming languages and storing data. YAML files are extensively used in the DevOps field. YAML is considered better than its alternative because of its easy-to-read way.

It is often used for configuration files, data exchange between programming languages, and storing data in a structured and easy-to-read way.

YAML files are denoted by **".YML"** OR **".YAML"**.

Initially, YAML was called "Yet Another Markup Language". but the name was changed to simply "YAML Ain't Markup Language" (or just YAML) in order to emphasize that YAML is not just another markup language like XML or HTML, but rather a data serialization language that is more human-readable and easier to work with.

## Example of YAML file:
This file includes almost all important YAML concepts. It's a bit lengthy but very important to understand the concepts of YAML. Do check it out.


```
"name": "Shubham"
# key value pairs
---
# lists
"list":
  - "apple"
  - "mango"
  - "three"
  - four
---
# documents are seperated by ---
# "..." denotes that documents has ended

# String variables ->
a: "string"
b: string
c: 'string'
float: 10.12
int: 23
boolean: No #n , N, false, False, FALSE
booleanTwo: Yes #y, Y, True, true, TRUE
nulltext: Null #null NULL
---
d: |
  This is how 
  we write multi line data

e: >
  This is how we write
  one line data in multiple lines

# specify the type:
zero: !!int 0
commaValue: !!int +100_000 #equivalent to 100,000
booleanThree: !!bool Yes
stringtwo: !!str "Shubham"
floatTwo: !!float 12.12
nulltext: !!null Null #null NULL

# maps: key value pairs
map example:
  name: Shubham
  age: 19
  country: India
# another way of representing a map
map example two: {name: Shubham, age: 19, country: India}

# sets are used to store items
expertise: !!set
  ? Web dev
  ? Android dev
  ? DevOps

# Dictionary or !!omap
students: !!omap
  - Shubham:
    age: 19
  - Ram:
    age: 23
  - Shayam:
    age: 24

# Using anchors
age_and_gender: &19_and_male
  gender: male
  age: 19

person1:
  name: Shubham
  <<: *19_and_male
person2:
  name: Ram
  <<: *19_and_male
```


## Data serialization:

Data serialization is the process of converting data from one format to another. When data is serialized, it is transformed into a format that can be easily stored or transmitted across different applications or systems.

The reverse process of converting the serialized data back to its original format is called deserialization.

For Data serialization, we use the data serialization formats like JSON, YAML and XML. YAML is one of the more popular options out of these. JSON and YAML are more human-readable than XML.

## Advantages of YAML:

Here are some benefits of YAML:

***Human-readable:*** One of the biggest benefits of YAML is that it is easy for humans to read and write. Unlike JSON or XML, YAML documents use simple indentation and whitespace to structure data, making them easier to understand and modify.

***Supports complex data structures:*** YAML supports a wide range of data structures, including lists, dictionaries, and nested structures, making it ideal for working with complex data.

***Language agnostic:*** YAML is language agnostic, meaning that it can be used with any programming language without requiring any special libraries or tools.

***Less verbose:*** YAML generally requires less syntax compared to other data serialization formats like JSON or XML, making it cleaner and more concise.

***Supports comments:*** YAML supports comments, which allow developers to add notes and explanations within the document.

***Great for configuration files:*** YAML is commonly used for configuration files, such as those used in Docker or Kubernetes, because of its simplicity and readability.

These benefits make YAML a popular choice for developers looking for a data serialization format that is easy to work with and supports a wide range of data structures.

## Conclusion:
In conclusion, YAML is a simple, human-readable data serialization format that allows developers to easily exchange and store data across different programming languages and systems. Its support for complex data structures, language agnosticism, and support for comments, make it an ideal choice for developers working in the DevOps field or those dealing with configuration files. With its intuitive syntax and streamlining of complex data structures, YAML is becoming a popular choice for developers looking to improve readability and maintainability of their code.