---
layout: pipe
title: arithmetic_sequence
type: generatorpipe
permalink: /foam/pipeline/arithmetic_sequence.html
---


{% highlight cpp%}
template<typename T, typename U>
impl-dfn-deferred-range arithmetic_sequence(T start, U difference) 
{% endhighlight %}

Creates a deferred sequence of infinite size. As the name implies, the sequence is an [arithmetic sequence][1].

[1]:http://en.wikipedia.org/wiki/Arithmetic_progression

*Parameters*

- start - the first element of the sequence.
- difference - the difference of the two consecutive elements. 

Each consecutive element is computed using the formula

    int next = current + difference; //initially current is start.

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

    auto numbers = arithmetic_sequence(10, 7)  //infinite sequence!
                 | take (6) ;                  //take first 6 elements
                
    for(int i : numbers)
        std::cout << i << " ";
}
{% endhighlight %}

Output

    10 17 24 31 38 45
