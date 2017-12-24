---
toc:    true
title:  "Templated classes, their templated parents, and their templated methods"
date:   2017-12-24
---

Mixing templates and inheritance can cause some unfamiliar and confusing compiler errors.

## The problem

A convoluted title for a somewhat convoluted subject. Imagine you have the
following class template

```cpp
template <typename T>
struct Base {
    void method_1() {}

    template <typename U>
    void method_2() {}
};
```

Now, if you want to inherit from this class, chances are that you want to
support the full range of templated parameters of the base class, so you would
write your derived class as follows:

```cpp
template <typename T>
struct Derived : Base<T> {
    void method_3();
};
```

What happens if `Derived<T>::method_3()` wants to call `Base<T>::method_1()`?
You might write something like

```cpp
template <typename T>
void Derived<T>::method_3() {
    method_1();
}
```

If you try to compile this, you would likely get a compiler error similar to
the output I get using clang 5.0.0

```shell
16 : <source>:16:5: error: use of undeclared identifier 'method_1'; did you
mean 'method_3'?
    method_1();
    ^~~~~~~~
    method_3
15 : <source>:15:18: note: 'method_3' declared here
void Derived<T>::method_3() {
                 ^
1 error generated.
Compiler returned: 1
```

## Explanation

This happens because in the context of `Derived<T>`, `Base<T>` is a dependent
name. To put it simply, non-dependent names are looked up and bound at the
point of template definition. Dependent names, on the other hand, are looked
up and bound at the point of template instantiation.

In fact, all templates are compiled in two steps, referred to as **Two Phase
Lookup**. Among other things, the first phase is responsible for looking up
non-dependent names, while the second phase is responsible for the dependent
name lookup.

One of the reasons for this is to allow for templated code to refer to
functions or classes that are visible only at the point of template
instantiation, not at the point of definition.

## The fix

There are a few ways to work with dependent names.

### Using the `using` keyword

The `using` keyword can be used to bring dependent names into the scope of
the class or the method, so that you can refer to the dependent name like you
would normally.

```cpp
template <typename T>
struct Derived : Base<T> {
    using Base<T>::method_1;

    void method_3();
};

template <typename T>
void Derived<T>::method_3() {
    method_1();
}
```

### Referring to the dependent name directly

Instead of using the `using` keyword, you can instead refer directly to the
class name when calling the function.

```cpp
template <typename T>
void Derived<T>::method_3() {
    Base<T>::method_1();
}
```

### Using `this`

Prefixing `this->` in front of the method calls will also fix the compilation
errors. In fact, prefixing the call with the `this` tells the compiler to do
the name lookup specifically in the second phase of the template compilation.

```cpp
template <typename T>
void Derived<T>::method_3() {
    this->method_1();
}
```

### The recommended method

Referring to dependent names by prefixing with `this->` is the recommended
way. It is a lot more resilient to code changes; if you change the name of
`Base<T>` in the examples above, `this->` would still work correctly, while
the others would have to be updated accordingly.

This can be particularly important when dealing with class templates with
virtual member functions.

```cpp
template <typename T>
struct Base {
    virtual void v_method() {}
};

template <typename T>
struct Derived : Base<T> {
    void call_method();
};
```

If, for example, the definition of `Derived<T>::call_method()` is

```cpp
template <typename T>
void Derived<T>::call_method() {
    Base<T>::v_method();
}
```

And then you proceed to add an override for `Base<T>::v_method()` in
`Derived<T>`, `Derived<T>::call_method()` will need to be updated. If you
forget to update definition of `Derived<T>::call_method()`, the code will
still compile but you'll be inadvertently calling the wrong member function.
On the other hand, using `this->` avoids this scenario altogether.

```cpp
template <typename T>
void Derived<T>::call_method() {
    this->v_method();
}
```

## A wrinkle

In all the above examples, `this->` works because it refers to the **current
instantiation** (of the class templates). If, on the other hand, you're
referring to a different template instantiation while in a template
instantiation (essentially, a dependent template), you have to make use of
the `template` keyword in a way different than you may be used to.

```cpp
template <typename T>
struct Base {
    template <typename U>
    void method_1() {}
};

template <typename T>
struct Derived : Base<T> {
    void method_2() {
        this->template method_1<T>();
    }
}
```

In order for `Derived<T>::method_2()` to call `Base<T>::method_1<U>()`, the
name of the member function template needs to be prefixed with the keyword
`template`. This is because the member function represented by
`Base<T>::method_1<U>()` (i.e., the fully instantiated member function) is
not available in the instantiation of the class template (`Derived<T>`). As a
result, the C++ parsing rules state that the `<` in `this->method_1<T>()` be
read as a less-than sign. The use of the `template` keyword tells the
compiler to look at the original class template for the definition of the
member function template and instead instantiate that. In fact, the
`template` keyword can be used in this manner to even refer to member
functions (as opposed to member function **templates**). It just isn't
explicitly required.

```cpp
template <typename T>
struct Base {
    template <typename U>
    void method_1() {}

    void method_2() {}
};

template <typename T>
struct Derived : Base<T> {
    void method_3() {
        this->template method_1<T>();
        this->template method_2();
    }
};
```

## A note about Visual Studio's behavior

Microsoft Visual Studio, until version [Visual Studio 2017
"15.3"](https://blogs.msdn.microsoft.com/vcblog/2017/09/11/two-phase-name-lookup-support-comes-to-msvc/)
(where you have to use the `/permissive-` switch), did not support two phase
name lookup. As such, the compiler is more permissive and does not require
the special treatment of dependent names in template definitions.

## Further reading

[Dependent Names](http://en.cppreference.com/w/cpp/language/dependent_name)

