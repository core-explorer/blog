---
layout: post
title:  "Debugging C++ is a UI nightmare"
date:   2025-01-19 11:11:11 +0100
categories: c++ debugging
---
If you are a maintainer or developer of a debugger, don't be offended, I'm not talking about you or your product.
I spend a lot of time debugging, and I know that one can use these tools to solve problems effectively.
In fact, I spend so much time debugging that I decided to write [my own debugger](https://core-explorer.github.com/core-explorer). Like every project worth doing, it turned out to be harder than expected. I don't want to ramble, I will restrict myself to elaborating three pain points:
1. Entities with names
2. Entities without names
3. Names without entities

## Entities with names ##

C++ produces horrible names for types, functions, and variables (independently of the naming ability of individual developers).
Here is a function that appears in most of the stack traces of my webserver.

{% highlight C++ %}
boost::beast::detail::asio_handler_invoke<boost::asio::detail::binder2<boost::beast::basic_stream<boost::asio::ip::tcp, 
boost::asio::any_io_executor, boost::beast::unlimited_rate_policy>::ops::transfer_op<true, boost::asio::mutable_buffer, 
boost::asio::detail::composed_op<boost::beast::http::detail::read_some_op<boost::beast::basic_stream<boost::asio::ip::tcp, 
boost::asio::any_io_executor, boost::beast::unlimited_rate_policy>, boost::beast::basic_flat_buffer<std::allocator<char> >, true>, 
boost::asio::detail::composed_work<void(boost::asio::any_io_executor)>, 
boost::asio::detail::composed_op<boost::beast::http::detail::read_op<boost::beast::basic_stream<boost::asio::ip::tcp, 
boost::asio::any_io_executor, boost::beast::unlimited_rate_policy>, boost::beast::basic_flat_buffer<std::allocator<char> >, true, 
boost::beast::http::detail::parser_is_done>, boost::asio::detail::composed_work<void(boost::asio::any_io_executor)>, 
boost::beast::detail::bind_front_wrapper<void (http_session::*)(boost::system::error_code, long unsigned int), 
std::shared_ptr<http_session> >, void(boost::system::error_code, long unsigned int)>, void(boost::system::error_code, 
long unsigned int)> >, boost::system::error_code, long unsigned int>&>(binder2<boost::beast::basic_stream<boost::asio::ip::tcp, 
boost::asio::any_io_executor, boost::beast::unlimited_rate_policy>::ops::transfer_op<true, boost::asio::mutable_buffer, 
boost::asio::detail::composed_op<boost::beast::http::detail::read_some_op<boost::beast::basic_stream<boost::asio::ip::tcp, 
boost::asio::any_io_executor, boost::beast::unlimited_rate_policy>, boost::beast::basic_flat_buffer<std::allocator<char> >, true>, 
boost::asio::detail::composed_work<void(boost::asio::any_io_executor)>, 
boost::asio::detail::composed_op<boost::beast::http::detail::read_op<boost::beast::basic_stream<boost::asio::ip::tcp, 
boost::asio::any_io_executor, boost::beast::unlimited_rate_policy>, boost::beast::basic_flat_buffer<std::allocator<char> >, true, 
boost::beast::http::detail::parser_is_done>, boost::asio::detail::composed_work<void(boost::asio::any_io_executor)>, 
boost::beast::detail::bind_front_wrapper<void (http_session::*)(boost::system::error_code, long unsigned int), 
std::shared_ptr<http_session> >, void(boost::system::error_code, long unsigned int)>, void(boost::system::error_code, 
long unsigned int)> >, boost::system::error_code, long unsigned int>& f, bind_front_wrapper<void (http_session::*)
(boost::system::error_code, long unsigned int), std::shared_ptr<http_session> >* op)
{% endhighlight %}

And there are ten more that look just like it.

How can my debugger present this in a human readable way? The common approaches are:
1. This is what you need, so this is what you get. Deal with it (gdb, lldb).
2. This won't fit into an 80 character limit, so I'll shorten it for you (GNU binutils readelf).
`boost::beast::de[...]`
3. I will contract all template arguments for you (kcachegrind)
`boost::beast::detail::asio_handler_invoke<...>`

And for a generic debugger (i.e. handling multiple languages) without extensive logic just for C++, that is the best you can hope for:
1. present the long name
2. shorten the long name without finesse
3. shorten the name with some effort

In my opinion, option 3 looks like the lesser evil here, except:
* It requires parsing the C++ code in the demangled function signature. C++ parsing is hard and error prone, and _this code has already been parsed successfully by the compiler_.
* This is `invoke`. The actual meat is in the template arguments, telling us _what_ is supposed to be invoked.
Lets show the first template argument: `boost::asio::detail::binder2`. Okay, so not that. What about the others? `compose_op`, `compose_work`, `bind_front_wrapper`. There is no way a generic tool can pick out the signal from the noise in this template signature.

Before you say "dude, if you don't like the complexity, don't use boost::beast, use something simpler": 

I am writing a debugger, it needs to deal with existing code. It cannot react with  
`Lol, your code sucks. Thx, bye.`

By the way, this is purely a debugging problem. These long template signatures do not appear in the source code, the source looks like this:

{% highlight C++ %}
http::async_read(
		    stream_,
		    buffer_,
		    *parser_,
		    beast::bind_front_handler(
		        &http_session::on_read,
		        shared_from_this()));

{% endhighlight %}
Obviously the source also uses `using namespace boost::beast`.

These are the functions I plan to implement to make this more manageable:
1. I will not try to re-parse compiler-generated C++ types. Instead I will rely on debug information to reconstruct these names: synthesizing C++ is a lot easier than analyzing it.
2. User-controllable contraction of template arguments, with sensible defaults, if that is possible at all.
3. User-controllable omission of namespace prefixes with sensible defaults. That is a lot easier, obviously we are in namespace `boost` and can drop it from the types in template arguments and parameters with low risk of confusion. It works in the source code, it should work in the debbuger.
4. Use _common names_ when possible.

I don't like to add user-controllable options, it makes the debugger more complicated, stateful, and harder to use correctly, but I see no way to make this magically work out of the box.

### Common names? ###
Consider `std::vector<std::string>`. That is a very common type in C++. It is also an illusion. When a debugger reads your binaries,
it will come out like this instead:
`std::vector<std::basic_string<char,std::char_traits<char>, std::allocator<char>, std::allocator<std::basic_string<char,std::char_traits<char>, std::allocator<char>>>`.
It is possible to translate this back to `std::vector` and `std::string`, but there are several steps involved.
1. Simplify `std::basic_string<char,std::char_traits<char>,std::allocator<char>>` to `std::basic_string<char>`.  
   The DWARF debug information for a type can record the template parameters and it can record that a template argument has been defaulted. There is nothing we can do if we only have a symbol table, but with full debug information it is possible to recognize defaulted template parameters and strip them from the type signature.
2. Simplify `std::basic_string<char>` to `std::string`. Debug information can also record `typedef`s, and a debugger could recognize the following facts:
* `std::string` is the only typedef for `std::basic_string<char>` in namespace std.
* `std::string` is used more often than `std::basic_string<char>` in user source code. It won't be true for generic code that deals with `template<typename Char> std::basic_string<Char>`, but a debugger can recognize that.
We can use that to conclude that `std::string` is probably the preferred name for `std::basic_string<char>` without explicit hard-coding.
If you actually check the generated debug information, you will find that it is full of `typedef`s that are not preferred names, like `fast_uint32_t` for `unsigned`. This is going to be a heuristic.
* A producer of debug information needs to faithfully record the typedefs that we use and cannot resolve them to their targets, including template parameters.

If you check this out yourself ([Compiler Explorer](https://godbolt.org/z/n1zxPGd1v)), you'll find that I simplified things for you:  
because libstdc++ broke the ABI with previous versions to be C++11-compliant, your string template is actually called `std::__cx11::basic_string`, but `__cx11` is an inline namespace, so it is transparent to source code, but a debugger has to manually remove it to provide users with names they recognize.  
If you use clang with libc++, you will find all standard types in the inline namespace `__1` instead, e.g. `std::__1::vector`.

### Template all the things ###
So in summary: it's not going to be pretty, but it's going to work, _when it comes to templated functions and classes_.
Of course, in C++ we template all the things, not just functions and classes.

### Debug information for template aliases is a mess ###

I've read the DWARF5 standard, I've compiled a simple example with clang ([Compiler Explorer](https://godbolt.org/z/MGb5eGnYv)), and I've checked the generated debug information.
I expected to find a _Debug Information Entry_ of type `DW_TAG_template_alias`. Neither clang nor gcc will emit entries with that tag,
they generate an entry of type `DW_TAG_typedef`, just like the debug information for a simple `typedef`.
As far as I can tell, that is a good thing and the introduction of `DW_TAG_template_alias` was a mistake. The compiler behavior mirrors how template functions and classes are handled, there is no special type for them compared to non-templated entries, instead they get augmented with child entries (with types like `DW_TAG_template_type_parameter`) describing their type and non-type template parameters.
Unfortunately, clang does not generate these children for template aliases. Which means all the stuff I've talked about when it comes to common names does not work for template aliases at the moment.

### Could it _be_ any worse? ###

Then I compiled the same code with gcc. Like clang it does not record the template arguments as children in the debug information.
Unlike clang it doesn't even record the template arguments in the name of the type. The compiler generates multiple `typedef`s for different types all having the same name. Neither gdb nor lldb expect that and will show you the wrong type if you ask them to display a type by specifiying its name. I was planning on using debug information to detect ODR violations but gcc produces ODR violations in the debug information that do not exist in the source.

### Template variables ###
Clang does not record the template parameters in the variable name, but adds them as attributes to debug information.
For template variables, gcc records the template arguments neither in the name nor in the attributes, but if your variable has external linkage (constexpr variables won't have it), you can recover the type from the mangled name, as long as it is a named type...  

I believe all debuggers could provide a better debugging experience if compilers would generate better debug information in this case.


## Entities without names ##

Long before we started C++, C was already full of anonymous types.
{% highlight C %}
struct {
    int c;
    int d;
} a;
struct X {
enum {
    yes = 0
};
};
struct Y {
enum {
    yes = 1
};
};
{% endhighlight %}

And of course, true abominations like

{% highlight C %}
struct Z {
    union {
        long a;
        char b;
        struct {
            int c;
            int d;
            union {
                float f;
            };
        };
    };
};
{% endhighlight %}

C++ is different. Because of name mangling, every unnamed type needs a name if it is a template parameter, function argument or the type of a global variable, including static variables inside functions.
On the other hand, it is vitally important to record that these types have no name, ODR rules treat named types and anonymous types differently:  
types with a name in different compilation units must have the same definition or we are in _ill-formed, no diagnostic required_ territory. Which is a terrible territory to find yourself lost in, so if there is any way to diagnose it anyway and find our way out, that seems worth doing.

This is the situation for a templated function `tfunc` taking one argument and being instantiated with an anonymous struct ([Compiler Explorer](https://godbolt.org/z/r34WGGTrs)):
* The debug information entry for the struct records no name and debug information for types, unfortunately, never records mangled names.
* The debug information entry for the function records a name `tfunc<<unnamed struct> >` and a mangled name `_Z5tfuncI8._anon_0EvT_`.
* You can try to parse the function name to get the name of the template argument, but the choice of angled brackets here is very unfortunate and going to confuse naive parsers.
* We can demangle the mangled name and get `void tfunc<._anon_0>(._anon_0)` giving us `._anon_0` as the internal name for our anonymous struct. I find it quite unfortunate that the function name is different from the name we get by demangling.
* On its own, `._anon_0` is of course an invalid identifier and requires special handling for recognizing it.

That was gcc, clang gives a different picture:
* The debug information entry for the function records a name `tfunc<(unnamed struct at /app/example.cpp:1:1)>` and a mangled name `_Z5tfuncI3$_0EvT_`.
* You can try to parse the function name to get the name of the template argument, but the choice of angled and brackets and parentheses will break the parser that we just fixed to work with gcc's naming scheme.
* We can demangle the mangled name and get `void tfunc<$_0>($_0)` giving us `$_0` as the internal name for our anonymous struct. I find it quite unfortunate that the function name is different from the name we get by demangling.
* On its own, `$_0` is an invalid identifier, but using `$` to create identifiers that won't clash with anything else is quite common. It might cause surprises if you do any kind of shell scripting with `nm` or `readelf` and don't expect it.
* The debug information entry for the struct records no name and debug information for types, unfortunately, never records mangled names.

This is kinda disappointing, but as long as I have to support only two compilers I think I can manage.  
Except for lambda expressions, obviously.

The type of a lambda expression is an unnamed struct, so nothing changes here, right? Right? [Compiler Explorer](https://godbolt.org/z/99ToETYKd)

Here is gcc:
* The debug information entry for the struct records no name and debug information for types, unfortunately, never records mangled names. This is not a copy-paste mistake, that's how unfortunate that is.
* The debug information entry for the function records a name `tfunc<<lambda(int)> >` and a mangled name `_Z5tfuncIN1aMUliE_EEvT_`. In case you can't demangle in your head, this means that `N1aMUliE_E` is the mangled type name of the lambda, which demangles to `a::{lambda(int)#1}`. 
* Angled brackets in the name, again. But different from an anonymous struct.
* We can demangle the mangled name and get `void tfunc<a::{lambda(int)#1}>(a::{lambda(int)#1})`. For some reason anonymous structs are numbered starting with zero, but lambda expressions start from one.
* This time, the mangled name for our anonymous type starts like any other mangled name and follows some special rules made for lambda expressions.

Here is clang:
* The debug information entry for the struct records no name and debug information for types, unfortunately, never records mangled names.
* The debug information entry for the function records a name `tfunc<(lambda at /app/example.cpp:1:10)>` and a mangled name `_Z5tfuncI3$_0EvT_`. Yes, that is exactly the same mangled name as for the anonymous struct.
* Note that this time there are _no_ angled brackets for the name of the anonymous type and if we had written a parser, we would have to fix it again.

Lambda expressions are everywhere, including in other lambda expressions. There is no way to consistently give the same name for the same thing. By the way, for me as C++ developer the name of `[](int x){return x;}` is `[](int x){return x;}`, maybe shortened to `[](int x) -> int` if the body does not fit into one line. Consequently, the type of this thing in my head is `decltype([](int x){return x;})`.

If you carefully study the generated debug information, you will realize:
* There is no way to tell wether it is the type of a lambda expression or an anonymous struct.
* Not that it matters, but gcc makes them `struct`s and clang makes them `class`es. 

Okay, this was the introduction, now lets talk about the _problems_ with lambda expressions ([Compiler Explorerer](https://godbolt.org/z/WE38q53xz)).
* lambda captures
* the call operator

A lambda expression can capture by value a type with a user-defined destructor.
That means our anonymous type now has a destructor. In fact, it has a named destructor, gcc calls it `~<lambda>()`, while clang calls it
`~(lambda at /app/example.cpp:15:14)`. For gcc, the `tfunc` signature becomes `tfunc<main(int, char**)::<lambda()> >`.
This time, the outer scope (i.e. `main`) is part of the name, not just of the mangled name. I think it would be more consistent if gcc had named the destructor `~<lambda()>`. Clang is consistent with its function signature of `tfunc<(lambda at /app/example.cpp:15:14)>`, that's cool. Anyway, we can now tell that our type is the type of lambda expression by looking at the name of constructors or destructors. Doesn't work for lambda expressions without captures, but I'll take what I can get.

Of course, if we are debugging, we would like to inspect the captures of our lambda expression,
they should be member variables. And they are! Depending on the compiler, they have the name you expect or that name prefixed with two underscores.

Moving on to the call operator:  
It is called `operator()` as you would expect.
All non-static methods for a class or struct have a hidden parameter as their first argument, which is marked _artificial_ in the debug information. For ordinary classes it is called `this`. A lambda expression's call operator also has this parameter, but is called `__closure` in gcc and doesn't have a name at all when using clang. A debugger might need to show this parameter when debugging, so we need to make up a name for it, might as well go with `__closure`. If you are wondering, why aren't they using `this`, then its likely because of the special case of capturing `this` for a lambda expression defined in a non-static method.

Currently both gdb and lldb can fail to properly display the lambda and enclosing scope when showing a backtrace:  
the call operator appears as `operator()` instead of `main::{lambda()#1}::operator()() const` or `(lambda at /app/example.cpp:15:14)::operator()()`.

If you like you can try how my WIP core dump analyzer displays the [binary and debug information for gcc](https://core-explorer.github.io/core-explorer/?download=https://core-explorer.github.io/core-explorer/example/ui_nightmare.gcc) and [for clang](https://core-explorer.github.io/core-explorer/?download=https://core-explorer.github.io/core-explorer/example/ui_nightmare.clang)

## Names without entities ##

I am not talking about forward declarations (though I am sure, those are going to cause me headaches sooner or later), I am talking about functions in the binary that do not have any meaningful source code.

For example

{% highlight C %}
void func() {
    std::vector<int> victor;
    std::set<int> seth;
    std::string sven;
}
{% endhighlight %}

The final `}` does a lot of work, including calling three destructors. There is no good way for a line-based source debugger to allow you to conveniently step through the destruction sequence when debugging memory corruption.
You must work on the assembly, and you need to know the assembly language for your target machine, and you must not let your muscle memory get in the way and accidently step one line instead of one instruction.

This is also the part where Compiler Explorer excels, it was built to show you the hidden costs (or their absence) of C++ constructs. The UI assumes you are an expert in assembly, though (it's never too late to start your journey).

This is the nasty part, function prologues, and epilogs and stuff like that.

What if we had a class instead of a function?


{% highlight C %}
struct Funk {
    std::vector<int> victor;
    std::set<int> seth;
    std::string sven;
};
{% endhighlight %}

This class has a compiler generated destructor, it'll show up in your symbol table, in your debug information, it even has a name `Funk::~Funk()`. It has everything except source code. We know exactly what it does, it is required to call the destructors in reverse order:

{% highlight C %}
{
    sven.~string();
    seth.~set();
    victor.~vector();
}
{% endhighlight %}

There are other things a destructor has to do, particularly for virtual classes, but we can imagine the compiler not just generating code but also source code for the destructor. It doesn't have to be valid, it just needs to show us where we are in the destruction sequence, so that we can step through it using existing debuggers. They need lines and line numbers, they don't need source files.

This was state of the art in 1998, but since then we've had an explosion of compiler generated methods:
* defaulted constructors
* defaulted assignment operators
* defaulted comparisons
* fold expressions

All of these will be horrible to debug _for the original developer_. A North Korean security researcher reverse engineering a shipped binary with no access to source code or debug information will a have much easier job.
His debugger will run a decompiler on the binary, allowing him to step through all these methods. It is not necessary for the decompiled code to be correct or compilable.

I know the journey from source code to finished binary:
1. We start with the source code. Let's ignore the preprocessor.
2. The compiler turns source code into an abstract syntax tree (AST).
3. There might be optimization passes running on the AST.
4. The abstract syntax tree gets turned into an intermediate representation like LLVM IR or GIMPLE.
5. There is sequence of optimization passes that modify the intermediate representation.
6. The intermediate representation gets turned into machine code (technically assembly).
7. There might be more optimization passes, this time on the generated assembly.
8. Finally the assembly gets turned into binary code by the assembler (which also emits unwinding information and symbol tables).

A disassembler succeeds in going from step 8 to step 7, this always works except for obfuscated malware.
Our debugging information then takes us from step 7 to step 1, skipping everything in between.

If that doesn't work, you are stuck working on disassembly. You can try lifting your machine code back to IR and decompiling the IR, as the security researchers do. But that seems really wasteful to me.

We have the AST, the compiler can dump the AST.
We have the initial IR, we have the final IR, the compiler can dump the IR.

We even have tools like Cpp Insights, that translate intermediate compiler stages back into C++.

Here is what I want from a future debugger when working on a hard to debug problem:
* It should be able to show me the source code
* It should be able to show me the IR
* It should be able to show me disassembly
* I want to be able to step through the program on each of these levels of abstraction, depending on the function and the problem.

As I said, I am writing my own debugger, so I will make this happen one way or other. Eventually.

But you don't have to wait for me. Go ahead and implement this:  
Get a reproducible build going with clang, compile it into an executable with debug information.
Compile it to LLVM IR. Compile the IR to an executable with debug information.  
Check that the executables match and tell me how it works in practice.

<small>
I've mentioned several bugs or short comings in gcc, clang, gdb and lldb.
I know the right thing to do is to search for corresponding open issues and if there are none, create a new one.
I haven't done it, I will do it, I just needed to get some ranting out before I can write concise bug reports. Thanks.
</small>