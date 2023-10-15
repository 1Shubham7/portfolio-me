---
title: "Difference between SASS and SCSS"
description: "SASS vs SCSS, Differences and similarities. · You must have heard of SCSS or SASS, but do you know the full form of the two tools, how similar these two..."
dateString: July 2023
draft: false
tags: ["SASS", "CSS", "SCSS", "Web Design"]
weight: 101
cover:
    image: "blog/sass.png"
---

You must have heard of SCSS or SASS, but do you know the full form of the two tools, how similar these two terms are ? and what are the differences between them? join me in my exploration of these two CSS Preprocessors.

## What is SASS ?
SASS (Syntactically Awesome Style Sheets) is a CSS preprocessor that allows you to write more maintainable and flexible code for your stylesheets.

Sass works by providing a set of extensions to CSS, which are then compiled to standard CSS syntax, so they can be used in your web project. Here is an example of how a file with SASS looks like.


```
$primary-color: #ff0000;

$secondary-color: #00ff00;

@mixin center-element {

display: flex;

justify-content: center;

align-items: center;

}

.header {

background-color: $primary-color;

color: #ffffff;

padding: 10px;

h1 {

font-size: 24px;

margin-bottom: 10px;

}

&.large {

font-size: 32px;

}

&.small {

font-size: 18px;

}

}

.container {

@include center-element;

background-color: $secondary-color;

height: 200px; width: 200px;

}
```

## What is SCSS ?
SCSS stands for Sassy CSS. It is also a CSS preprocessor. SCSS is a superset of CSS, which means that any valid CSS code is also valid SCSS code. SCSS syntax is similar to CSS syntax with some additional features and enhancements, such as variables, nesting, mixins, inheritance, and more.

SCSS can help to shorten your CSS code, make it more maintainable, and easier to read. For example, you can define variables in SCSS, which allows you to use them throughout your CSS code.

## Example of SCSS code:


```
$primary-color: #007bff;

$secondary-color: #6c757d;

@mixin center-element {

display: flex;

align-items: center;

justify-content: center;

}

.container {

width: 100%;

.header {

background-color: $primary-color;

color: white;

padding: 20px;

h1 {

font-size: 24px;

}

}

.main {

background-color: white;

padding: 20px;

.content {

margin-bottom: 20px;

p {

color: $secondary-color;

}

}

}

.footer {

background-color: $secondary-color;

color: white; padding: 20px;

}

}

.centered-box {

@include center-element;

width: 200px;

height: 200px;

background-color: $primary-color;

color: white;

}
```


## Difference between SCSS and SASS:
### ***SCSS:***
SCSS is a superset of CSS, which means that any valid CSS code is also valid SCSS code.

- SCSS uses the same syntax as CSS, with curly braces {} and semicolons ;.

- It supports all the features of CSS, including nested selectors, variables, mixins, and more.

- SCSS files have the .scss file extension.

### ***Sass:***
Sass has its own syntax, which is more concise and indentation-based.

- It doesn't use curly braces {} or semicolons ;. Instead, indentation is used to indicate nesting and separate properties.

- It doesn't require the use of brackets or semicolons, which can make the code look cleaner and more readable.

- Sass supports all the features of SCSS, including nested selectors, variables, mixins, and more.

- Sass files have the .sass file extension.

## Conclusion :
In conclusion, SCSS and SASS are both CSS preprocessors that allow developers to write more maintainable, flexible, and organized code by extending the capabilities of standard CSS syntax. SCSS is a superset of CSS and uses the same syntax as CSS, while SASS has its own syntax that is more concise and uses indentation instead of curly braces and semicolons. Both of these preprocessors provide features such as variables, nesting, mixins, inheritance, and more, which can help to make code more readable and maintainable. It's up to personal preference which preprocessor to use, and both have their pros and cons.

Thank you for reading.

Follow me for more such blogs.