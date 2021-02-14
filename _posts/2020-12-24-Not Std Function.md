---
layout: post
---

title: Function View


Type erasure based polymorphism example.

    template<class Sig>
    struct function_view;
    
    template<class R, class...Args>
    struct function_view<R(Args...)> {
      void const* pstate = nullptr;
      void(*pf)(void const*, Args&&...) = nullptr;
      
      explicit operator bool() const{ return pf!=nullptr; }
      
      R operator()(Args...args)const{
        return pf(pstate, std::forward<Args>(args)...);
      }
      
      template<class F,
        std::enable_if_t<!std::is_same_v<std::decay_t<F>, function_view> &&
          std::is_convertible_v<std::invoke_result_t<F const&, Args...>, R>,
          bool> = true
      >
      function_view(F&& f):
        pstate(std::addressof(f)),
        pf(+[](void const* pvoid, Args&&...args)->R{
          auto ptr = (F const*)pvoid;
          return (*ptr)(std::forward<Args>(args)...);
        })
      {}
      function_view(function_view const&)=default;
      function_view& operator=(function_view const&)=default;
    };
    
this is a standard layout, non-owning `std::function<R(Args...)>`.
If your `std::function` does not need to persist past the function call, this can be more efficient and also more DLL-safe.
