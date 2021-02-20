---
layout: post
published: false
title: Simple Thread Pool
---

The C++11 threading primitives are great, but they are only primitives.  Using them "raw" is extremely hard to get right.

A common utility is a threadpool.  This manages a set of threads, maybe based off of your hardware, that you can submit tasks to, get results from,
and the threads are recycled.  It manages a queue of incoming tasks, and works on them in rough sequence.

For an API:

    struct thread_pool {
      template<class R, class F>
      std::future<R> run_task( F f );
      void start_thread(std::size_t N);
      explicit thread_pool(std::size_t threads);
      // for shutdown:
      void abort_pending_tasks();
      void refuse_future_tasks();
      ~thread_pool();
      thread_pool(thread_pool const&)=delete;
      thread_pool(thread_pool &&)=delete;
      thread_pool& operator=(thread_pool const&)=delete;
      thread_pool& operator=(thread_pool &&)=delete;
    };
you can get fancier by tagging the tasks that go in, and deducing the return value of the functions you pass in, but we'll start with that.

We'll need some threads, and a queue of tasks to work on.

      threadsafe_queue<some_task_type> tasks;
      std::vector<std::thread> threads;
      bool refuse_all_tasks = false;
for our task type, we'll start with `std::function<void()>`, but I'll show you why that doesn't work.

Next, deducing `R` is easy:

      template<class F, class R=std::invoke_result_t<F&>>
      std::future<R> run_task( F f );
Time to fill in the method bodies.

      template<class F, class R=std::invoke_result_t<F&>>
      std::future<R> run_task( F f ) {
        if (refuse_all_tasks)
          return {};
        std::packaged_task<R()> task(std::move(f));
        auto retval = tasks.get_future();
        tasks.push_back(std::move(task));
        return retval;
      }
        
      void start_thread(std::size_t N=1) {
        for (std::size_t i = 0; i < N; ++i) {
          threads.emplace_back([this]{
            while (auto next = tasks.wait_and_pop()) {
              (*next)();
            }
          });
        }
      }
        
      explicit thread_pool(std::size_t threads) {
        start_thread(threads);
      }
      // for shutdown:
      void abort_pending_tasks() {
        tasks.clear_queue();
      }
      void refuse_future_tasks() {
        refuse_all_tasks = true;
      }
this leaves the queue unimplemented.

Now, `some_task_type` if it is `std::function<void()>` this won't compile!  `std::promised_task<R()>` is move-only, and function requires that its contents be copyable.

The easist replacement is `std::packaged_task<void()>`, which amusingly can be constructed from a `std::packaged_task<void()>`, but that is a bit heavy weight.

We really want a `std::function` that only supports move only types.

Now the thread safe queue.

    template<class T>
    struct thread_safe_queue {
      T wait_and_pop();
      std::optional<T> wait_for_and_pop(std::chrono::milliseconds);
      std::optional<T> try_pop();
      std::deque<T> pop_all();
      void push(T);
      void clear_queue();
    };
I'll cover that in another post.
