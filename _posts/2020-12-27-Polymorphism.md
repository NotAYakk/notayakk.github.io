---
layout: post
published: false
title: Intro to C++ Polymorphism 
---


I'm going to use a broad definition of Polymorphism; Polymorphism is when you have some human written code that behaves differently based on the "type" of the data it works on, and the language facilities that help you make this work.

This definition is ridiculously generic.  Almost any branch ends up being a kind of Polymorphism.  But instead of being a purist, and saying things are Polymorphic or not, I am going to focus on the ways the language makes Polymorphic code and data work better.

C++ is descended from C, and was originally called C with Classes.  The two branches of C++ polymorphism support are dynamic dispatch and code generation.

The most familiar form of polymorphism in C++ is good old virtual functions and inheritance.  This comes from a C technique where you stuff a table of functions in your structure, then dispatch operations to that table.

The code generation form of polymorphism is templates; this comes from the C "native" code generation via macros -- you could make blocks like:

    #define SWAP3WAY(TMP, B, C) \
      do { \
        (A) = (C); \
        (C) = (B); \
        (B) = (A); \
      } while(false)

Here we generate code that does something different if the types of `TMP` `B` and `C` are different.  C programmers can build really complicated structures this way, like linked list that stored different types in the nodes depending on the macro parameter.

C++ made both of these easier by wrapping them up.

    struct Interface {
      virtual void do_something() = 0;
    };
    struct Implementation:Interface {
      void do_something() override { printf("hello world"); }
    };

We can write a pseudo-C version in C++ as follows:

    struct Interface_vtable {
      void(*do_something)(void*) = nullptr;
    };
    struct Interface {
      Interface_vtable const* vtable = nullptr;
      void do_something() {
        return vtable->do_something( this );
      }
    };
    struct Implementation:Interface {
      static Interface_vtable make_vtable() {
        return {&Implementation::do_something_entry};
      }
      static Interface_vtable const* get_vtable() {
        static const auto retval = make_vtable();
        return &retval;
      }
      Implementation():
        Interface{ get_vtable() }
      {}
      static void do_something_entry(void* self) {
        static_cast<Implementation*>(static_cast<Interface*>(self))->do_something_impl();
      }
      void do_something_impl() { printf("hello world"); }
    };

This mimics the C++ `virtual` method in a basically functionally identical way without ever using the `virtual` keyword.  It does use a bunch of C++ syntatic sugar; the pure C version wouldn't have methods, but functions that operate on the struct direct, wouldn't have a constructor but rather a construction function, etc.

But I presume you can understand how you'd implement vtables in C.  Fancier cases lead to more boilerplate.

Why do you want to know this?  Because in the above the C++ language made a bunch of decisions.

It decided that the vtable would have 1 copy per type, not 1 copy per object.  It decided to store a pointer to it, not the functions within the object itself.  It decided to store that pointer to the functions directly with the data.  It decided when the table is created and modified at each level.  C++ decided that each type has a complete copy of the vtable.  And it  decided that the only way to make a new type -- a new vtable -- is by writing a new C++ type.  It decided that vtable entries are offsets and not names.

And probably a bunch of other implicit decisions.

All of these decisions are not absolute, and alternative choices could have given other benefits.

The third major branch of vanilla C++ polymorphism shows up at this point.  What is known in C++ circles as "type erasure", and you probably have used as `std::function`.

`std::function` is polymorphic, but it isn't static polymorpism like `std::vector<int>`, nor is it inheritance based polymorphism.

Instead, `std::function<int()>` uses template based code generation to generate dynamic polymorphism on demand.

Now, while `std::function` is a `std` library element, and as such its implementation is **not** formally defined, it turns out it can be implemented in C++ as specified.

A simple way to get something like `std::function` is:

    template<class R, class...Args>
    struct ICallable {
      virtual ~ICallable() = default;
      virtual R invoke(Args&&...) = 0;
    };
    template<class F, class R, class...Args>
    struct Callable:ICallable<R,Args...> {
      F f;
      template<class...Ts>
      explicit Callable(Ts&&...ts):f(std::forward<Ts>(ts)...) {}
      
      ~Callable() final = default;
      R invoke(Args&&...args) final {
        return f(std::forward<Args>(args)...);
      }
    };
    template<class Sig>
    struct task;
    template<class R, class...Args>
    struct task<R(Args...)> {
      std::unique_ptr<ICallable> pImpl;
      template<class F>
      task( F&& f ):
        pImpl( std::make_unique<Callable<std::decay_t<F>, R, Args...>>( std::forward<F>(f) ) )
      {}
      R operator()(Args...args)const {
        return pImpl->invoke( std::forward<Args>(args)... );
      }
      explicit operator bool() const { return pImpl != nullptr; }
    };

