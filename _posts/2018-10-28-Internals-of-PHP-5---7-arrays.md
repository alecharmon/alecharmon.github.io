---
title: Internals of PHP 5 & 7 arrays.
layout: post
description: >-
  In the documentation for PHP 7 there is a small note that Internal array
  pointers are not implemented in PHP 7’s foreach. As someone who…
date: '2018-10-28T18:32:38.192Z'
categories: []
keywords: []
featured_image: images/php-array-internals/hero.gif#wide
author: alec
# slug: /internals-of-php-5-7-arrays
---


In the documentation for PHP 7 there is a small note that Internal array pointers are not implemented in PHP 7’s `foreach`. As someone who has built iterable classes in PHP before this sent me on a puzzling adventure on how the internal of \`foreach\` works in a broad sense but also the implications of what this means for compatibility between PHP 5 and PHP 7.

### An Example to Get us Going:

Let’s look at this block of code and evaluate twice, Once in PHP 5.6 and again in PHP 7.2

$array = \[1, 2, 3, 4, 5\];  
$ref = &$array;

foreach ($array as $key => $val) {

    var\_dump($val);

    $array\[$key+1\] = 0;

}

PHP 5

1,0,0,0,0

PHP 7

1,2,3,4,5

Pretty dramatic right? Though this it is hard to come up with a scenario where you would see a block like this in a production environment it still makes me nervous. Let’s dive in!

### What are references anyway?

In a plain sense, a reference is an address to where in memory your object lives. This value is immutable and can be referenced by one or more variables. In PHP we can define one object and bind it to a variable and then set the value of it.

Every PHP object stores information about how many times it is referenced by a variable variables in a value called a\`refcount\` for the instance’s `refcount` or is or has been referenced by a user-defined reference to another variable, `is_ref`. Where `refcount` can be arbitrarily high, because it initializes at one but increments once for every time it is referenced. The user-defined variable `is_ref` is a boolean which will define if the object has been referenced directly by a user. reads as “Has this object been referenced directly by a user”? Using `xdebug_debug_zval()` Let’s look at our example to illustrate.

PHP > $array = \[1,2,3,4\];  
PHP > xdebug\_debug\_zval(‘array’);

(refcount=1, is\_ref=0)

$ref = &$array;

PHP > xdebug\_debug\_zval(‘array’);

(refcount=2, is\_ref=1)

PHP > xdebug\_debug\_zval(‘ref’);

(refcount=2, is\_ref=1)

> Notice that since the original `$array` variable and `$ref` reference the same object they have the same refcount and is\_ref values.

### How this applies to `foreach.`

PHP implicit tries to create as few copies an object as possible to be conscious of memory usage. In practice this means whenever a modification of an object with a refcount of greater than or equal to 2 is attempted then we copy that object and perform the modification there. Looking at a simple example below we see that foreach implicitly creates a new variable that points to the array it is iterating over.

PHP > $arr = \[1\];  
PHP > xdebug\_debug\_zval(‘arr’);

(refcount=1, is\_ref=0)

PHP > foreach ($arr as $v) {xdebug\_debug\_zval(‘arr’);}

(refcount=2, is\_ref=0)

This essentially is an implementation of the [Copy on Write](https://en.wikipedia.org/wiki/Copy-on-write) (COW) principle which lets programs share memory without worrying about having concurrent programming issues.

There is a qualification here; if the array has referenced directly by the user then we don’t want to create a copy because PHP assumes that we want the changes to propagate. We can then observe the rule that if the `is_ref` is true (1) then we don’t want to copy on modification, instead, we want to access the original object.

Let’s go back to our original example with one modification, commenting out our declared reference:

$array = \[1, 2, 3, 4, 5\];  
//$ref = &$array; Let’s not reference the array this time.

foreach ($array as $key => $val) {

     var\_dump($val);

     $array\[$key+1\] = 0;

}

Has the output: `1,2,3,4,5` in both PHP 5 and 7.

This is because in our loop we are still changing the values in the original array but the \`$ref\` is always coming from a copy of the array. It is not until we uncomment our second line and create a reference that we tell our program to not make a copy of our array.

This kind of behavior is confusing and breaks the implementation of COW since it makes it obvious that PHP’s memory management is not necessarily transparent or consistent.

This problematic implementation was fixed in PHP by removing the usage of IAP (internal array pointers). PHP 7 gets around this issue by only creating copies of arrays in the foreach loop when a modification is attempted. That is why in our original example we don’t see any modifications to the array while it is being iterated upon.

Since editing the array you iterating over is not very common what is observed is an increase in general performance by this new implementation. The caveat here is that existing PHP code that modifies arrays while iterating over them will have a negative impact on performance.

References:

[http://www.PHPinternalsbook.com/zvals/memory\_management.html](http://www.PHPinternalsbook.com/zvals/memory_management.html)

[https://PHPinternals.net/docs/zend\_hash\_foreach\_val](https://PHPinternals.net/docs/zend_hash_foreach_val)