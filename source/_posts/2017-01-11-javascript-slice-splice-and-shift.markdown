---
published: false
layout: post
title: "Javascript - slice splice and shift"
date: 2017-01-11 22:06:06 +1100
comments: true
categories: 
---

Working with arrays. Use slice to remove some items from an array, but try to remove the first element. - Not quite what you would expect....

this doesn't work

arr.slice(0,1);

this works:

arr.splice(0,1) and this arr.shift() - so which should you use?

## Slice
Does not modify the original list but returns a new list containing the remainder after removing the element.
To use slice use: arr.slice(- (arr.length-1), 1) <-- This starts at the nth element from the end

## Splice
Mutates the array in place and returns the removed items


## Shift
Removes the first element and shrinks the array

