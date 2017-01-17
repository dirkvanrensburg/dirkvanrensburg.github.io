---
published: false
layout: post
title: "Challenges of polyglot programmer"
date: 2017-01-11 14:14:59 +1100
comments: true
categories: 
---

Chat about importance of learning languages all the time.

talk about the challenges:

* Use all the time to not forget
* Learning the non explict stuff of the language. Conventions and assumptions that are not realy documented or explicitly stated. And even if it is stated in a book you have to keep track of all these rules such as how module names are resolved and scoping rules etc.

This is in my opinion why some people will develop a strong dislike in a language when it clashes significantly with the persons programming knowledge because things does not work as they expect it to. They find that their assumptions based on knowledge of other languages makes it hard to reason about a program in the new language and they make mistakes. 

It is then only natural to blame the language of course!


example nullchecks java vs javascript/typescript

if (x && x.a) 

vs 

if (x != null && x.a)
