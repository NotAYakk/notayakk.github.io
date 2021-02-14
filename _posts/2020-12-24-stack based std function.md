---
layout: post
published: false
title: Fixed size std function
---


Sometimes you want to avoid heap allocation in your vocabulary types.  Maybe you are on an embedded system with no heap, or maybe you just don't want accidental overhead.

    template<class Sig, std::size_t Size=sizeof(void*)*3, std::size_t Align=alignof(void*)>
    struct stack_function;

    template<class R, class...Args>
    struct operations {
      R(*action)(void*, Args&&...args) = nullptr;
      void(*move_ctor)(void* dest, void* src) = nullptr;
      void(*copy_ctor)(void* dest, void const* src) = nullptr;
      void(*cleanup)(void*) = nullptr;
      template<class F>
      static operations make_operations() {
        operations retval;
        retval.action = [](void* pvoid, Args&&...args)->R {
          std::decay_t<F>* pf = static_cast<std::decay_t<F>*>(pvoid);
          return (*pf)(std::forward<Args>(args)...);
        };
        retval.move_ctor = [](void* pdest, void* psrc) {
          F* src = static_cast<F*>(psrc);
          ::new(pdest) F( std::move(*src) );
        };
        retval.copy_ctor = [](void* pdest, void const* psrc) {
          F const* src = static_cast<F const*>(psrc);
          ::new(pdest) F( *src );
        };
        retval.cleanup = [](void* pobj) {
          static_cast<F*>(pobj)->~F();
        };
        return retval;
      }
    };    
    template<class R, class...Args, std::size_t Bytes, std::size_t Align>
    struct stack_function<R(Args...), Bytes, Align> {
      std::aligned_storage_t<Bytes, Align> m_bytes;
      using ops = operations<R,Args...>;
      ops m_ops;
      
      template<class F,
        std::enable_if_t<
          alignof(F)<=Align && sizeof(F)<=Bytes
          && std::is_convertible_v<std::invoke_result_t<F&, Args...>, R>,
        bool = true
      >
      stack_function( F&& f ) {
        ::new((void*)&m_bytes) std::decay_t<F>( std::forward<F>(f) );
        m_ops = ops::make_operations<std::decay_t<F>>();
      }
      R operator()(Args...args) {
        return m_action(&m_bytes, std::forward<Args>(args)...);
      }
      explicit operator bool()const{return m_action!=nullptr;}
      void clear() {
        auto ops = m_ops;
        m_ops = {}; // if destructor throws, don't double-destroy
        if (ops.cleanup)
          ops.cleanup(&m_bytes);
      }
      ~stack_function() { clear(); }
      stack_function(stack_function&&o)
      {
        if (!o) return;
        o.m_ops.move_ctor(&m_bytes, &o.m_bytes);
        m_ops = o.m_ops;
        o.clear();
      }
      stack_function(stack_function const&o)
      {
        if (!o) return;
        o.m_ops.copy_ctor(&m_bytes, &o.m_bytes);
        m_ops = o.m_ops;
      }
      stack_function& operator=(stack_function&& o)
      {
        if (&o == this) return *this;
        clear();
        if (!o) return *this;
        o.m_ops.move_ctor(&m_bytes, &o.m_bytes);
        m_ops = o.m_ops;
        o.clear();
        return *this;
      }
      stack_function& operator=(stack_function const& o)
      {
        if (&o == this) return *this;
        clear();
        if (!o) return *this;
        o.m_ops.copy_ctor(&m_bytes, &o.m_bytes);
        m_ops = o.m_ops;
        return *this;
      }
    };

This should be torn apart; the copy/move erasure isn't core here, and is a lot of extra code.
