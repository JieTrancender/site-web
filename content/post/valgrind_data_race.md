---
title: "解决数据竞争-Valgrind"
date: 2025-01-08T17:19:05+08:00
tags: ["C++", "valgrind", "thread", "游戏后端"]
categories: ["C++", "valgrind", "thread", "游戏后端"]
toc: false
draft: false
---

#### 一、背景
数据竞争在多线程编程中容易出现且难以察觉，尤其是项目复杂变量变多之后。靠人力阅读代码心智负担太大，这时候引入**valgrind**工具就可以帮我们很容易发现数据竞争从而解决问题。
<!--more-->

#### 二、数据竞争测试

1. 创建工作线程文件`worker.hpp`并输入以下内容
```
#pragma once

#include <thread>
#include <functional>

class Worker
{

public:
    using ThreadFunction = std::function<void()>;

    Worker(ThreadFunction func)
    {
        thread_ = new std::thread(func);
    }
    ~Worker()
    {
        if (thread_->joinable())
        {
            thread_->join();
        }
    }
    
    void WaitForExit()
    {
        if (thread_->joinable())
        {
            thread_->join();
        }
    }
    
private:
    std::thread* thread_;
};

```

2. 创建测试代码文件`valgrind_test.cpp`并输入以下内容
```
#include "worker.hpp"

#include <iostream>
#include <vector>
#include <mutex>
#include <memory>
#include <chrono>
#include <atomic>

using namespace std;

int g_count = 0;
// 线程同步工具，保护共享资源
mutex g_mutex;
mutex log_mutex;
atomic_bool flag = false;


void TestMutexFunc()
{
    while (true)
    {
        if (flag)
        {
            break;
        }
        
        this_thread::sleep_for(chrono::microseconds(1));
    }
    
    int cur_count;
    {
        for (int i = 0; i < 1000; ++i)
        {
            // lock_guard<mutex> lg(g_mutex);
            ++g_count;
        }
        
        // lock_guard<mutex> lg(g_mutex);
        cur_count = g_count;
    }
    
    {
        lock_guard<mutex> log_lg(log_mutex);
        cout << "exec TestMutexFunc " << this_thread::get_id() << " count  is " << cur_count << endl;
    }
}

int main()
{
    vector<shared_ptr<Worker>> workers;
    for (int i = 0; i < 8; ++i)
    {
        workers.emplace_back(make_shared<Worker>(std::bind(TestMutexFunc)));
    }
    
    flag = true;
    
    for (auto& w : workers)
    {
        w->WaitForExit();
    }
    
    cout << "result: count  is " << g_count << endl;
}

```

3. 编译运行后的运行结果可能类似如下总和并不是8000的情况，这是因为多个不同线程在递增前拿到了同一值导致最终结果变小
```
kapi@SK-20221213WZNQ:/mnt/f/code/shine/build/test$ ./valgrind_test
exec TestMutexFunc 139965421913664 count  is 1000
exec TestMutexFunc 139965447091776 count  is 2000
exec TestMutexFunc 139965463877184 count  is 3000
exec TestMutexFunc 139965430306368 count  is 4000
exec TestMutexFunc 139965480662592 count  is 4412
exec TestMutexFunc 139965455484480 count  is 4556
exec TestMutexFunc 139965472269888 count  is 5556
exec TestMutexFunc 139965438699072 count  is 6556
result: count  is 6556
```

4. 使用**valgrind helgrind**工具启动程序能有效帮助我们找到数据竞争点
```
kapi@SK-20221213WZNQ:/mnt/f/code/shine/build/test$ valgrind --tool=helgrind ./valgrind_test
==209614== Helgrind, a thread error detector
==209614== Copyright (C) 2007-2024, and GNU GPL'd, by OpenWorks LLP et al.
==209614== Using Valgrind-3.24.0 and LibVEX; rerun with -h for copyright info
==209614== Command: ./valgrind_test
==209614==
exec TestMutexFunc 152913472 count  is 1000
==209614== ---Thread-Announcement------------------------------------------
==209614==
==209614== Thread #7 was created
==209614==    at 0x4BDD9F3: clone (clone.S:76)
==209614==    by 0x4BDE8EE: __clone_internal (clone-internal.c:83)
==209614==    by 0x4B4C6D8: create_thread (pthread_create.c:295)
==209614==    by 0x4B4D1FF: pthread_create@@GLIBC_2.34 (pthread_create.c:828)
==209614==    by 0x4857436: pthread_create_WRK (hg_intercepts.c:445)
==209614==    by 0x4858D42: pthread_create@* (hg_intercepts.c:478)
==209614==    by 0x4948328: std::thread::_M_start_thread(std::unique_ptr<std::thread::_State, std::default_delete<std::thread::_State> >, void (*)()) (in /usr/lib/x86_64-linux-gnu/libstdc++.so.6.0.30)
==209614==    by 0x10ADAD: std::thread::thread<std::function<void ()>&, , void>(std::function<void ()>&) (std_thread.h:143)
==209614==    by 0x10AABC: Worker::Worker(std::function<void ()>) (worker.hpp:14)
==209614==    by 0x10D43C: void __gnu_cxx::new_allocator<Worker>::construct<Worker, std::_Bind<void (*())()> >(Worker*, std::_Bind<void (*())()>&&) (new_allocator.h:162)
==209614==    by 0x10D1FF: void std::allocator_traits<std::allocator<Worker> >::construct<Worker, std::_Bind<void (*())()> >(std::allocator<Worker>&, Worker*, std::_Bind<void (*())()>&&) (alloc_traits.h:516)
==209614==    by 0x10CDD7: std::_Sp_counted_ptr_inplace<Worker, std::allocator<Worker>, (__gnu_cxx::_Lock_policy)2>::_Sp_counted_ptr_inplace<std::_Bind<void (*())()> >(std::allocator<Worker>, std::_Bind<void (*())()>&&) (shared_ptr_base.h:519)
==209614==
==209614== ---Thread-Announcement------------------------------------------
==209614==
==209614== Thread #9 was created
==209614==    at 0x4BDD9F3: clone (clone.S:76)
==209614==    by 0x4BDE8EE: __clone_internal (clone-internal.c:83)
==209614==    by 0x4B4C6D8: create_thread (pthread_create.c:295)
==209614==    by 0x4B4D1FF: pthread_create@@GLIBC_2.34 (pthread_create.c:828)
==209614==    by 0x4857436: pthread_create_WRK (hg_intercepts.c:445)
==209614==    by 0x4858D42: pthread_create@* (hg_intercepts.c:478)
==209614==    by 0x4948328: std::thread::_M_start_thread(std::unique_ptr<std::thread::_State, std::default_delete<std::thread::_State> >, void (*)()) (in /usr/lib/x86_64-linux-gnu/libstdc++.so.6.0.30)
==209614==    by 0x10ADAD: std::thread::thread<std::function<void ()>&, , void>(std::function<void ()>&) (std_thread.h:143)
==209614==    by 0x10AABC: Worker::Worker(std::function<void ()>) (worker.hpp:14)
==209614==    by 0x10D43C: void __gnu_cxx::new_allocator<Worker>::construct<Worker, std::_Bind<void (*())()> >(Worker*, std::_Bind<void (*())()>&&) (new_allocator.h:162)
==209614==    by 0x10D1FF: void std::allocator_traits<std::allocator<Worker> >::construct<Worker, std::_Bind<void (*())()> >(std::allocator<Worker>&, Worker*, std::_Bind<void (*())()>&&) (alloc_traits.h:516)
==209614==    by 0x10CDD7: std::_Sp_counted_ptr_inplace<Worker, std::allocator<Worker>, (__gnu_cxx::_Lock_policy)2>::_Sp_counted_ptr_inplace<std::_Bind<void (*())()> >(std::allocator<Worker>, std::_Bind<void (*())()>&&) (shared_ptr_base.h:519)
==209614==
==209614== ----------------------------------------------------------------
==209614==
==209614== Possible data race during read of size 4 at 0x1131A0 by thread #7
==209614== Locks held: none
==209614==    at 0x10A495: TestMutexFunc() (valgrind_test.cpp:36)
==209614==    by 0x10D9D8: void std::__invoke_impl<void, void (*&)()>(std::__invoke_other, void (*&)()) (invoke.h:61)
==209614==    by 0x10D9A2: std::__invoke_result<void (*&)()>::type std::__invoke<void (*&)()>(void (*&)()) (invoke.h:96)
==209614==    by 0x10D97B: void std::_Bind<void (*())()>::__call<void>(std::tuple<>&&, std::_Index_tuple<>) (functional:420)
==209614==    by 0x10D8B2: void std::_Bind<void (*())()>::operator()<, void>() (functional:503)
==209614==    by 0x10D823: void std::__invoke_impl<void, std::_Bind<void (*())()>&>(std::__invoke_other, std::_Bind<void (*())()>&) (invoke.h:61)
==209614==    by 0x10D6D4: std::enable_if<is_invocable_r_v<void, std::_Bind<void (*())()>&>, void>::type std::__invoke_r<void, std::_Bind<void (*())()>&>(std::_Bind<void (*())()>&) (invoke.h:111)
==209614==    by 0x10D5AB: std::_Function_handler<void (), std::_Bind<void (*())()> >::_M_invoke(std::_Any_data const&) (std_function.h:290)
==209614==    by 0x10DDED: std::function<void ()>::operator()() const (std_function.h:590)
==209614==    by 0x10DD95: void std::__invoke_impl<void, std::function<void ()>>(std::__invoke_other, std::function<void ()>&&) (invoke.h:61)
==209614==    by 0x10DD3E: std::__invoke_result<std::function<void ()>>::type std::__invoke<std::function<void ()>>(std::function<void ()>&&) (invoke.h:96)
==209614==    by 0x10DCDF: void std::thread::_Invoker<std::tuple<std::function<void ()> > >::_M_invoke<0ul>(std::_Index_tuple<0ul>) (std_thread.h:259)
==209614==
==209614== This conflicts with a previous write of size 4 by thread #9
==209614== Locks held: none
==209614==    at 0x10A49E: TestMutexFunc() (valgrind_test.cpp:36)
==209614==    by 0x10D9D8: void std::__invoke_impl<void, void (*&)()>(std::__invoke_other, void (*&)()) (invoke.h:61)
==209614==    by 0x10D9A2: std::__invoke_result<void (*&)()>::type std::__invoke<void (*&)()>(void (*&)()) (invoke.h:96)
==209614==    by 0x10D97B: void std::_Bind<void (*())()>::__call<void>(std::tuple<>&&, std::_Index_tuple<>) (functional:420)
==209614==    by 0x10D8B2: void std::_Bind<void (*())()>::operator()<, void>() (functional:503)
==209614==    by 0x10D823: void std::__invoke_impl<void, std::_Bind<void (*())()>&>(std::__invoke_other, std::_Bind<void (*())()>&) (invoke.h:61)
==209614==    by 0x10D6D4: std::enable_if<is_invocable_r_v<void, std::_Bind<void (*())()>&>, void>::type std::__invoke_r<void, std::_Bind<void (*())()>&>(std::_Bind<void (*())()>&) (invoke.h:111)
==209614==    by 0x10D5AB: std::_Function_handler<void (), std::_Bind<void (*())()> >::_M_invoke(std::_Any_data const&) (std_function.h:290)
==209614==  Address 0x1131a0 is 0 bytes inside data symbol "g_count"
...
```

`==209614== Possible data race during read of size 4 at 0x1131A0 by thread #7`这条日志告诉我们#7线程在读取0x1131A0地址数据的时候可能存在数据竞争，`This conflicts with a previous write of size 4 by thread #9`日志告诉我们竞争的对象是#9线程正在这个地址写数据；通过代码行定位到就是这里多个线程都在递增（先读后加一）。

取消valgrind_test.cpp代码文件中两处`// lock_guard<mutex> lg(g_mutex);`的注释，使用锁解决数据竞争问题。
编译后重新执行查看valgrind报告就没有数据竞争了。
```
kapi@SK-20221213WZNQ:/mnt/f/code/shine/build/test$ valgrind --tool=helgrind ./valgrind_test
==209847== Helgrind, a thread error detector
==209847== Copyright (C) 2007-2024, and GNU GPL'd, by OpenWorks LLP et al.
==209847== Using Valgrind-3.24.0 and LibVEX; rerun with -h for copyright info
==209847== Command: ./valgrind_test
==209847==
exec TestMutexFunc 127735360 count  is 1000
exec TestMutexFunc 152913472 count  is 2000
exec TestMutexFunc 136128064 count  is 3000
exec TestMutexFunc 144520768 count  is 4000
exec TestMutexFunc 102557248 count  is 5000
exec TestMutexFunc 119342656 count  is 6000
exec TestMutexFunc 110949952 count  is 7000
exec TestMutexFunc 94164544 count  is 8000
result: count  is 8000
==209847==
==209847== Use --history-level=approx or =none to gain increased speed, at
==209847== the cost of reduced accuracy of conflicting-access information
==209847== For lists of detected and suppressed errors, rerun with: -s
==209847== ERROR SUMMARY: 0 errors from 0 contexts (suppressed: 21098 from 7)
```
