---
layout: post
---

title: Poly Method

Polymorphism isn't just virtual functions and inheritance.

Sometimes you have a finite list of types you want to support.  In that case, a `std::variant` is often more efficient and clearer than a polymorphic interface.

The downside is that the `std::visit` syntax can be a bit ... awkward.

What if we could do this?

    some_magic print;
    std::variant<int, std::string, double> value;
    (value->*print)(std::cout);
Well, here goes.

    template<class F>
    struct poly_method {
      F f;
      template<class Variant>
      friend auto operator->*(Variant&& var, poly_method<F> const& self)
      {
        return [&](auto&&...args)->decltype(auto)
        {
          return std::visit([&](auto&&object)->decltype(auto){
            return std::invoke( self.f, decltype(object)(object), decltype(args)(args)... );
          }, std::forward<Variant>(var));
        };
      }
    };
    template<class F>
    poly_method(F)->poly_method<F>;
Now that is a beast.

    poly_method print = {[](auto const& self, std::ostream& os){ os << self; }};
    
test code:

    std::variant<int, double, std::string> v_int = 7;
    std::variant<int, double, std::string> v_pi = 3.14;
    std::variant<int, double, std::string> v_hello = "hello world";
    (v_int->*print)(std::cout);
    (v_pi->*print)(std::cout);
    (v_hello->*print)(std::cout);
we can even make it `v_int->*print(std::cout)` by cheating.

      template<class...Args>
      struct args_first {
        F const& f;
        std::tuple<Args&&...> args;
        template<class Variant>
        friend decltype(auto) operator->*(Variant&& var, args_first&& self)
        {
          auto unpackArgs = [&](auto&&...args)->decltype(auto){
            auto applyOnObject = [&](auto&&object)->decltype(auto){
              return std::invoke( self.f, decltype(object)(object), decltype(args)(args)... );
            };
            return std::visit(applyOnObject, std::forward<Variant>(var));
          };
          return std::apply(unpackArgs, std::move(self.args));
        }
      };
      template<class...Args>
      auto operator()(Args&&...args)const
      {
        return args_first<Args...>{ f, std::tuple<Args&&...>(std::forward<Args>(args)...) };
      }
which is a lot of work for a small improvement.

https://godbolt.org/z/9EKj3e
