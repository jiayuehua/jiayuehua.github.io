---
layout: post
title:  "c++20 coroutine example"
date:   2020-09-15 09:23:58 +0800
categories: jekyll update
---
coroutine 作为C++20的新特性，在编程有着重要的应用。以下我们示范了如何使用coroutine，resumeable 类型和co_await able类型如何编写，以及coroutine中callee 和caller如何手动切换上下文。
```cpp
#include <coroutine>
#include <iostream>
#include <type_traits>
template <class T>
struct resumable_thing {
  template <class U>
  struct promise_type_impl_base {
    resumable_thing get_return_object() noexcept
    {
      return resumable_thing(std::coroutine_handle<promise_type>::from_promise(*(static_cast<U*>(this))));
    }
    constexpr auto initial_suspend() noexcept { return std::suspend_never{}; }
    constexpr auto final_suspend() noexcept { return std::suspend_always{}; }
  };
  template <class U>
  struct promise_type_impl : promise_type_impl_base<promise_type_impl<U>> {
    U    _value;
    void return_value(const U& value) noexcept { _value = value; }
  };
  template <>
  struct promise_type_impl<void> : promise_type_impl_base<promise_type_impl<void>> {
    constexpr void return_void() noexcept {}
  };
  using promise_type = promise_type_impl<T>;

  constexpr explicit resumable_thing(std::coroutine_handle<promise_type> coroutine) noexcept : _coroutine(coroutine) {}
  constexpr const auto& get() const requires !std::is_void_v<T> { return _coroutine.promise()._value; }
  constexpr void        get() const requires std::is_void_v<T> {}
  ~resumable_thing()
  {
    if (_coroutine) {
      _coroutine.destroy();
    }
  }
  void resume() noexcept { _coroutine.resume(); }
  constexpr resumable_thing()                       = default;
  constexpr resumable_thing(resumable_thing const&) = delete;
  constexpr resumable_thing& operator=(resumable_thing const&) = delete;
  resumable_thing(resumable_thing&& other) : _coroutine(other._coroutine) { other._coroutine = nullptr; }
  resumable_thing& operator=(resumable_thing&& other) &
  {
    if (&other != this) {
      _coroutine       = other._coroutine;
      other._coroutine = nullptr;
    }
  }

  std::coroutine_handle<promise_type> _coroutine = nullptr;
};
struct Task {
  constexpr bool await_ready() noexcept { return false; }
  constexpr void await_suspend(std::coroutine_handle<>) noexcept {}
  constexpr int  await_resume() noexcept { return 1; }
};
resumable_thing<void> f_void()
{
  co_await Task{};
  co_return;
}
resumable_thing<int> f_int()
{
  co_await Task{};
  co_return 1;
}
resumable_thing<int> f_suspend_always()
{
  struct A {
    ~A() { std::cout << "exit f_suspend_always\n"; }
  };
  A a;
  co_await std::suspend_always{};
  std::cout << "suspend_always 1\n";
  co_await std::suspend_always{};
  std::cout << "suspend_always 2\n";
  co_return 1;
}
resumable_thing<int> f_suspend_never()
{
  struct A {
    ~A() { std::cout << "exit f_suspend_always\n"; }
  };
  A a;
  co_await std::suspend_never{};
  std::cout << "suspend_never 1\n";
  co_await std::suspend_never{};
  std::cout << "suspend_never 2\n";
  co_return 1;
}
int main()
{
  {
    auto r = f_int();
    std::cout << "main\n";
    r.resume();
    std::cout << r.get() << std::endl;
  }
  {
    auto r = f_void();
    std::cout << "main\n";
    r.resume();
    r.get();
  }
  {
    auto r = f_suspend_always();
    std::cout << "main A\n";
    r.resume();
    std::cout << "main B\n";
    std::cout << r.get() << std::endl;
  }
  {
    auto r = f_suspend_never();
    std::cout << "main A\n";
    std::cout << "main B\n";
    std::cout << r.get() << std::endl;
  }
}
```
