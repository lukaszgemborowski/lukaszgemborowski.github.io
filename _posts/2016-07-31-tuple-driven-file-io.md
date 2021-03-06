---
layout: post
title:  "tuple-driven file read/write with variadic templates"
date:   2016-07-31 11:35:21 +0100
categories: c++ c++11 tuple
---

Today will be something completely different. We’ll focus on C++11 feature called variadic templates. Goal of
this post is providing functions which will read binary files described by tuples, for example you can describe
content of the file as std::tuple and by only one call you will dump or read this tuple to file. Tuple is
somehow similar to std::array with the difference that it’s capable of holding different types of data in one container.

## Variadic templates ##

First of all we need to understand what variadic template is. You can of course start with some overview on
[wiki](https://en.wikipedia.org/wiki/Variadic_template). I think the simplest usable example of variadic
templates would be something like this snippet:

{% highlight c++ %}
#include <iostream>

void print_many() {}

template<typename T, typename... Args>
void print_many(T value, Args... args)
{
	std::cout << value << ", ";
	print_many(args...);
}

int main()
{
	print_many("foo", 1, 2);
}
{% endhighlight %}

What is it? We have created print_many variadic function. It accepts any number of arguments of any type. We
call it with 3 arguments of types ‘const char*’, ‘int’ and ‘int’. Compiler matches print_many(T value, Args… args)
and put ‘const char*’ as T and {‘int’, ‘int’} as Args… The function is now created by the compiler. We cannot
easily access elements in parameter pack (args) but we do have access to the first argument ‘T value’. We can
access it as any other regular function argument. In this case we will pass it to std::cout for printing on
stdout. After that we’re recursively calling print_many but this time with 2 arguments, int’s. Now T becomes
int and Args becomes {‘int’}. Recursing again, T is once again int but Args is empty, printing last item and
recursing again with empty parameter pack. This is why we have explicitly defined print_many() without any
arguments, just to satisfy termination condition. The equivalent of the above code without variadic templates
would be (this is what will be generated by compiler):

{% highlight c++ %}
#include <iostream>

void print_many() {}

void print_many(int c) {
	std::cout << c << ", ";
}

void print_many(int b, int c) {
	std::cout << b << ", ";
	print_many(c);
}

void print_many(const char* a, int b, int c) {
	std::cout << a << ", ";
	print_many(b, c);
}

int main()
{
	print_many("foo", 1, 2);
}
{% endhighlight %}

And this is how for example make_shared was implemented without variadic templates. Libraries (boost) provides
make_shared overload for any number of arguments. To support N arguments you need to create N+1 functions.

## Iterating through std::tuple ##

Next step will be to write some routines for iterating through std::tuple and printing it content. It will be quite similar to our previous example with print_many().

{% highlight c++ %}
#include <iostream>
#include <tuple>
#include <type_traits>

template<std::size_t I = 0, typename... Tp>
typename std::enable_if<I == sizeof...(Tp), void>::type
print_tuple(std::tuple<Tp...> &) {}

template<std::size_t I = 0, typename... Tp>
typename std::enable_if<I < sizeof...(Tp), void>::type
print_tuple(std::tuple<Tp...> &t) {
	std::cout << std::get<I>(t) << ", ";
	print_tuple<I + 1, Tp...>(t);
}

int main()
{
	std::tuple<char, float, int> t{'A', 1.5f, 6};
	print_tuple(t);
}
{% endhighlight %}

It’s a little bit more complicated than the first example. But let’s start from the beginning, what happen
when you call print_tuple function? Compiler will match both functions so the call should be ambiguous. It is
not thanks to enable_if which uses C++ feature called SFINAE. The I argument defaults to 0 which is less than
sizeof…(Tp) (3 at this point) so enable_if will produce error for first overload leaving us with the second
one and this is what we want. Now each call to print_tuple will cause calling second variant until we meet the
I == sizeof…(Tp) condition. You can check this by changing print_tuple call in main function to
print_tuple<3>(t);, it will cause the condition in first overload to be true and the second to be false.
Nothing will be printed, you have just used the first function. But let’s dive into implementation of the
second print_tuple, it’s two-liner. First line is just for getting an I-th element from tuple and printing it.
The second line is recursive call with I+1 causing next element to be recursively printed. It will recurse
until first print_tuple’s enable_if condition evaluates to true, namely until I == 3 in our case, this will
terminate recursion.

## Putting it all together ##

Only thing which is left is to provide functions for reading and writing data. They will be very similar to
print_tuple but instead of writing to stdout they will read/write from a file stream.

{% highlight c++ %}
#include <iostream>
#include <tuple>
#include <type_traits>
#include <fstream>

// printing tuple
template<std::size_t I = 0, typename... Tp>
typename std::enable_if<I == sizeof...(Tp), void>::type
print_tuple(std::tuple<Tp...> &) {}

template<std::size_t I = 0, typename... Tp>
typename std::enable_if<I < sizeof...(Tp), void>::type
print_tuple(std::tuple<Tp...> &t) {
	std::cout << std::get<I>(t) << ", ";
	print_tuple<I + 1, Tp...>(t);
}

// writing tuple to a file
template<std::size_t I = 0, typename... Tp>
typename std::enable_if<I == sizeof...(Tp), void>::type
write_tuple(std::ofstream &, std::tuple<Tp...> &) {}

template<std::size_t I = 0, typename... Tp>
typename std::enable_if<I < sizeof...(Tp), void>::type
write_tuple(std::ofstream &ofs, std::tuple<Tp...> &t) {
	ofs.write((char *)&std::get<I>(t), sizeof(typename std::tuple_element<I, std::tuple<Tp...>>::type));
	write_tuple<I + 1, Tp...>(ofs, t);
}

// reading tuple from a file
template<std::size_t I = 0, typename... Tp>
typename std::enable_if<I == sizeof...(Tp), void>::type
read_tuple(std::ifstream &, std::tuple<Tp...> &) {}

template<std::size_t I = 0, typename... Tp>
typename std::enable_if<I < sizeof...(Tp), void>::type
read_tuple(std::ifstream &ifs, std::tuple<Tp...> &t) {
	ifs.read((char *)&std::get<I>(t), sizeof(typename std::tuple_element<I, std::tuple<Tp...>>::type));
	read_tuple<I + 1, Tp...>(ifs, t);
}

int main()
{
	std::tuple<char, float, int> tout{'A', 1.5f, 6};
	std::tuple<char, float, int> tin;

	std::ofstream out("test.bin");
	std::cout << "Writing tuple to test.bin: ";
	print_tuple(tout);       // print out the tuple
	write_tuple(out, tout);  // write tuple to file
	out.flush();             // flush the file so we can read it immediately
	std::cout << std::endl;

	std::ifstream in("test.bin");
	std::cout << "reading tuple from test.bin: ";
	read_tuple(in, tin);     // read the tuple from file
	print_tuple(tin);        // print it out
	std::cout << std::endl;
}
{% endhighlight %}

Simple, isn’t it?;-) Looks pretty obfuscated but this is C++ meta programming. As a homework you can create a
function for iterating through the tuple and applying some functor over each element. That way you can have
only one iterate_over function and three functors for reading from file, writing to file and printing to stdout.
No code duplication!
