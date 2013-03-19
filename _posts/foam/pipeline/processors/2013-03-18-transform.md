---
layout: pipe
title: transform
type: processorpipe
permalink: /foam/pipeline/transform.html
---


{% highlight cpp%}
template<typename Iterable>
impl-dfn-deferred-range filter(Iterable range);
{% endhighlight %}

Creates a generator-pipe.

*Parameter*

- range - the range from which to create the pipe.

*Return*

A deferred-range of same size as that of the input *range*.

*Example*

{% highlight cpp%}
#include <iostream>
#include <vector>
#include <foam/composition/pipeline.h>

int main()
{
    using namespace foam::composition;

    std::vector<int> v{1,2,3,4,5,6,7,8,9,10};
    auto evens = from(v) 
               | filter([](int i) { return i % 2 == 0; }); //filter even numbers
    for(int i : evens)
        std::cout << i << " ";
}
{% endhighlight %}

Output

    2 4 6 8 10
