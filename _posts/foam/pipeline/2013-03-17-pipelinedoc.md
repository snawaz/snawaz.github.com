---
layout: doc
title: Documentation of Pipeline
permalink: /foam/pipeline/doc.html
---

Pipeline library defines functions for a variety of purposes (e.g. generating, filtering, transforming, ordering, etc) that operate on ranges of elements. 
Note that a range is defined as `[begin, end)` where *end* refers to the element past the last element to inspect or modify.

###Anatomy of Pipeline

To make it easier for understanding how pipeline works and what each function does, this documentation classifies functions based on their behavior 
and defines some terminologies along the way.

- A **pipe** is a function which either generates (or prepares) data or processes them. 

- A pipe which generates or prepares data is called **generator pipe**.

- A pipe which processes data is called **processor pipe**. 

- A **pipeline** consists of many pipes separated by **pipe operator |**. It could have zero or one generator pipe, and zero or *more* processor pipes.

- A pipeline with zero generator pipe is called **unbounded pipeline**.

- A pipeline which has one generator pipe is called **bounded pipeline**. In a bounded pipeline, the generator pipe is always the first pipe.

Consider this pipeline,

{% highlight cpp%}
auto evens = from(v)  |  filter(e % 2 == 0)  |  transform(e * 2);

             ^^^^^^   ^  ^^^^^^^^^^^^^^^^^   ^  ^^^^^^^^^^^^^^^^
               |      |          |           |        |
               |      |          |           |        procoessor-pipe
               |      |          |           pipe-operator
               |      |          processor-pipe
               |      pipe-operator
               generator-pipe
{% endhighlight %}

As indicated using ascii diagram above, *from*, *filter* and *transform* are called pipes. *from* is a generator pipe which only prepares the data. *filter* and *transform* are called processsor pipes which process the data one after another. The symbol | between two pipes is *pipe-operator*. In a typical pipeline, the first pipe is a generator pipe, and rest are processor pipes, separated by pipe operators.

    //this is how a typical pipeline looks like
    auto results = generator-pipe | processor-pipe1 | processor-pipe2 | processor-pipe3; 

There are lots of pipes belonging to each group. So it is easier to understand if we know which group they belong to.

###General Guidelines

Most of the signature has return type mentioned as *implementation-defined*.

#####Generator Pipes

<ul>
{% include JB/setup %}
{% assign allposts = site.posts %}
{% for post in allposts reversed  %}
    {% if post.type == "generatorpipe" %}
<li><a href="{{ post.url }}">{{ post.title }}</a></li>  
    {% endif %}
{% endfor %}
</ul>

#####Processor Pipes

<ul>
{% include JB/setup %}
{% assign allposts = site.posts %}
{% for post in allposts reversed %}
    {% if post.type == "processorpipe" %}
<li><a href="{{ post.url }}">{{ post.title }}</a></li>  
    {% endif %}
{% endfor %}
</ul>
