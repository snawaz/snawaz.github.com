---
layout: pipe
title: prime_sequence
type: generatorpipe
permalink: /foam/pipeline/prime_sequence.html
---


{% highlight cpp%}
template<typename T>
impl-dfn-deferred-range prime_sequence(T start);
{% endhighlight %}

Creates a deferred prime sequence of infinite size. 

*Parameters*

start - the lower bound of the sequence to be generated whose first element is *start* if it is a prime number 
       or the minimum prime number greater than *start*.

*Return*

A deferred-range of infinite size. Since the returned range is an infinite sequence, it can be used with *take()* or *take_while()* processor-pipe.

*Example*

{% highlight cpp%}
#include <iostream>
#include <vector>
#include <foam/composition/pipeline.h>

int main()
{
    using namespace foam::composition;

    auto primes = prime_sequence(10)  //infinite sequence
                | take(15)            //take first 15 elements

    for(int i : primes)
        std::cout << i << " ";
}
{% endhighlight %}

Output

    11 13 17 19 23 29 31 37 41 43 47 53 59 61 67
