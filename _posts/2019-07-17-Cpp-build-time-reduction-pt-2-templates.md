---
layout: post
title:  "Reduction of C++ project build time - Templates"
date:   2019-07-16 21:00:00 +0000
tags: [ Cpp, build, time, C++, C, templates, template, reduce, optimization]
---

Hello!
I hope you were not starving for some build time reduction tips. It's been looong two weeks. In case you've missed out on the last post, it can be found [here]({% post_url 2019-07-03-Cpp-build-time-reduction-pt-1-introduction %}). Give it a go!

We now have all the necessary tools to deal with templates. Before we take them on, let's take a step back and understand what we are dealing with.

- Template entities (either class templates or functions) can be thought of as "generators" of entities. It would be wrong to assume that an ordinary function is equal to template definition. Instantiated entity is closer to the plain function than the template definition that was used to create it.

- This post will deal exclusively with function/method templates - class templates costs cannot really be avoided in cases where a full type needs to be known. 

- Instantiated functions are implicitly marked with *inline* specifier. Note that this does not necessarily force compiler to inline all calls to instantiated functions. *inline* simply allows for multiple occurences of the same symbol to exist across many translation units during linking stage (it makes the emitted symbol *weak*).

## "And so you were born" - template instantiation methods

While we are at it, let's look into two methods of instantiating templates. These are *implicit* and *explicit* instantiations. Based on my limited programming experience, most of the instantiations are implicit.. which is a shame, because 
it is easy to get carried away with template madness this way.

### Implicit instantiation
Implicit instantiation takes place whenever the template with given set of template argument is referenced and the full definition of type/function/method has to be known. 

```cpp
// .h file
template<typename T>
size_t mySizeof() {
    return sizeof(T);
}

size_t foo() {
    return mySizeof<uint32_t>(); // implicit instantiation takes place here.
}
```
This also applies to types. Here's a riddle for you. When does implicit instantiation take place in the following piece of code?
```cpp
// .h file
template<typename T>
struct Wrapper{
    T value;
};

uint32_t foo(Wrapper<uint32_t>* val);
uint32_t foo(Wrapper<uint32_t> second_val);
```
I would not ask if it was not a trick question. The instantiation will take place for `Wrapper<int> second_val` version due to the fact that - in pointer case - compiler does not need to know anything about the type `Wrapper<int>` to emit a valid function call for this type. It also knows it\`s size (pointer size depends only on the architecture). 
However, in case of function *definitions* there might be an implicit instantiation even in case of pointers. Consider following example:
```cpp
uint32_t foo(wrapper<uint32_t>* val)
{
    return val->value;// member offset from pointed-to-address has to be known here. 
    // Implicit instantiation of type wrapper<uint32_t> takes place.
}

uint32_t foo(wrapper<uint32_t> val)
{
    // Implicit instantiation has taken place earlier on.
    // There's nothing to do from compiler perspective.
    return val.value;
}
```

### Explicit instantiation
Explicit instantiation is - well, how do I put it - explicit.
It can be achieved in a following way.
```cpp
template class wrapper<uint64_t>;
```
This might not be of much use for your day-to-day work. It will come in handy in just a second.

## What is wrong with templates, again?
Templates are wonderful, but in my honest opinion they can impact the build time when used carelessly. Each translation unit that makes use of given template must have a full access to it\`s definition in order to instantiate it (either implicitly or explicitly). Thus, there is a great opportunity for doing redundant work - `mySizeof` might\`ve been cheap to instantiate, but in case of more sophisticated templates it can take quite some time. Since the instantiated functions are weak symbols, N - 1 out of N instantiations for given argument set that's instantiated N times in your whole project should get removed by any decent linker. Congratulations, you have just spent 3 minutes doing nothing but template instantiations. 

The previous paragraph has one blatant lie in it. I wonder whether you've spotted it.

### The cake...
In some cases the compiler does not need to know the template definition in order to use it. To be precise, the instantiation step can be skipped for N - 1 out of N instantiations for given template argument set. Let's look at how the "vanilla" (non-template) functions are used (usually):
1. There is a function **declaration** in header file.
```cpp
//header.h file
int foo(int bar);
```

2. The function **definition** resides in implementation file.
```cpp
//impl.cpp file
#include "header.h"
int foo(int bar){
    return bar;
}
```

3. A translation unit that wants to call the `foo(int)` function includes a header file with it\`s **declaration**.
```cpp
// other.cpp file
#include "header.h"
int baz(){
    return foo(5); // The definition nor address of foo is not known at this point.
                   // Hence, the call is saved for resolution at link-time.
}
```

4. The call is resolved at link-time - the address of `foo(int)` is placed at "call" site.

On the other side of the spectrum, a "call" to function that's a product of implicit template instantiation goes as follows:

1. There is a template **definition** in header file.
```cpp
//header_with_template.h
template<typename T>
T identity(T val) {
    return val;
}
```
*Don't give me slack for not using std::forward, please!*

2. There is a translation unit that wants to use the template.
```cpp
// implementation.cpp
#include "header_with_template.h"
int baz() {
    return identity(5); // identity template is implicitly instantiated
                        // at this point for int argument.
}
```
Definition of `identity<int>` is present in this translation unit **and** any other translation unit that wants to use `identity`. 

3. There is nothing to resolve at link time - moreover, since the definition is available at call site it can be inlined by compiler.

The problem lies within redundant instantiations. To deal with that, let's prevent the compiler from performing it. It cannot instantiate a template if it does not know it's full definition. Going back to the last example..

1. There is a function template **declaration** in header file.
```cpp
// mytemplate.h
template<typename T>
T identity(T val);
```

2. There is a function template **definition** in implementation file along with explicit instantiations.
```cpp
// mytemplate.cpp
#include "mytemplate.h"
template<typename T>
T identity(T val) {
    return val;
}
template // explicit template instantiation.
int identity<int>(int val);
```

3. Finally, an end user uses only .h file - it cannot instantiate the function template, so no work is performed. The symbol is left for linker resolution.
```cpp
// mytemplateuser.cpp
#include "mytemplate.h"
int baz() {
    return identity(5);
}
```

Hey.. Doesn't that remind you of an "ordinary function" approach?

### ... is a lie. 
There are few caveats with this approach though..

1. The call to instantiated function is no longer a candidate for inlining, since the function body is not known in given translation unit. 
2. Explicit instantiation is performed beforehand - types of arguments have to be known...
3. And if you try to fight it by instantiating all known usages of given template in one translation unit, the same TU can quickly become bloated by dependencies that it does not really need. 

Last point is very important - I would advise against compensating for minor losses in multiple translation units by paying a potentially huge cost in one TU. I have once tried to disable implicit instantiations for some template which had 10 or so different argument types. It quickly turned out to not be worth it. 

I must mention though that the first of the listed drawbacks can be remedied by LTCG. I hope you recall that LTCG can help immensely with inlining cross-TU function calls. 
This approach is not very helpful for library authors though - in case function template is exposed as a part of library interface, there is no possibility of "hiding" the template implementation and improving the build time, because then the library user won't be able to instantiate the template with their own list of arguments.

## Extern templates to the rescue

C++11 introduced a very helpful tool - extern templates.
It allows us to skip any implicit instantiation of a template with specific argument set. Compiler can then assume that the instantiated function is available in other translation unit and leave the symbol for link-time resolution.
Consider our favorite (yet sub-optimal) template:
```cpp
// header.h 
template<typename T>
T identity(T value) {
    return value;
}
```
Declaring an explicit template instantiation can be achieved by specifying template name and argument set before it's usage with a keyword (you've guessed correctly) `extern` in front. Hence, to prevent instantiation in some translation unit (and leave it up for linker to provide the function definition), the following construct should be used:
```cpp
extern template
int identity(int); // Skip any instantiations of this template.
bool biz(){
    return identity(33) == 33; // The implicit instantiation won't take place.
}
```
What would be the use scenario for such a construct?
Let's say that `identity` is used in two translation units..
```cpp
// first_user.cpp
#include "header.h"
int bar() {
    return identity(42);
}
```
```cpp
// second_user.cpp
#include "header.h"
int baz(int val) {
    return identity(val);
}
```
The function template is instantiated implicitly in both cases for `int`. However, the instantiation could be skipped in one of those (let's pick `second_user.cpp`).
Thus, `second_user.cpp` contents should be changed in following fashion in order to make use of extern templates.
```cpp
// second_user.cpp
#include "header.h"
extern template
int identity(int);
int baz(int val) {
    return identity(val);
}
```

Remember my rant about library authors in previous paragraph? Well, `extern template` can be used in code that's not really yours. It has the advantage over removing template definition from header in that the caller is responsible for managing the symbol (it will be left for linker resolution) and not the callee (who has to make the symbol inaccessible and force linker resolution in case of template definition erasure). Hence if you notice that this super-cool-yet-super-slow-to-instantiate-template-from-library is slowing down your build and the instantiations are repetitive across multiple translation units, `extern template` might be a good answer.
If you are interested in more details, give the following materials a look:
- [Andre Haupt\`s blog post about extern templates](http://blog.bitwigglers.org/extern-templates/).
- [cppreference.com entry on class templates (with few words about extern templates)](https://en.cppreference.com/w/cpp/language/class_template).

### That's not a silver bullet!
There's an issue though - which translation unit will be responsible for instantiating the template? There has to be one. I haven't really found the answer to that question just yet, but the problem of instantiation responsibility is persistent across both definition erasure method and `extern template`. Moreover, what if the template is instantiated in header file that you happen to pull in? 
Yep, the code can get pretty ugly. But if that's the price to pay.. Personally I would be fine with that. Also, keep in mind that ability to inline the template is lost once again - only for the mighty and glorious LTCG to reclaim it again.

## Summary
To recap, today we went through template instantiation methods and the ways to avoid doing unnecesary work.
Two of the presented methods can be used in different circumstances:
- Template definition erasure is useful when there are a lot of callers for the instantiated function template, yet the possible domain of instantiation arguments is known beforehand (preferably most of them should be built-in types).
- `extern template` allows us to specify per-TU intantiation options. It might result in more boilerplate.

While both of them have their pros and cons (and different use cases, which I've just stated) I would lean towards using `extern template`. Definition erasure is a global technique which requires user attention in order to support new type. On the other side of the spectrum, `extern template` is local to the current TU - using it in TU A does not surpress implicit function template instantiation in other TUs, so if a colleague tries to instantiate your template in theirs TU with cool and fancy argument set, they can expect reduced build time performance and not linker resolution error.
