---
layout: pipe
title: geometric_sequence
type: generatorpipe
permalink: /foam/pipeline/geometric_sequence.html
---

{% highlight cpp%}
template<typename T>
impl-dfn-deferred-range geometric_sequence(T start, T ratio);
{% endhighlight %}

Creates a deferred sequence of infinite size. As the name implies, the sequence is an [geometric sequence][1].

[1]:http://en.wikipedia.org/wiki/Geometric_progression

*Parameters*

- start - the first element of the sequence.
- ratio - the ratio of the two consecutive elements.

Each consecutive element is computed using the formula

    int next = current * ratio; //initially current is start.

*Return*

A deferred-range of infinite size. Since the returned range is an infinite range, it can be used with *take()* or *take_while()* processor-pipe.

*Example*

{% highlight cpp%}
#include <iostream>
#include <vector>
#include <foam/composition/pipeline.h>

int main()
{
    using namespace foam::composition;

    auto numbers = geometric_sequence(3,2)  //infinite sequence!
                 | take(5);                 //take first 5 elements

    for(int i : numbers)
        std::cout << i << " ";
}
{% endhighlight %}

Output

    3 6 12 24 48
