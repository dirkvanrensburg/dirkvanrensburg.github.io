---
published: false
layout: post
title: "TypeDoc and Angular 2"
date: 2017-01-11 14:07:45 +1100
comments: true
categories: 
---

In getting TypeDoc to work I ran into errors because of the 'moduleId: module.id' thing in Angular components. TypeDoc complains that it doesn't know what module is.

## What is moduleId for


Can we tell TypeDoc what it is?

`declare var module: any;`

## How declare works 

using `var` will override a var defined earlier

declare will use the one already available by that name.

TODO: Find the docs.

TODO: Show some code



