---
layout: post
title: Review of Casablanca
tags: [c++, template, microsoft, casablanca]
---

- [Introduction](#introduction)
- [Enforcing type-constraint](#enforcing_typeconstraint)
- [Choosing alternatives](#choosing_alternatives)
- [Immutability Is Important](#immutability_is_important)
- [Less is more](#less_is_more)
- [Dont encapsulate partially](#dont_encapsulate_partially)
- [Lack of friendly logging api](#lack_of_friendly_logging_api)
- [And the rest](#and_the_rest)

[casablanca]:http://msdn.microsoft.com/en-us/devlabs/casablanca.aspx
[where]:http://msdn.microsoft.com/en-us/library/d5x73970(v=vs.110).aspx
[enable]:http://en.cppreference.com/w/cpp/types/enable_if
[base]:http://en.cppreference.com/w/cpp/types/is_base_of
[faulty-as]:http://ideone.com/fZTtd
[nonfaulty-as]:http://ideone.com/J3Aqi
[better-as]:http://ideone.com/CX2FM
[bad-cast]:http://ideone.com/gn3Y1
[defensive-programming]:http://en.wikipedia.org/wiki/Defensive_programming

###Introduction

[Casablanca][casablanca] is a codename for the cloud-based client-server communication SDK developed by Microsoft. It is currently in beta phase which I downloaded just to play around with. Before anything else, I first started looking at the class templates and other header files. That is something I often do - reading code written by others. Anyway, soon I felt like commenting on the implementation of few classes and functions, especially the template code. So I thought why not write a blog so that others (maybe the ones who wrote the code) could also review my review? Sounds like a good idea.

Obviously, it isn't possible to comment on the overall design of the SDK, as I don't have access to the full code-base and the design documentation. However, it is certainly possible to comment on some pieces, the small yet important pieces of the code which comes with the SDK. In this blog, I've attempted to list few ideas which can help improving the code in many ways.

###Enforcing type-constraint

**Why do we need type-constraint? ** Well, because every bug has its life. There are bugs with shorter lifespan because the programmer can find them easily; such bugs get fixed in the developement phase itself. There are others with a bit longer lifespan which get fixed after they are discovered in the testing phase and reported as bugs to be fixed by the developers. However, there are others with *longest* lifespan, discovered in production phase, or *worst* after the product is released!

Therefore, the shorter the lifespan of bugs, the better is the situation, as it not only avoids lots of headache, it also increases the efficiency of developers and testers, and the quality of the product. So type-constraint lets compiler find error in your code at *compile-time* itself, making the lifespan of bugs *shortest*, because they get fixed immediately. Enforcing type-constraint in template code is one way to [program defensively][defensive-programming]. Moreover, it is a *best* documentation in itself, as the template code *itself* speaks what type(s) it will acccept as type argument(s)!

So here is a class which has a member function template which is to be instantiated with ***only*** derived class type(s), **but no such type-constraint is enforced on its implementation**:
{% highlight cpp %}
///code-snippet taken from context.h - Sarfaraz Nawaz

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

Despite the comment that `'T'` must be a derived class of `_actor_context,`the function template `as()` can be invoked with *any* type, and the compiler will happily accept the code. That is to say, the comment neither prevents a programmer from writing an incorrect code, nor does it enable the compiler in enforcing the type-constraint *required* by the function template:

{% highlight cpp %}
struct X : _actor_context{}; //i.e X is a derived class of _actor_context
struct Y {};                 //i.e Y is a NOT derived class of _actor_context

_actor_context & ctx = get_context();
X const & x = ctx.as<X>(); //ok - and that is fine.
Y const & y = ctx.as<Y>(); //ok - but the compiler must NOT allow this code!
{% endhighlight %}

The fact that `Y` is not a derived class of `_actor_context` must be a *strong* hint for the compiler to reject the second call to `as()` but the current implementation of the function template `as()` doesn't help the compiler in detecting this *obvious* error ([demo][faulty-as]). So we must find a way to enforce the type-constraint in the code itself, as opposed to in the comment!

C# provides [where contexual keyword][where] which is used to enforce type-constraint on generic type parameters. C++ doesn't have such keyword. However, a little template-metaprogramming can do the job; specifically C++11's two meta-functions viz [std::enable_if][enable] and [std::is_base_of][base] are all that we need. Here is an improved version of `as()`:

{% highlight cpp %}
template<typename T> //I know it is ugly; we'll come to that soon!
typename std::enable_if<std::is_base_of<_actor_context,T>::value,T>::type const & as() 
{ 
    return *dynamic_cast<T *>(this); 
}

//usage
X const & x = ctx.as<X>(); //ok 
Y const & y = ctx.as<Y>(); //error - Y isn't derived class of _actor_context, after all.
{% endhighlight %}

As shown above, calling `as()` with an incorrect type argument will now give *compilation* error, as opposed to *runtime* error which may take longer to discover. So getting compile-time error, as opposed to runtime-error, is a huge improvement, in my opinion, as we get to know the bug as early as possible ([demo][nonfaulty-as]). Agree that the error message is not that clear (in GCC), but it does tell us the line number at least. Also, the return type now looks ugly and cumbersome but then that too can be fixed with the help of a tiny wrapper class template such as this:

{% highlight cpp %}
//a tiny wrapper class template
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

That looks better, and it certainly increases the readability, as it doesn't *require* the programmer to comment the code with *"T must be a derived class"*, because now the return-type says that pretty much clearly, and the resulting code is a documentation in itself and a comment doesn't add much to that. Now if a programmer violates the type-constraint, the compiler will immediately bite him, making him realize what needs to be fixed!

Also note that the class template `if_derived<>` not only increases the readability of the code, GCC provides better diagnostics in this case. See the [demo][better-as] yourself. I hope MSVC 2012 too will provide better diagnostics. :-)

###Choosing alternatives

We've fixed the issues of type-constraint by modifying the return type; now have a look at the body of the function:

{% highlight cpp %}
template<typename T>
typename if_derived<T>::type const & as() 
{
  return *dynamic_cast<T *>(this);  //body : it is taken from the source.
}
{% endhighlight %}

Any serious problem with the body? The body down casts `this` to `T*` and then dereference the result before returning it. That is it. 

Now, what if the runtime type of `this` is not `T*`? What would the expression `dynamic_cast<T*>(this)` evaluate to? Well, in that case, it will evaluate to `nullptr`, and dereferencing a `nullptr` invokes undefined behavior which means *anything* could happen; neither the language specification, nor the compiler gives any gaurantee about the code once it enters into the region of undefined behaviour. Undefined-behaviours are the worst kind of bugs in C and C++ code. A programmer must strive to avoid them at any cost. So the syntax `*dynamic_cast<T *>(this)` is potentially dangerious, as at runtime it *may* attempt to dereference a nullptr. 

So, do we have any alternative (yet equivalent) syntax that avoids the above problem? Yes, we do. I would suggest this:

{% highlight cpp %}
template<typename T>
typename if_derived<T>::type const & as() 
{
  return dynamic_cast<T const &>(*this);  //improved!
}
{% endhighlight %}

That is, instead of dereferencing the result of `dynamic_cast`, why not dereference `this` itself? In other words, instead of casting `this` to `T*`, why not cast `*this` to `T const&`. After all, the function returns `T const&` - that is what we need eventually. In this alternative syntax, we dereference the source, not the result, thereby avoiding the danger of dereferencing a nullptr, as `this` can never be nullptr.

Now lets face the same question : what if the runtime type of `this` is not `T*`? Well in that case the expression `dynamic_cast<T const&>(*this)` would throw `std::bad_cast` exception ([demo][bad-cast]) which is a *much better* situation, as exceptions are C++ feature and can be caught with `try-catch` and dealt with accordingly. Moreover, your program remains well-defined in all cases, as it doesn't enter into the region of undefined-behaviour, anymore!

### Immutability Is Important

[herb-const-correctness]:http://www.gotw.ca/gotw/006.htm

Immutability is good and important. That is the central notion of the functional programming languages, with which I totally agree. In C++, there is a very *special* aspect of immutability, which is implemented in terms of const member function and const-correctness. So, in C++, const-correctness is as important as immutability. Now lets analyse this *original* code:

{% highlight cpp%}
///original code
template<typename T> 
const T& as() 
{ 
  return *dynamic_cast<T *>(this); 
}
{% endhighlight %}

It does two things: it casts down `this` to `T*` and *modifies the const-ness* of the object pointed to by `this` while returning it. The modification of *const-ness* results in many issues, which are best shown in the following code and the comments:

{% highlight cpp%}
struct X : _actor_context
{
   void  f() {}        //non-const member function
   void cf() const {}  //const member function
};

void do_something(_actor_context & nonConstCtx) //non-const context reference!
{
   auto & x1 = nonConstCtx.as<X>(); //ok
   x1.cf(); //ok
   x1.f();  //error : oops, the type of `x1` is inferred to be `X const &` 
            //and a non-const member cannot be invoked on const object.
}
{% endhighlight %}

But wait, `nonConstCtx` is a non-const context, which means we should be *able* to call non-const member function referred to by the reference `nonConstCtx`, even after the dynamic_cast:

{% highlight cpp%}
auto & x2 = dynamic_cast<X&>(nonConstCtx); //this cast is perfectly fine!
x2.f();  //ok, as we should be able to do that. the type of x2 is inferred to be `X &` now.
x2.cf(); //ok, as const member function can be invoked on non-const object!
{% endhighlight %}

Therefore, that means `as()` conversion function does more than what it is asked to do : it modifies the const-ness, in addition to the down-cast!

Well in my opinion, we should not modify the const-ness of the object, therefore `as()` should be implemented in such a way to ensure the following behaviors:

- if ctx is a non-const reference (i.e it is *\_actor\_context &*), then it should be down-casted to non-const reference to T (i.e it should return *T&*).

- if ctx is a const reference (i.e it is *\_actor\_context const &*), then it should be down-casted to const reference to T (i.e it should return *T const &*).

To ensure these behaviors, we need to define two overloaded `as()` member functions as follows:

{% highlight cpp%}
template<typename T> 
T& as() //non-const member function which returns non-const reference to T
{ 
  return *dynamic_cast<T *>(this); 
}
template<typename T> 
T const & as() const //const member function which returns const reference to T
{ 
  return *dynamic_cast<T const *>(this); 
}
{% endhighlight %}

Merging all ideas discussed so far, we have these:

{% highlight cpp%}
template<typename T> 
typename if_derived<T>::type & as() //non-const member function.
{ 
  return dynamic_cast<T&>(*this);  //changed here also!
}
template<typename T> 
typename if_derived<T>::type const & as() const //const member function.
{ 
  return dynamic_cast<T const &>(*this);  //changed here also!
} 
{% endhighlight %}

Once we have these overloads, we will have consistent code:

{% highlight cpp%}
void do_something(_actor_context & nonConstCtx,    //non-const reference
                  _actor_context const & constCtx) //const reference
{
   //Working with non-const reference
   auto & x1 = nonConstCtx.as<X>(); //it invokes non-const member function
                                    //which returns non-const reference to X!
                                    //x1 is inferred to be X &
   x1.cf(); //ok - const member function can be invoked on const object :-)
   x1.f();  //ok

   //Working with const reference 
   auto & x2 = constCtx.as<X>();   //it invokes const member function
                                   //which returns const reference to X!
                                   //x2 is inferred to be X const &
   x2.cf(); //ok
   x2.f();  //error, which is consistent behavior because constCtx
            //is a const reference and the compiler must stop us 
            //from invoking non-const member function on const object i.e the casted-down object from constCtx.
}
{% endhighlight %}

And finally, I would redirect this discussion to Herb Sutter's article on [const-correctness][herb-const-correctness] which discusses this topic in great detail.

###Less is more

[less-is-more]:http://www.stevemcconnell.com/articles/art06.htm

[Sometime less is more][less-is-more] : concise code if it doesn't harm readability should always be preferred, precisely because it is concise and, well, readable.

It is pretty much common in languages which supports function overloading that we often implement one overload in terms of others. Something like this:

{% highlight cpp %}
bool fun(A a);

//an overloaded function
bool fun(B b) 
{ 
  /* some code might be here */ 

  return fun(b.avalue); //call the other overload for the rest of the work.
}
{% endhighlight %}

Such implementations are almost always elegant. Now have a look at this code taken from `uri_tokenizer.h`:

{% highlight cpp %}
///code-snippet taken from uri_tokenizer.h - Sarfaraz Nawaz

template<typename _char_t, typename _t>
bool bind(const std::basic_string<_char_t> &text, _t &ref);

template<typename _char_t, typename _t>
bool bind(const std::basic_string<_char_t> &text, optional<_t> &opt)
{ 
   //code omitted for brevity!
    
    if( !bind(text, opt.value) )
    {
       opt.has_value = false;
       return false;
    }
    opt.has_value = true;
    return true;
}
{% endhighlight %}

This function is implemented in terms of the other overload of `bind()` but the verbosity appears to hide that fact from the reader. I would rather prefer to make that fact obvious by refactoring the above code as:

{% highlight cpp %}
template<typename _char_t, typename _t>
bool bind(const std::basic_string<_char_t> &text, optional<_t> &opt)
{ 
   //code omitted for brevity!
    
  opt.has_value = bind(text, opt.value); //invoke the other overload for the rest of the work
  return opt.has_value; //returns what other overload returned!
}
{% endhighlight %}

That, in my opinion, increases readability and it is evidently concise. :-)

### Dont encapsulate partially

[rule-of-three]:http://stackoverflow.com/questions/4172722/what-is-the-rule-of-three
[raii]:http://en.wikipedia.org/wiki/Resource_acquisition_is_initialization
[deleted-functions]:http://blogs.msdn.com/b/vcblog/archive/2011/09/12/10209291.aspx
[noncopyable]:http://www.boost.org/doc/libs/1_51_0/boost/noncopyable.hpp

I don't understand why this class is implemented in this way:

{% highlight cpp %}
///code-snippet taken from registry.h - Sarfaraz Nawaz

template<typename Element>
struct registry_node
{
    registry_node() : m_delegate(nullptr) {}

    ~registry_node()
    {
        for (auto iter = m_children.begin(); iter != m_children.end(); iter++)
        { 
            delete iter->second; 
        }
    }

    Element m_delegate;
    std::map<std::string, registry_node *> m_children;
};
{% endhighlight %}

Is it a resource-managing class (like Standard containers)? Means, is it owning the resource which it keeps references (or pointers) to? That doesn't seem to be the case, but it deletes the pointers anyway as if it owns them! Why does it take the responsibility to delete the pointers when it has not taken the responsibility for their creations and copying? 

To cut the long story short, it violates [the rule of three][rule-of-three] - devised on a basic principle called [RAII][raii] which is the single most important idiom in C++.

So the fix is : either implement copy-semantics for the class or **disable** them if copying doesn't make sense for it:

{% highlight cpp %}
//disable copy-semantics by deriving it from noncopyable!
template<typename Element>
struct registry_node : noncopyable //now derives from noncopyable :-)
{
    //same as before 
};
{% endhighlight %}

Now if any piece of code attempts to make copy of registry_node instance, even accidently, the compiler will reject that code, letting the programmer know the cause of the rejection. Anyway the base class *noncopyable* has trivial implementation:

{% highlight cpp %}
class noncopyable 
{
    protected:
        noncopyable() {}
    private:
        noncopyable(noncopyable const &) = delete;
        noncopyable& noncopyable=(noncopyable const &) = delete;
};
{% endhighlight %}

But sad, MSVC 2012 [doesn't support deleted functions][deleted-functions] currently, so we have to be happy with this old trick:

{% highlight cpp %}
class noncopyable 
{
    protected:
        noncopyable() {}
    private:
        noncopyable(noncopyable const &);  //declared but not defined!
        noncopyable& noncopyable=(noncopyable const &); //declared but not defined!
};
{% endhighlight %}

Good thing is that this class can be used as base class for many other classes for which copy-semantics don't make sense. Therefore, I think it is good to have this class in any C++ library ([like Boost has this!][noncopyable]).

###Lack of friendly logging api

[templog]:http://www.templog.org/
[macro]:http://msdn.microsoft.com/en-us/library/ms177415(v=vs.110).aspx
[macro-bad]:http://stackoverflow.com/questions/4767667/c-enum-vs-const-vs-define

I was a bit uncomfortable seeing the logging code in the project. One has to write *minimum* three lines of code to log just one message:

{% highlight cpp%}
///code-snippet taken from agnt_tcp_listen.h - Sarfaraz Nawaz

std::stringstream builder;
builder << "  Hosting instance of '" << type_name.substr(6) << "' at " << m_uri;
log::post(LOG_INFO, builder.str());
{% endhighlight %}

Such triplets has been repeated throughout the project. So I would say, it is very unfriendly (or rather scary) way to log messages, and the resulting code looks unnecessarily verbose and huge. On the top of it, if one wants to disable logs with certain *severity*, the third line *might* be optimized away depending on how `log::post` is implemented, but the first two lines will still remain in the code to be executed, wasting CPU cycles. Ideally, a logging API shouldn't consume CPU cycles for all logs with disabled severity. [templog][templog] is one such attempt in that direction.

At first it doesn't seem to be a problem with the logging API itself; after all `log::post()` is just one line which cannot be reduced to zero. It appears to be a problem with the formatting of the message. So one may think of the following improvement (or something along this line):

{% highlight cpp%}
log::post(LOG_INFO, format(" Hosting instance of {0} at {1}", type_name.substr(6), m_uri));
{% endhighlight %}

But then why even `format`? Why not this instead:

{% highlight cpp%}
log::post(LOG_INFO, " Hosting instance of {0} at {1}", type_name.substr(6), m_uri);
{% endhighlight %}

I would prefer this. :-)

Now if one wishes to log line, filename, and function name, then [macro][macro] would make that easy:

{% highlight cpp%}
#define T_INFO(fmtstring, ...)  log::post(LOG_INFO, "{0}\n\n{1}, {2}, {3}", \
                                format(fmtstring, __VA_ARGS__),             \
                                __FILE__, __FUNCTION__, __LINE__)
{% endhighlight %}

Once we've such macro, logging becomes so easy:

{% highlight cpp%}
T_INFO(" Hosting instance of {0} at {1}", type_name.substr(6), m_uri);
{% endhighlight %}

Note that the name of the macro itself says what *severity* it is. With macro, we have also such opportunities:

{% highlight cpp%}
#ifdef NDEBUG
   #define T_VERBOSE(fmtstring, ...)  log::post(LOG_VERBOSE, etc )
#else
   #define T_VERBOSE(fmtstring, ...)  do {} while(false) //no-op instruction!
#endif
{% endhighlight %}

Well, I do agree that [macros are bad][macro-bad], as they don't respect scope, but they are not always bad, not atleast here. :-)

###And the rest

[rest]:http://en.wikipedia.org/wiki/Representational_state_transfer

Well it is not about [REST][rest] which is supported by [Casablanca][casablanca], rather it is about the rest. 

Alright, this blog is already long, and I don't wish to make it any longer by adding more stuffs to it, though there are some topics still popping up in my mind, such as *return value vs. exception*, *cocktail of different and inconsistent naming conventions* (see `procdir.h` for example), and many others.