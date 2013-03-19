---
layout: pipe
title: prime_range
type: generatorpipe
permalink: /foam/pipeline/prime_range.html
---


{% highlight cpp%}
template<typename T>
impl-dfn-deferred-range prime_range(T start, T end)
{% endhighlight %}

Creates a deferred range of prime numbers.

*Parameters*

- start - the lower bound of the range to be generated whose first element is *start* if it is a prime number
          or the minimum prime number greater than *start*.

- end - the upper bound of the range to be generated. It is not included in the range even if it is a prime number.

*Return*

A deferred-range of prime numbers.

*Example*

{% highlight cpp%}
#include <iostream>
#include <vector>
#include <foam/composition/pipeline.h>

int main()
{
    using namespace foam::composition;

    auto primes = prime_range(49, 87);

    for(int i : primes)
        std::cout << i << " ";
}
{% endhighlight %}

Output

    53 59 61 67 71 73 79 83
