---
layout: post
title:  "JMH for LLMs"
date:   2025-08-22 07:08:09 +0200
categories: "Java Performance"
published: false
---

## The Idea
I recently had a conversation about Java casting performance with Copilot, mostly to test the model, but also open to learn
something new. Very soon, Copilot suggested to generate a JMH benchmark for me, and being curious, I agreed. What followed
was very interesting, because it became clear pretty quickly, that I had overstretched the capabilities of the model 
significantly. This inspired the following idea: While JMH benchmarks are typically used to measure runtimes of Java
code snippets, why not use it differently, by comparing the ability of different LLMs to create JMH benchmarks
to answer subtle performance related questions.

## The Rules
Thus, I decided to run a competition between different models, which me being the referee. The conversation with each model
would start with the following prompt:

> I know that casts in Java are pretty cheap, since JVM implementations go at great lengths to make them as performant
> as possible. Still, even cheap operations might add up, if executed in large enough numbers. Given the fact that Java 
> collections are operating on plain `Object` references, with implicit casts being inserted by the compiler as needed, 
> typical Java programs contain far more casts than meets the eye. So, to make this more concrete, I'm wondering how much casting
> overhead is involved when iterating over an `ArrayList<Integer>`. Is it always negligible, or can it have a significant impact
> in tight loops? 
> 
> Please don't answer my questions directly: Instead, I would like you to generate a JMH benchmark class, that shines some light
> on this topic. The benchmark should be small and finish in less than 1 minute. Adding some comments to the source code as to 
> how to interpret the results would be a bonus, but please try to be verbose. Also, please restrict your answer to the JMH 
> benchmark class and note that I don't need instructions about how to run it. I will execute the benchmark, and send you the 
> results, so that we can look into it together.

I would then run the benchmark, and respond with 

> Thanks, here are the results of the benchmark:
> 
> ${JMH results table}
>
> Would you like to refine the benchmark? If you think that the numbers make sense,
> and the benchmark should stay as is, please just answer with no. Otherwise, please restrict your answer to an updated
> JMH benchmark class that I should run instead.

After that I would end the conversation, and discuss potential problems in the generated benchmarks. Last but not least, I will
give a score out of `{0, 1, 2}` with
* `0` meaning useless, or even misleading
* `1` meaning OK, though there is room for improvement
* `2` meaning very good.

This score will be mostly affected by the second attempt of the model, if there is one, since this iterative approach also most
closely resembles how humans write benchmarks. I could of course give the models multiple chances to refine their benchmarks,
but I decided to stop after one iteration, since I'm doing this alone, and want to limit the scope of this article and the 
amount of benchmark code to discuss.

## The Participants

The next problem choosing the participants. Again, due to resource constraints, I want to limit myself to 3 models, thus it's 
very likely that your favorite LLM is not in the list. It should however be straight forward enough to replicate what I just
did for any model, at least if you don't wait for too long, because it's only a matter of time till models will learn about
this article in one way or another. So without further ado, here are the participants:
* Microsoft Copilot using `Smart (GPT-5)` mode
* Google Gemini 2.5 Pro
* deepseek V3 using `DeepThink` mode

## The Competition
### Microsoft Copilot

### Google Gemini 2.5 Pro

### deepseek V3

## My Benchmark to answer the Prompt

## Conclusion


