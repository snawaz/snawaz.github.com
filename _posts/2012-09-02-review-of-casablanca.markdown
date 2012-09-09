---
layout: post
title: Review of Casablanca
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
[defensive-programming]:http://en.wikipedia.org/wiki/Defensive_programming

###Introduction

[Casablanca][casa] is a codename for the cloud-based client-server communication SDK being developed by Microsoft. It is currently in beta phase which I downloaded just to play around with. Before anything else, I first started looking at the class templates and other header files. That is something I often do - reading code written by others. Anyway, soon I felt like commenting on the implementation of few classes and functions. So I thought why not write a blog so that others (maybe the ones who wrote the code) could also review my review? Sounds like a good idea.

Obviously, it isn't possible to comment on the overall design of the SDK, as I don't have access to the full code-base and the design documentation. However, it is certainly possible to comment on some pieces, the small yet important pieces of the code which comes with the SDK. In this blog, I've attempted to list few ideas which can help improving the code in many ways.

###Enforcing type-constraint

**Why do we need type-constraint? ** Every bug has its life. There are bugs with shorter lifespan because the programmer can find them easily; such bugs get fixed in the developement phase itself. They're others with a bit longer lifespan which get fixed after they're discovered in the testing phase and reported as bugs to be fixed by developers. However, they are others with longest lifespand, discovered in production phase, or worst after the product is released!

Therefore, shorter the lifespan of bugs, better is the situation, as it not only avoids lots of headache, it also increases the efficiency of developers and testers. So type-constraint lets compiler find error in your code at *compile-time* itself, making the lifespan of bugs *shortest*, because they get fixed immediately. Enforcing type-constraint in template code is one way to [program defensively][defensive-programming].

So here is a class which has a function template which is to be instantiated with ***only*** derived class type, **but no such type-constraint is enforced on its implementation**:
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

Despite the comment that `'T'` must be a derived class of `_actor_context,`the function template `as()` can be invoked with *any* type, and the compiler will happily accept the code. That is to say, the comment neither prevents a programmer from writing incorrect code, nor does it enable compiler in enforcing the type-constraint required by the function template:

{% highlight cpp %}
struct X : _actor_context{}; //i.e X is a derived class of _actor_context
struct Y {};                 //i.e Y is a NOT derived class of _actor_context

_actor_context & ctx = get_context();
X const & x = ctx.as<X>(); //ok - and that is fine.
Y const & y = ctx.as<Y>(); //ok - but the compiler must NOT allow this code!
{% endhighlight %}

The fact that `Y` is not a derived class of `_actor_context` must be enough hint for the compiler to reject the second call to `as()` but the current implementation of the function template `as()` doesn't help the compiler in detecting the error ([demo][faulty-as]). So we must find a way to enforce the type-constraint in the code itself, as opposed to in the comment!

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

Also note that the class template `if_derived<>` not only increases the readability of the code, GCC provides better diagnostics in this case. See the [demo][better-as] yourself!

###Choosing alternatives

We've fixed the issues of type-constraint by modifying the return type; now have a look at the body of the function:

{% highlight cpp %}
template<typename T>
typename if_derived<T>::type const & as() 
{
  return *dynamic_cast<T *>(this);  //body : it is taken from the source.
}
{% endhighlight %}

Any serious problem with the body? The body down casts `this` to `T*` and dereference the result and return it. That is it. 

Now, what if the runtime type of `this` is not `T*`? What would the expression `dynamic_cast<T*>(this)` evaluate to? Well, in that case, it will evaluate to `nullptr`, and dereferencing a `nullptr` invokes undefined behavior which means *anything* could happen; neither the language specification, nor the compiler gives any gaurantee about the code once it enters into the region of undefined behaviour. Undefined-behaviours are the worst kind of bugs in C and C++ code. A programmer must strive to avoid them at any cost. So the syntax `*dynamic_cast<T *>(this)` is dangerious, as at runtime it *may* attempt to dereference a nullptr. 

So, do we have any alternative (yet equivalent) syntax that avoids the above problem? Yes, we do. I would suggest this:

{% highlight cpp %}
template<typename T>
typename if_derived<T>::type const & as() 
{
  return dynamic_cast<T const &>(*this);  //improved!
}
{% endhighlight %}

That is, instead of dereferencing the result of `dynamic_cast`, why not dereference `this` first? In other words, instead of casting `this` to `T*`, why not cast `*this` to `T const&`. After all, the function returns `T const&` - that is what we need eventually. In this alternative syntax, we dereference the source, not the result, thereby avoiding the danger of dereferencing a nullptr, as `this` can never be nullptr.

Now lets face the same question : what if the runtime type of `this` is not `T*`? Well in that case the expression `dynamic_cast<T const&>(*this)` would throw `std::bad_cast` exception ([demo][bad-cast]) which is a *much better* situation, as exceptions are well-defined, and can be caught with `try-catch` and dealt with accordingly. The code doesn't enter into the region of undefined-behaviour, anymore!

###Less is more

[less-is-more]:http://www.stevemcconnell.com/articles/art06.htm

[Sometime less is more][less-is-more] : concise code if doesn't harm readability should always be preferred, precisely because it is, well, concise and readable.

It is pretty much common in languages which supports function overloading that we often implement one overload in terms of others. Something like this:

{% highlight cpp %}
bool fun(A a);

//overload function
bool fun(B b) 
{ 
  /* some code might be here */ 

  return f(b.avalue); //call the other overload for rest of the work
}
{% endhighlight %}

Such implementations are almost always elegant. Now have a look at this code:

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
    
  opt.has_value = bind(text, opt.value); //invoke the other overload for rest of the work
  return opt.has_value; //returns what other overloaded returned!
}
{% endhighlight %}

That, in my opinion, increases readability and it is evidently concise. :-)

### Dont encapsulate partially

To cut the long story short, I don't understand why this class is implemented in this way:

{% highlight cpp %}
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

Is it a resource-managing class (like Standard containers)? Means, is it owning the resource which it keeps references (or pointers) to? Why does it take the responsibility to delete the pointers when it has not taken the responsibility of their creations?

The issues with this class will surface when one want to make copy of instances of this class. If the copy is never done, is an argument to support this class, then it is better made sure by making it *noncopyable* as:

struct registry_node : noncopyable

RAII is the single most important notion of C++. Everything else is related.
Either encapsulate the functionality properly or don't encapsulate at all.

###Return value vs. exception?

some code

### Immutabilty Is Important (III)

Lets revisit `as()` function template:

//code

It does two things: it casts down `this` to `T*` and modifies the const-ness while returning it, as it returns `const` reference to the object, rather than non-const reference to the object. Returing const-reference reduces its usage!

###Lack of friendly logging api


###Inconsistent naming

{% highlight cpp %}
///code-snippet taken from procdir.h - Sarfaraz Nawaz

class RemoteDirectory : public ProcessDirectory
{
public:
     //note : some and comments are omitted for brevity!

    _DISPIMP void AddStorageUri(const std::string &uri);
    _DISPIMP virtual bool add_to_directory(const type_info &type, const std::string &address);
    _DISPIMP virtual bool remove_from_directory(const type_info &type, const std::string &address);
    _DISPIMP virtual bool Find(const type_info &type, std::vector<std::string> &locations);
    _DISPIMP virtual bool Find(const std::string &address, type_info &type);
};
{% endhighlight%}

That is a cocktail of different naming conventions, isn't it?

