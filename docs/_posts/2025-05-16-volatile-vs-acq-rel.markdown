---
layout: post
title:  "About the subtle differences between volatile and acquire release semantics in Java"
date:   2025-05-16 08:52:34 +0200
categories: "Java memory-model"
---
In this blog post I want to take a close look at the subtle difference between using 
* [getAcquire](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/invoke/VarHandle.html#getAcquire(java.lang.Object...)) with
[setRelease](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/invoke/VarHandle.html#setRelease(java.lang.Object...))
* and using [getVolatile](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/invoke/VarHandle.html#setRelease(java.lang.Object...))
with [setVolatile](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/invoke/VarHandle.html#setRelease(java.lang.Object...)).

At first, I will repeat the basic concepts, using simple examples. Then I want to discuss two classical mutual exclusion 
algorithms, and explain why they rely on volatile semantics, and acquire-release is not enough. Hopefully this will clarify
the difference between these two concepts for you. It certainly did so for me.

### getAcquire and setRelease
Let's start the discussion with `getAcquire` and `setRelase`: According to the Javadocs, `getAcquire` "returns
the value of a variable, and ensures that subsequent loads and stores are not reordered before this access." and `setRelase`
"sets the value of a variable to the newValue, and ensures that prior loads and stores are not reordered after this access."

This means that `getAcquire` and `setRelease` can be used to establish happens before relationships across thread boundaries:
If thread `A`, after an elaborate procedure finally uses `setRelease` on a boolean `done` flag, all other threads that observe
true when calling `getAcquire` on done, can safely assume that `A` is indeed done. Translated to Java, this means
```java
AtomicBoolean initialized = new AtomicBoolean();

void threadA() {
  state = initialze();
  initialized.setRelease(true); // (1)
}

void threadB() {
  if (initialized.getAcquire()) {
    // Can safely assume that state is initialized because a 
    // happens-before-relationship has been established with (1)
    assert state != null;
  }
}
```
Note that the following code is broken and is just waiting to fail when you least expect it, since no happens-before-relationship
is established by plain reads and writes:
```java
boolean initialized;

void threadA() {
  state = initialze();
  initialized = true;
}

void threadB() {
  if (initialized) {
    // This assertion can fail, since the compiler 
    // or the hardware might have reordered operations.
    assert state != null;
  }
}
```
Due to the strong memory model of the X86 architecture (see [Chapter 10.2 in Volume 3 in the Combined Volume Set of Intel® 64 and IA-32 Architectures Software Developer’s Manuals](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html))
code like this has a high chance of working unless JIT interferes, but it might start to fail randomly on ARM. This is not
just a theoretical possibility, but can easily be [demonstrated in tests](https://github.com/mlangc/java-snippets/blob/cd9c3a8fa4b8d49894f70d4d72efec8e7f92e431/src/test/java/at/mlangc/concurrent/seqcst/vs/ackrel/SafePublicationTest.java#L18).

Note that exactly the same applies to publishing objects with non-final fields:
```java
class HasNonFinalField { 
  int nonFinalField;
    
  HasNonFinalField() {
      this.nonFinalField = initializeWithPositiveValue();
  }
}

HasNonFinalField published;

void threadA() {
  published = new HasNonFinalField();  
}

void threadB() {
    if (published != null) {
        // This assertion can fail, since the compiler 
        // or the hardware might have reordered operations.
        assert published.nonFinalField > 0;
    }
}
```
To fix this, we could transform it into
```java
AtomicReference<HasNonFinalField> published = new AtomicReference<>();

void threadA() {
  published.setRelease(new HasNonFinalField()); // (1)
}

void threadB() {
  if (published.getAcquire() instanceof HasNonFinalFields obj) {
    // This won't fail, since a happens-before-relationship has been established
    // with (1)
    assert obj.nonFinalField > 0;
  }
}
```



Note that this is actually identical to the [ARM developer documentation for the LDAR and STLR instructions](https://developer.arm.com/documentation/102336/0100/Load-Acquire-and-Store-Release-instructions).
X86 has a relatively strong ordering model by default (see [Chapter 10.2 in Volume 3 in the Combined Volume Set of Intel® 64 and IA-32 Architectures Software Developer’s Manuals](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html)),
which means that loads and stores always have acquire-release semantics.




You’ll find this post in your `_posts` directory. Go ahead and edit it and re-build the site to see your changes. You can rebuild the site in many different ways, but the most common way is to run `jekyll serve`, which launches a web server and auto-regenerates your site when a file is updated.

Jekyll requires blog post files to be named according to the following format:

`YEAR-MONTH-DAY-title.MARKUP`

Where `YEAR` is a four-digit number, `MONTH` and `DAY` are both two-digit numbers, and `MARKUP` is the file extension representing the format used in the file. After that, include the necessary front matter. Take a look at the source for this post to get an idea about how it works.

Jekyll also offers powerful support for code snippets:

{% highlight ruby %}
def print_hi(name)
  puts "Hi, #{name}"
end
print_hi('Tom')
#=> prints 'Hi, Tom' to STDOUT.
{% endhighlight %}

Check out the [Jekyll docs][jekyll-docs] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll Talk][jekyll-talk].

[jdk9-memory-order-modes]: https://gee.cs.oswego.edu/dl/html/j9mm.html
[rust-atomics-and-locks-memory-ordering]: https://marabos.nl/atomics/memory-ordering.html

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/

