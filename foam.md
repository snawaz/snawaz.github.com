---
layout: page
title : Foam C++ Library
header : Post Archive
group: navigation
---

Foam is a collection of small yet cross-platform C++ libraries and utilities written in C++11.

###License

Foam is distributed under the [Boost Software License][license], Copyright &copy; 2013-2014 Sarfaraz Nawaz.

[license]:http://www.boost.org/LICENSE_1_0.txt

###Supported Platforms

Foam has been tested with following platforms and compilers:

- Win32 using GCC 4.7.0 (MinGW).

###Source code

The source code can be found at:

 - [https://github.com/snawaz/foam](https://github.com/snawaz/foam)

###Libraries

{% include JB/setup %}
{% assign allposts = site.posts %}
{% for post in allposts  %}
    {% if post.layout == "library" %}
- [{{ post.name }}]({{ BASE_PATH }}{{ post.url }})   
  > {{ post.description }}  
    {% endif %}
{% endfor %}

###A bit about Foam

The libraries and utilities are written out of fun while exploring various features of C++11. New ideas keep coming to my mind and I keep updating and refactoring the source code, making the programming experience better and better. It is a work in progress, with the following objectives and goals:

 - Cool and useful functionality.
 - Easy to use. Aesthetic coding.
 - Concise yet readable.
 - Performant.
