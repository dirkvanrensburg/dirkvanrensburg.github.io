---
published: false
layout: post
title: "Java double and NaN"
date: 2017-01-11 12:08:52 +1100
comments: true
categories: 
---

Learn something everyday. today learned something i never really thought about since I always prefer BigDecimal over double when working with numbers I want to be precise.

I recently worked on an integration project and received a bug saying my service is returning a '500' on investigation found that the down stream system is returning

a 'Nan' in a number field which JAXB parses as a null. 

This got me thinking about NaN and why they return this value to me. I realised that they must be using Double to represent numbers on their side and have a divide by zero or something.

I played with double a bit and realised that I never really internalised that a double or float can get stuck in one of 4 states:

* NaN
* infinaty
* negative infinity
* negative0



<pre>



    public static void main(String[] args) {
        double a = 10;

        for (int i=0; i < 1000; i ++) {
            a = a + (a * i);
        }
        System.out.println(a);

        System.out.println("Quotations.main" + (a - 100000000d));

        if (Double.isInfinite(a)) {
            System.out.println("INFINATE!!!!");
        }
        System.out.println(BigDecimal.valueOf(a));

    }


</pre>


