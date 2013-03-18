---
layout: foam
title: Expression
type: foamlibrary
permalink: /foam/expression.html
description: An expression template library that works great with Foam.Pipeline as well as with the C++ Standard library.
---

Expressions are essentially functors, with an ability to compose itself with other expressions. You start with a simple expression, and then you go on composing more complex expressions using simpler ones.  It can be best understood by examples. So here you go.

{% highlight cpp linenos=table %}

#include <foam/composition/expression.h>  //must include this header

//consider this function which takes a functor as argument
//and then invoke it passing 0 to 9, and print the result.
template<typename Functor>
void f(Functor && fun)
{
  for ( int i = 0 ; i < 10 ; i++ ) 
     std::cout << fun(i) << std::endl; //invoke fun for each i in [0, 10)
}

using foam::composition::expression; 
expression<int> x;  //x is a simple expression (identity expression)

f(x); //print 0 to 9

auto y = x * x - 4 * x + 4; //compose an expression y using x and literals as coefficients
f(y); //print value of (i * i - 4 * i + 4) for each i in [0, 10)

f( (x + 10)* y ); //x and y are expressions. compose further using them 
                  //and pass the resulting expression to f which
                  //prints value of [(i + 10) * (i * i - 4 * i + 4)] for each i in [0, 10)

{% endhighlight %}

###Comparison of expression with lambda

Since expressions are functors, they can be used in place of lambdas, because they simplify the code. 
Consider this code which uses lambda,

{% highlight cpp %}
std::vector<int> v {0,1,2,3,4,5};

auto it = std::find_if(begin(v), end(v), [](int x) { return x == 2; } );

{% endhighlight %}

Using expresson the above `std::find_if` can be written as,

{% highlight cpp %}

expression<int> x; //declare an expression variable first

auto it = std::find_if(begin(v), end(v),  x == 2 );

{% endhighlight %}

which is much more concise. Well that is not where you would use expression (or lambda for that matter), because you would use `std::find` as:

{% highlight cpp %}

auto it = std::find(begin(v), end(v), 2 ); //simplest!

{% endhighlight %}

That is better expressed without expression! But consider this code which uses expression:

{% highlight cpp %}

auto it = std::find_if(begin(v), end(v), (x*x+4) == (4*x) ); //find x such that LHS == RHS
if ( it != end(v) )
  std::cout << *it << std::endl; //prints 2 
                                 //because at x = 2, (x*x+4) is equal to (4*x)
{% endhighlight %}

Compare `std::find_if` in the above snippet, with the lambda version,

{% highlight cpp%}
auto it = std::find_if(begin(v), end(v), [](int x ) { return (x*x+4) == (4*x); } ); 
{% endhighlight %}

The advantage of expression over lambda is that once you declare `x`, you can reuse it and that too in many different ways. Here are few examples,

{% highlight cpp %}

if ( std::any_of(begin(v), end(v), x < 0 ) )
{
    std::cout << "v contains at least one negative number." << std::endl;
}

int n = std::count_if ( begin(v), end(v), x % 2 == 0 ); //count even numbers
std::cout  << "v contains " << n < " even numbers." << std::endl;
{% endhighlight %}

As we see, expressions make the code concise without destroying its readability. On the contrary, it increases the readability.

###Member expression

C++11 has [std::mem_fn][mem_fn] which creates wrappers for member-function-pointers and member-data-pointers. The following example illustrates its usage:

{% highlight cpp %}
struct person
{
    int age;
    std::string name;
    int get_age() const { return age; }
};

std::vector<person> ps = get_persons();

auto age = std::mem_fn(&person::age); //creates a wrapper for person::age 
std::vector<int> ages;

std::transform(ps.begin(), 
               ps.end(), 
               std::back_inserter(ages), 
               age); //for each element e in ps, get `e.age` and add it to vector `ages`.

for(auto && x : ages)
    std::cout << x << std::endl; //print each element of ages
	
{% endhighlight %}

The problem is that the wrapper `age`, created by `std::mem_fn`, doesn't support composition. We **cannot** use it to do this, for example:

{% highlight cpp %}
//pseudocode : it would not compile
std::transform(ps.begin(), 
               ps.end(), 
               std::back_inserter(ages), 
               10 * age );  //add 10 times of age to the vector `ages`.

{% endhighlight %}

To accomplish this, composition library provides a function called `memexp` which is just like `std::mem_fn`, except that it supports composition. So with it we can do this:

{% highlight cpp %}

auto age = memexp(&person::age); //creates a composable wrapper!
std::transform(ps.begin(), 
               ps.end(), 
               std::back_inserter(ages), 
               10 * age ); //okay now

{% endhighlight %}

Or we may use member function of `person` instead, as:

{% highlight cpp %}

auto age = memexp(&person::get_age); //notice the difference here!
std::transform(ps.begin(), 
               ps.end(), 
               std::back_inserter(ages), 
               10 * age ); //same as before!

{% endhighlight %}

We can use it in filtering as:

{% highlight cpp %}

std::vector<person> filtered;
std::copy_if(ps.begin(), 
             ps.end(), 
             std::back_inserter(filtered), 
             age >= 10 && age <= 40 ); //filter all person between age >= 10 && age <= 40

{% endhighlight %}

which is *functionally* equivalent to this lambda version:

{% highlight cpp %}

std::vector<person> filtered;
std::copy_if(ps.begin(), 
             ps.end(), 
             std::back_inserter(filtered), 
             [](person const & p) { return p.age >= 10 && p.age <= 40 } );

{% endhighlight %}		

The expression version is definitely more readable in addition to being concise.

Even more, `memexp` allows us to pipe as shown below, using `_m` which is a wrapper object of `memexp`:

{% highlight cpp %}

//first get the name of person object,
//which then gets passed to the next pipe 
//which returns the size of the name.
auto namelen = _m(&person::name) | _m(&std::string::size); //we can also use memexp instead.

std::vector<person> lengths;
std::transform(ps.begin(), 
               ps.end(), 
               std::back_inserter(lengths), 
               namelen); //note: namelen is composable, so we can pass (namelen * 10) instead!

{% endhighlight %}		

which is same as (lambda version):

{% highlight cpp %}

std::vector<person> lengths;
std::transform(ps.begin(), 
               ps.end(), 
               std::back_inserter(lengths), 
               [](person const & p) { return p.name.size(); } );

{% endhighlight %}		

[mem_fn]:http://en.cppreference.com/w/cpp/utility/functional/mem_fn
