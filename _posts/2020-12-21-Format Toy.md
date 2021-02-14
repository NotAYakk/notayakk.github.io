---
layout: post
---

title: Format Toy

This is a test of formatting.  Here is an untested toy threadsafe queue.

    template<class T>
    struct threadsafe_queue {
      T pop() {
        auto l = lock();
        cv.wait( l, [&]{return !queue.empty();} );
        auto r = queue.front();
        queue.pop_front();
        return r;
      }
      std::deque<T> pop_all() {
        auto l = lock();
        return std::move(queue);
      }
      void push( T in ) {
        auto l = lock();
        queue.push_back(std::move(in));
        cv.notify_one();
      }
    private:
      mutable std::mutex m;
      std::condition_variable cv;
      std::deque<T> queue;
      auto lock() const { return std::unique_lock(m); }
    };

you can extend it with these methods:

      template<class...Ts>
      std::optional<T> try_pop_for( std::chrono::duration<Ts...> duration ) {
        auto l = lock();
        if (!cv.wait_for(l, duration, [&]{return !queue.empty();}))
          return std::nullopt;
        auto r = queue.front();
        queue.pop_front();
        return r;
      }
      std::optional<T> try_pop() {
        auto l = lock();
        if (queue.empty())
          return std::nullopt;
        auto r = queue.front();
        queue.pop_front();
        return r;
      }
