---
layout: post
title:  "JMH for LLMs"
date:   2025-08-22 07:08:09 +0200
categories: "Java Performance"
published: false
---

> I know that casts in Java are pretty cheap, since JVM implementations go at great lengths to make them as performant
> as possible. Still, even cheap operations might add up, if executed in large enough numbers. Given the fact that Java 
> collections are actually operating on plain `Object` references, with implicit casts being inserted by the compiler as needed, 
> typical Java programs contain far more casts than meets the eye. So, to make this more concrete, I'm wondering how much casting
> costs when iterating over an `ArrayList<Integer>`. Is it negligible, or can it have a significant impact? 
> 
> Please don't answer my questions yet: Instead, I would like you to generate a JMH benchmark class, that shines some light on 
> this topic. Ideally the benchmark should finish in less than 1 minute. I can always tweak the configuration, if more accuracy is 
> required. Adding some comments to the source code as to how to interpret the results would be a bonus, but please try
> to be verbose. Also, please restrict your answer to the JMH benchmark class. I will run the benchmark, and send you the results,
> so that we can look into it together.
