---
layout: foam
title: Documentation of Pipeline
type: pipelinedoc
permalink: /foam/pipeline/documentation.html
---

###Anatomy of Pipeline

A pipeline consists of many pipes:

 - One generator pipe
 - Zero or more processor pipes. 

A pipe is a function which either generates (or prepares) data or processes them. Consider this pipeline,

{% highlight cpp%}
auto evens = from(v) | filter(e % 2 == 0) | transform(e * 2);
{% endhighlight %}

Here *from*, *filter* and *transform* are called pipes. *from* is a generator pipe which only prepares the data. *filter* and *transform* are called processsor pipes which process the data one after another. So in a typical pipeline, the first pipe is a generator pipe, and rest are processor pipes:

    //this is how a typical pipeline looks like
    auto results = generator-pipe | processor-pipe1 | processor-pipe2 | processor-pipe3; 

We would be using these terminologies in the rest of this article. There are lots of pipes belonging to each group. So it is easier to understand if we know which group they belong to.

###General Guidelines

Most of the signature has return type mentioned as *implementation-defined*.

#####Generator Pipes

<ul>
{% include JB/setup %}
{% assign allposts = site.posts %}
{% for post in allposts  %}
    {% if post.type == "generatorpipe" %}
<li><a href="{{ post.url }}">{{ post.title }}</a></li>  
    {% endif %}
{% endfor %}
</ul>

#####Processor Pipes

<ul>
{% include JB/setup %}
{% assign allposts = site.posts %}
{% for post in allposts  %}
    {% if post.type == "processorpipe" %}
<li><a href="{{ post.url }}">{{ post.title }}</a></li>  
    {% endif %}
{% endfor %}
</ul>
