---
layout: post
title: Casablanca
tags: [c++, template, microsoft, casablanca]
---

[casa]:http://msdn.microsoft.com/en-us/devlabs/casablanca.aspx
[where]:http://msdn.microsoft.com/en-us/library/d5x73970(v=vs.110).aspx
[enable]:http://en.cppreference.com/w/cpp/types/enable_if
[base]:http://en.cppreference.com/w/cpp/types/is_base_of
[faulty-as]:http://ideone.com/GpL0r
[nonfaulty-as]:http://ideone.com/np3A9
[better-as]:http://ideone.com/lURCw
[bad-cast]:http://ideone.com/gn3Y1


###Introduction

[Casablanca][casa] is a codename for the cloud-based client-server communication SDK being developed by Microsoft. It is currently in beta phase which I downloaded just to play around with. Before anything else, I first started looking at the class templates and other header files. That is something I often do - reading code written by others. Anyway, soon I felt like commenting on the implementation of few classes and functions. So I thought why not start a blog so that others (maybe the ones who wrote the code) could also review my review? Sounds like a good idea.

Obviously, it isn't possible to comment on the overall design of the SDK, as I don't have access to the full code-base and the design documentation. However, it is certainly possible to comment on some pieces, the small yet important pieces of the code which comes with the SDK. In this blog, I've attempted to list few ideas which can help improving the code in many ways.

###Enforcing type-constraint

Type-constraint lets compiler find error in your code at compile-time itself. The idea is pretty much straightforward : first of all, programmers must strive to write bug-free code. However, they're humans and humans make mistakes, introducing bugs in their code. If they do, then it is the compiler's job to find the errors, instead of silently accepting the faulty code. Enforcing type-constraint in template code enables compiler do all such checks on behalf of programmers in case they make mistakes.

So here is a class which has a function template which is to be instantiated with *only* derived class type, but no such type-constraint is enforced on its implementation:
{% highlight cpp %}
/// <summary>
/// Base class for all contexts passed into hosted instances.
/// </summary>
struct _actor_context
{
    virtual ~_actor_context() {}

    /// <summary>
    /// Recast context as 'T' which must be a derived class.
    /// </summary>
    template<typename T> const T& as() 
    { 
        return *dynamic_cast<T *>(this); 
    }
};
{% endhighlight %}

Despite the comment that `'T'` must be a derived class of `_actor_context,`the function template `as()` can be invoked with *any* type, and the compiler will happily accept the code. That is to say, the comment neither prevents a programmer from writing incorrect code, nor does it enable compiler in enforcing the type-constraint required by the function template:

{% highlight cpp %}
struct X : _actor_context{}; //i.e X is a derived class of _actor_context
struct Y {};                 //i.e Y is a NOT derived class of _actor_context

_actor_context & ctx = get_context();
X const & x = ctx.as<X>(); //ok - and compiler should allow this
Y const & y = ctx.as<Y>(); //ok - but compiler must NOT allow this.
{% endhighlight %}

The fact that `Y` is not a derived class of `_actor_context` must be enough hint for the compiler to reject the second call to `as()` but the current implementation of the function template doesn't help the compiler in detecting the error ([demo][faulty-as]). So we must find a way to enforce the type-constraint in the code itself, as opposed to in the comment.

C# provides [where contexual keyword][where] which is used to enforce type-constraint on generic type parameters. C++ doesn't have such keyword. However, a little template-metaprogramming can do the job; more specifically C++11's two meta-functions viz [std::enable_if][enable] and [std::is_base_of][base] are all that we need. Here is an improved version of `as()`:

{% highlight cpp %}
template<typename T>
typename std::enable_if<std::is_base_of<_actor_context,T>::value,T>::type const & as() 
{ 
    return *dynamic_cast<T *>(this); 
}

//usage
X const & x = ctx.as<X>(); //ok 
Y const & y = ctx.as<Y>(); //error - Y isn't derived class of _actor_context, after all.
{% endhighlight %}

As shown above, calling `as()` with an incorrect type argument will now give *compilation* error (as opposed to *runtime* error) and that is a huge improvement, in my opinion, as we get to know the bug as early as possible ([demo][nonfaulty-as]). Agree that the return type now looks ugly and cumbersome but then that too can be fixed with the help of a tiny wrapper class template such as this:

{% highlight cpp %}
template<typename T>
struct if_derived
{
   typedef typename std::enable_if<std::is_base_of<_actor_context,T>::value,T>::type type;
};

//now how does it look like?
template<typename T>
typename if_derived<T>::type const & as() 
{
  return *dynamic_cast<T *>(this); 
}
{% endhighlight %}

That looks better, and it certainly increases the readability, as it doesn't *require* the programmer to comment the code with *"T must be a derived class"*, because now the return type is a documentation in itself and a comment doesn't add much to that. Now if a programmer violates the type-constraint, the compiler will immediately bite him, making him realize what needs to be fixed!

Also note that `if_derived` class template not only increases the readability of the code, GCC provides better diagnostics in this case. See the [demo][better-as] yourself!

###Choosing alternatives

We've fixed the return type; now have a look at the body of the function:

{% highlight cpp %}
template<typename T>
typename if_derived<T>::type const & as() 
{
  return *dynamic_cast<T *>(this);  //it is taken from the source.
}
{% endhighlight %}

Any serious problem with the body? What if the runtime type of `this` is not `T*`? What would the expression `dynamic_cast<T *>(this)` evaluate to? 

Well, in that case, it will evaluate to `nullptr`, and dereferencing a `nullptr` invokes undefined behavior, which means *anything* could happen; neither the language specification, nor the compiler gives any gaurantee about the code once it enters into the region of undefined behaviour. Undefined-behaviours are worst kind of bugs in C and C++ code. A programmer must strive to avoid them at any cost. So the syntax `*dynamic_cast<T *>(this)` is dangerious, as at runtime it may attempt to dereference a nullptr. 

Anyway, do we have any alternative (yet equivalent) syntax? Yes, we do. I would suggest this:

{% highlight cpp %}
template<typename T>
typename if_derived<T>::type const & as() 
{
  return dynamic_cast<T const &>(*this);  //improved!
}
{% endhighlight %}

That is, instead of casting `this` to `T*`, why not cast `*this` to `T const&`. After all, the function returns `T const&`. In this alternative syntax, we dereference the source, not the result, thereby avoiding the danger of dereferencing a nullptr, as `this` can never be nullptr.

Now what if the runtime type of `this` is not `T*`? Well in that case the expression `dynamic_cast<T const&>(*this)` would throw [std::bad_cast] exception ([demo][bad-cast]) which is a much better situation, as exceptions are well-defined, and can be caught with `try-catch` and dealt with accordingly.

###Return value vs. exception?

some code

###Lack of friendly logging api
