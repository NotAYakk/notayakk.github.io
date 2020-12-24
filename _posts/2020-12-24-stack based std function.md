---
layout: post
title: Fixed size std function
---

Sometimes you want to avoid heap allocation in your vocabulary types.  Maybe you are on an embedded system with no heap, or maybe you just don't want accidental overhead.

    template<class Sig, std::size_t Size=sizeof(void*)*3, std::size_t Align=alignof(void*)>
    struct stack_function;

    template<class R, class...Args, std::size_t Bytes, std::size_t Align>
    struct stack_function<R(Args...), Bytes, Align> {
      std::aligned_storage_t<Bytes, Align> m_bytes;
      R(*m_action)(void*, Args&&...args) = nullptr;
      
      template<class F,
        std::enable_if_t<
          alignof(F)<=Align && sizeof(F)<=Bytes
          && std::is_convertible_v<std::invoke_result_t<F&, Args...>, R>,
        bool = true
      >
      stack_function( F&& f ) {
        ::new((void*)&m_bytes) std::decay_t<F>( std::forward<F>(f) );
        m_action = [](void* pvoid, Args&&...args)->R {
          std::decay_t<F>* pf = static_cast<std::decay_t<F>*>(pvoid);
          return (*pf)(std::forward<Args>(args)...);
        };
      }
      R operator()(Args...args) {
        return m_action(&m_bytes, std::forward<Args>(args)...);
      }
      explicit operator bool()const{return m_action!=nullptr;}

TODO
