---
layout: post
title: "JMH Visualizer 1.1.0 Released"
date: 2026-10-13
categories: [ "Java", "JMH", "Visualization" ]
excerpt: "A modernized version of JMH Visualizer with benchmark and parameter filtering has been released."
---

I've invested a considerable amount of time into modernizing [JMH Visualizer](https://github.com/mlangc/jmh-visualizer)
and the related [Gradle plugin](https://github.com/mlangc/gradle-jmh-report).

JMH Visualizer has been my go-to solution for visualizing JMH benchmark results for quite a while. It converts JMH
result files into interactive charts, and runs entirely in the browser.

If you are using [JMH](https://github.com/openjdk/jmh) regularly you should definitely have a look. The following
screenshots from a HashMap related benchmark are meant to make this as easy as possible:

<figure markdown="1" class="screenshot-group">
![HashMaps Iteration Performance, Operations per Second](/assets/img/2026-10-13-jmh-visualizer-1-1-screenshot1-1.png)
![HashMaps Iteration Performance, Allocation Rate](/assets/img/2026-10-13-jmh-visualizer-1-1-screenshot1-2.png)
<figcaption>HashMaps iteration performance</figcaption>
</figure>

<figure markdown="1" class="screenshot-group">
![HashMaps Get Any Performance, Operations per Second](/assets/img/2026-10-13-jmh-visualizer-1-1-screenshot2-1.png)
![HashMaps Get Any Performance, Allocation Rate](/assets/img/2026-10-13-jmh-visualizer-1-1-screenshot2-2.png)
<figcaption>HashMaps getting any entry performance</figcaption>
</figure>

<figure markdown="1" class="screenshot-group">
![HashMaps Put Performance, Operations per Second](/assets/img/2026-10-13-jmh-visualizer-1-1-screenshot3-1.png)
![HashMaps Put Performance, Allocation Rate](/assets/img/2026-10-13-jmh-visualizer-1-1-screenshot3-2.png)
<figcaption>HashMaps put performance</figcaption>
</figure>

<figure markdown="1" class="screenshot-group">
![Comparing 2 Runs, Summary](/assets/img/2026-10-13-jmh-visualizer-1-1-screenshot4-1.png)
![Comparing 2 Runs, Details](/assets/img/2026-10-13-jmh-visualizer-1-1-screenshot4-2.png)
<figcaption>Comparing runs executed with 1 and 2 threads</figcaption>
</figure>

<figure markdown="1" class="screenshot-group">
![Comparing Multiple Runs, Summary](/assets/img/2026-10-13-jmh-visualizer-1-1-screenshot5-1.png)
![Comparing Multiple Runs, Details](/assets/img/2026-10-13-jmh-visualizer-1-1-screenshot5-2.png)
<figcaption>Comparing runs executed with multiple thread counts</figcaption>
</figure>

Assuming that you use the [jmh-gradle-plugin](https://github.com/melix/jmh-gradle-plugin), just add

```groovy
jmh {
    resultFormat = 'JSON'
}
```

to your `jmh` configuration, run your benchmarks, and open `build/results/jmh/results.json` with
[jmh-visualizer](https://mlangc.github.io/jmh-visualizer/). Nothing is uploaded to any server; the tool runs
exclusively in your browser.

Additionally, you can use the [jmh-report plugin](https://github.com/mlangc/gradle-jmh-report), and instruct Gradle to
automatically render a report whenever you run the `jmh` task:

```groovy
jmhReport {
    // This is where the jmh-gradle plugin puts its result files by default
    jmhResultPath = "build/results/jmh/results.json"

    // Whatever you prefer
    jmhReportOutput = "build/reports/jmh"
}

// Runs `jmhReport` every time you execute your benchmarks using `jmh`
tasks.jmh.finalizedBy tasks.jmhReport
```

JMH Visualizer was originally developed by [Johannes Zillmann](https://github.com/jzillmann) who should take almost all
the credit for this very useful tool. Since the related GitHub Projects seem abandoned
(see [here](https://github.com/jzillmann/jmh-visualizer/pull/54)
and [here](https://github.com/jzillmann/gradle-jmh-report/issues/15)), I decided to carry on in my forks of both
the [JMH Visualizer browser application](https://mlangc.github.io/jmh-visualizer/), and the
related [Gradle plugin](https://github.com/mlangc/gradle-jmh-report).

A lot of effort has been put into keeping things exactly as they were from the outside, while not making any compromises
on modernizing the underlying software stack.

There is only one extension that I've added to the previously existing functionality: You can now conveniently select
subsets of benchmark methods and parameters. This is helpful when loading large result files containing many different
benchmark-parameter combinations, like in this screencast:

<figure markdown="1" class="screenshot-group">
<video muted playsinline controls preload="metadata" poster="/assets/img/2026-10-13-jmh-visualizer-1-1-screenshot6-poster.jpg">
<source src="/assets/img/2026-10-13-jmh-visualizer-1-1-screenshot6.mp4" type="video/mp4"/>
Your browser can't play this video; you can <a href="/assets/img/2026-10-13-jmh-visualizer-1-1-screenshot6.mp4">download it</a> instead.
</video>
<figcaption>HashMaps benchmarks, focusing on merge and specific parameters</figcaption>
</figure>

Any feedback is highly appreciated and best delivered via GitHub:

* [JMH Visualizer GitHub Project](https://github.com/mlangc/jmh-visualizer)
* [Gradle JMH Report GitHub Project](https://github.com/mlangc/gradle-jmh-report)