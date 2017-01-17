---
published: false
layout: post
title: "Programming and modulo"
date: 2017-01-12 11:09:56 +1100
comments: true
categories: 
---

Common misperception that the % operator performs modulo, however it is just a a remainder function on positive numbers. It gives an answer on negative numbers however the wrong one.

used to implement a step.


__IDEA__ Fizz buzz using negative.


Java
<pre>
        int a = -13;

        int c = 9;
        int q = a / c;

        int r = a % c;
    
        System.out.println(q);
        System.out.println(r);
        System.out.println("back again: " + (q * c + r) );

</pre>
It sort of works because you can get to x again but the quotient is wrong lets do it long div

     -2  r  5
  |^^^^^^^^^^
9 | -13
    ----
    -18
    ---- 
      5

Java: this is even worse. THe mod should be 0
<pre>
    public static void main(String [] args) {
        int a = -18;

        int c = 9;

        int q = a / c;

        int r = (a % c) + c;



        System.out.println(q);
        System.out.println(r);
        System.out.println("back again: " + (q * c + r) );
    }



</pre>

But does the integer divide work?

statement: -13 / 9 should be -2 and not -1
research: Is this statement correct?

for x/d  -> x = qd + r
for x mod d -> x = qd + r where d is divisor
thus: r = x - qd




therefore




Modulo works like this

10 % 3 = 1 but what about -10 %3 ? most languages (try some more. Javascript and Java works the same) will say that -10 % 3 = -1 however using the arithmetic module says that x modulo c is defined as x = qc + r

division algorithm is to find the largest number smaller than the dividend so that the difference between the dividend and this number is smaller than the divisor.


