# Crash Course: meta

<!--
@cond TURN_OFF_DOXYGEN
-->
# Table of Contents

TODO to be updated

* [Introduction](#introduction)
* [Reflection in a nutshell](#reflection-in-a-nutshell)
* [Any as in any type](#any-as-in-any-type)
* [Enjoy the runtime](#enjoy-the-runtime)
* [Properties and meta objects](#properties-and-meta-objects)
<!--
@endcond TURN_OFF_DOXYGEN
-->

# Introduction

Reflection (or rather, its lack) is a trending topic in the C++ world and, in
the specific case of `EnTT`, a tool that can unlock a lot of other features. I
looked for a third-party library that met my needs on the subject, but I always
came across some details that I didn't like: macros, being intrusive, too many
allocations. In one word: unsatisfactory.<br/>
I finally decided to write a built-in, non-intrusive and macro-free runtime
reflection system for `EnTT. Maybe I didn't do better than others or maybe yes,
time will tell me, but at least I can model this tool around the library to
which it belongs and not vice versa.

# Reflection in a nutshell

Reflection always starts from real types (users cannot reflect imaginary types
and it would not make much sense, we wouldn't be talking about reflection
anymore).<br/>
To _reflect_ a type, the library provides the `reflect` function:

```cpp
auto factory = entt::reflect<my_type>("reflected_type");
```

It accepts the type to reflect as a template parameter and an optional name as
an argument. Names are important because users can retrieve meta types at
runtime by searching for them by name. However, there are cases in which users
can be interested in adding features to a reflected type so that the reflection
system can use it correctly under the hood, but they don't want to allow
searching the type by name.<br/>
In both cases, the returned value is a factory object to use to continue
building the meta type.

A factory is such that all its member functions returns the factory itself.
It can be used to extend the reflected type and add the following:

* _Constructors_. Actual constructors can be assigned to a reflected type by
  specifying their list of arguments. Free functions (namely, factories) can be
  used as well, as long as the return type is the expected one. From a client's
  point of view, nothing changes if a constructor is a free function or an
  actual constructor.<br/>
  Use the `ctor` member function for this purpose:

  ```cpp
  entt::reflect<my_type>("reflected").ctor<int, char>().ctor<&factory>();
  ```

* _Destructors_. Free functions can be set as destructors of reflected types.
  The purpose is to give users the ability to free up resources that require
  special treatment  before an object is actually destroyed.<br/>
  Use the `dtor` member function for this purpose:

  ```cpp
  entt::reflect<my_type>("reflected").dtor<&destroy>();
  ```

* _Data members_. Both real data members of the underlying type and static or
  global variables can be attached to a meta type. From a client's point of
  view, all the variables associated with the reflected type will appear as if
  they were part of the type.<br/>
  Use the `data` member function for this purpose:

  ```cpp
  entt::reflect<my_type>("reflected")
      .data<&my_type::static_variable>("static")
      .data<&my_type::data_member>("member")
      .data<&global_variable>("global");
  ```

  This function requires as an argument the name to give to the meta data once
  created. Users can then access meta data at runtime by searching for them by
  name.

* _Member functions_. Both real member functions of the underlying type and free
  functions can be attached to a meta type. From a client's point of view, all
  the functions associated with the reflected type will appear as if they were
  part of the type.<br/>
  Use the `func` member function for this purpose:

  ```cpp
  entt::reflect<my_type>("reflected")
      .func<&my_type::static_function>("static")
      .func<&my_type::member_function>("member")
      .func<&free_function>("free");
  ```

  This function requires as an argument the name to give to the meta function
  once created. Users can then access meta functions at runtime by searching for
  them by name.

* _Base classes_. A base class is such that the underlying type is actually
  derived from it. In this case, the reflection system tracks the relationship
  and allows for implicit casts at runtime when required.<br/>
  Use the `base` member function for this purpose:

  ```cpp
  entt::reflect<derived_type>("derived").base<base_type>();
  ```

  From now on, wherever a `my_base_type` is required, an instance of `my_type`
  will also be accepted.

* _Conversion functions_. Actual types can be converted, this is a fact. Just
  think of the relationship between a `double` and an `int` to see it. Similar
  to bases, conversion functions allow users to define conversions that will be
  implicitly performed by the reflection system when required.<br/>
  Use the `conv` member function for this purpose:

  ```cpp
  entt::reflect<double>().conv<int>();
  ```

That's all, everything users need to create meta types and enjoy the reflection
system. At first glance it may not seem that much, but users usually learn to
appreciate it over time.<br/>
Also, do not forget what these few lines hide under the hood: a built-in,
non-intrusive and macro-free system for reflection in C++. Features that are
definitely worth the price, at least for me.

# Any as in any type

The reflection system comes with its own meta any type. It may seem redundant
since C++17 introduced `std::any`, but it is not.<br/>
In fact, the _type_ returned by an `std::any` is a const reference to an
`std::type_info`, an implementation defined class that's not something everyone
wants to see in a software. Furthermore, the class `std::type_info` suffers from
some design flaws and there is even no way to _convert_ an `std::type_info` into
a meta type, thus linking the two worlds.

A meta any object provides an API similar to that of its most famous counterpart
and serves the same purpose of being an opaque container for any type of
value.<br/>
It minimizes the allocations required, which are almost absent thanks to _SBO_
techniques. In fact, unless users deal with _fat types_ and create instances of
them though the reflection system, allocations are at zero.

A meta any object can be created by any other object or as an empty container
to initialize later:

```cpp
// a meta any object that contains an int
entt::meta_any any{0};

// an empty meta any object
entt::meta_any empty{};
```

It can be constructed or assigned by copy and move and it takes the burden of
destroying the contained object when required.<br/>
A meta any object has a `type` member function that returns a pointer to the
meta type of the contained value, if any. The member functions `can_cast` and
`can_convert` are used to know if the underlying object has a given type as a
base or if it can be converted implicitly to it. Similarly, `cast` and `convert`
do what they promise and return the expected value.

<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<

TODO

# Enjoy the runtime

Once the web of reflected types has been constructed, it's a matter of using it
at runtime where required.<br/>
All this has the great merit that, unlike the vast majority of the things
present in this library and closely linked to the compile-time, the reflection
system stands in fact as a non-intrusive tool for the runtime.

To search for a reflected type there are two options: by type or by name. In
both cases, the search can be done by means of the `resolve` function:

```cpp
// search for a reflected type by type
auto *by_type = entt::resolve<my_type>();

// search for a reflected type by name
auto *by_name = entt::resolve("reflected_type");
```

The returned value is a pointer to a `meta_type` object. This type of objects
offer a series of useful tools to know the name, to iterate all the meta
objects associated with them and even to build or destroy instances of the
underlying type.<br/>
Refer to the inline documentation for all the details.

Once someone has a meta-type, meta data and meta functions can be easily
retrieved by means of two member functions: `data` and `func`. In both cases,
members are searched by name:

```cpp
auto *data = entt::resolve<my_type>().data("cvalue")
auto *func = entt::resolve<my_type>().func("cfunc")
```

Meta data offer a set of member functions including methods to know the name, to
get the meta type of the underlying variable, to set and get the value of the
actual variable and so on.<br/>
Meta functions offer a slightly different set of member functions, among them
methods to know the meta type associated to the returnr type, to retrieve the
meta types of the arguments, to invoke the underlying function and so on.<br/>
Refer to the inline documentation for all the details.

Similarly, constructors can also be retrieved (by argument list) as well as the
destructor associated with the meta type. As for the other meta objects, also in
this case we have a set of tools to invoke and explore constructors and
destructors in all their parts.

Finally, meta types and meta objects in general contain much more than what is
said: a plethora of functions in addition to those listed whose purposes and
uses go unfortunately beyond the scope of this document.<br/>
I invite anyone interested in the subject to look at the code, experiment and
read the official documentation to get the best out of this powerful tool.

# Properties and meta objects

Sometimes (ie when it comes to creating an editor) it might be useful to be able
to attach properties to the meta objects created. Fortunately, this is possible
for all of them.<br/>
To attach a property to a meta object, no matter what, it is sufficient to
provide an object at the time of construction such that `std::get<0>` and
`std::get<1>` are valid for it. In other terms, the properties are nothing more
than key/value pairs users can put in an `std::pair`. As an example:

```cpp
entt::reflect<my_type>("reflected_type", std::make_pair("tooltip"_hs, "message"));
```

All the meta objects offer then a couple of member functions named `prop` to
iterate all the properties at once and to search a specific property by key:

```cpp
// iterate all the properties of a meta type
entt::resolve<my_type>().prop([](auto *prop) { /* ... */ });

// search for a given property by name
auto *prop = entt::resolve<my_type>().prop("tooltip"_hs);
```

Meta properties are objects having a fairly poor interface, all in all. They
only provide the `key` and the `value` member functions to be used to retrieve
the key and the value contained in the form of meta any objects, respectively.
