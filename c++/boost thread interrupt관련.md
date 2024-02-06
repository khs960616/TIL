## thread interrupt (boost)
boost:thread에서, interrupt()호출을 통해, 스레드에 인터럽트를 걸 수 있다. 
interrupt를 호출하면, 스레드를 관리하는 구조체의 한 플래그를 설정할 뿐,
실제로 하드웨어 interrupt가 걸리는 것 처럼 동작하지는 않는다.

**boost:this_thread::interrupt**

```c++
#if defined BOOST_THREAD_PROVIDES_INTERRUPTIONS
    void thread::interrupt()
    {
        detail::thread_data_ptr const local_thread_info=(get_thread_info)();
        if(local_thread_info)
        {
            lock_guard<mutex> lk(local_thread_info->data_mutex);
            local_thread_info->interrupt_requested=true;
            if(local_thread_info->current_cond)
            {
                boost::pthread::pthread_mutex_scoped_lock internal_lock(local_thread_info->cond_mutex);
                BOOST_VERIFY(!posix::pthread_cond_broadcast(local_thread_info->current_cond));
            }
        }
    }
```
1. thread info에 대한 lock을 잡고 (실제 interrupt를 거는 주체들은 사용에 따라 다르겠으나, 높은 확률로 여러 다른 스레드에서 호출 가능하므로)
2. interrupt_requested를 true로 변경
3. 로컬 스레드 정보에서 조건변수 current_cond (pthread_cond_t)를 확인한다.
4. 만약 조건 변수가 설정되어있다면, 조견변수에 대한 lock을 잡고, pthread_cond_broadcast를 호출하여 스레드를 깨운다. 

```참고
typedef struct _pthread_cond {  /* = cond_t in synch.h */
 struct {
  uint8_t  __pthread_cond_flag[4];
  uint16_t  __pthread_cond_type;
  uint16_t  __pthread_cond_magic;
 } __pthread_cond_flags;
 upad64_t __pthread_cond_data;
} pthread_cond_t;
```




boost에서 predefined된 interruption points 관련 함수들 (docs참고: https://www.boost.org/doc/libs/1_78_0/doc/html/thread/thread_management.html#thread.thread_management.thread)
```c++
boost::thread::join()
boost::thread::timed_join()
boost::thread::try_join_for(),
boost::thread::try_join_until(),
boost::condition_variable::wait()
boost::condition_variable::timed_wait()
boost::condition_variable::wait_for()
boost::condition_variable::wait_until()
boost::condition_variable_any::wait()
boost::condition_variable_any::timed_wait()
boost::condition_variable_any::wait_for()
boost::condition_variable_any::wait_until()
boost::thread::sleep()
boost::this_thread::sleep_for()
boost::this_thread::sleep_until()
boost::this_thread::interruption_point()
```
다음의 함수들은 만약, 스레드의 인터럽트가 걸린 경우 thread_interrupted 예외를 던진다. 

**boost::this_thread::interruption_point**
```c++
        void interruption_point()
        {
#ifndef BOOST_NO_EXCEPTIONS
            boost::detail::thread_data_base* const thread_info=detail::get_current_thread_data();
            if(thread_info && thread_info->interrupt_enabled)
            {
                lock_guard<mutex> lg(thread_info->data_mutex);
                if(thread_info->interrupt_requested)
                {
                    thread_info->interrupt_requested=false;
                    throw thread_interrupted();
                }
            }
#endif
        }
```


#### Ref
https://github.com/boostorg/thread/blob/develop/include/boost/thread/pthread/thread_data.hpp

posix thread관련 data 정의 // line 112부터 

-> 차후 필요할때 다시한번 체크하자.

```c++
 struct BOOST_THREAD_DECL thread_data_base:
            enable_shared_from_this<thread_data_base>
        {
            thread_data_ptr self;
            pthread_t thread_handle;
            boost::mutex data_mutex;
            boost::condition_variable done_condition;
            bool done;
            bool join_started;
            bool joined;
            boost::detail::thread_exit_callback_node* thread_exit_callbacks;
            std::map<void const*,boost::detail::tss_data_node> tss_data;

//#if defined BOOST_THREAD_PROVIDES_INTERRUPTIONS
            // These data must be at the end so that the access to the other fields doesn't change
            // when BOOST_THREAD_PROVIDES_INTERRUPTIONS is defined.
            // Another option is to have them always
            pthread_mutex_t* cond_mutex;
            pthread_cond_t* current_cond;
//#endif
            typedef std::vector<std::pair<condition_variable*, mutex*>
            //, hidden_allocator<std::pair<condition_variable*, mutex*> >
            > notify_list_t;
            notify_list_t notify;

//#ifndef BOOST_NO_EXCEPTIONS
            typedef std::vector<shared_ptr<shared_state_base> > async_states_t;
            async_states_t async_states_;
//#endif
//#if defined BOOST_THREAD_PROVIDES_INTERRUPTIONS
            // These data must be at the end so that the access to the other fields doesn't change
            // when BOOST_THREAD_PROVIDES_INTERRUPTIONS is defined.
            // Another option is to have them always
            bool interrupt_enabled;
            bool interrupt_requested;
//#endif
            thread_data_base():
                thread_handle(0),
                done(false),join_started(false),joined(false),
                thread_exit_callbacks(0),
//#if defined BOOST_THREAD_PROVIDES_INTERRUPTIONS
                cond_mutex(0),
                current_cond(0),
//#endif
                notify()
//#ifndef BOOST_NO_EXCEPTIONS
                , async_states_()
//#endif
//#if defined BOOST_THREAD_PROVIDES_INTERRUPTIONS
                , interrupt_enabled(true)
                , interrupt_requested(false)
//#endif
            {}
            virtual ~thread_data_base();

            typedef pthread_t native_handle_type;

            virtual void run()=0;
            virtual void notify_all_at_thread_exit(condition_variable* cv, mutex* m)
            {
              notify.push_back(std::pair<condition_variable*, mutex*>(cv, m));
            }

//#ifndef BOOST_NO_EXCEPTIONS
            void make_ready_at_thread_exit(shared_ptr<shared_state_base> as)
            {
              async_states_.push_back(as);
            }
//#endif
        };
``` 


