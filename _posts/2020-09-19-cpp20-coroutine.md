---
layout: post
title:  "c++20 coroutine example"
date:   2020-09-19 09:23:58 +0800
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
    constexpr resumable_thing get_return_object() noexcept
    {
      return resumable_thing(std::coroutine_handle<promise_type>::from_promise(*(static_cast<U*>(this))));
    }
    constexpr auto initial_suspend() const noexcept { return std::suspend_never{}; }
    constexpr auto final_suspend() const noexcept { return std::suspend_always{}; }
  };

  template <class U>
  struct promise_type_impl : promise_type_impl_base<promise_type_impl<U>> {
    U              _value;
    constexpr void return_value(const U& value) noexcept { _value = value; }
  };

  template <>
  struct promise_type_impl<void> : promise_type_impl_base<promise_type_impl<void>> {
    constexpr void return_void() const noexcept {}
  };

  using promise_type = promise_type_impl<T>;

  constexpr explicit resumable_thing(std::coroutine_handle<promise_type> coroutine) noexcept : _coroutine(coroutine) {}

  constexpr const auto& get() const noexcept requires !std::is_void_v<T> { return _coroutine.promise()._value; }

  constexpr void get() const noexcept requires std::is_void_v<T> {}

  ~resumable_thing() noexcept
  {
    if (_coroutine) {
      _coroutine.destroy();
    }
  }

  constexpr void resume() const noexcept { _coroutine.resume(); }

  constexpr resumable_thing() = default;

  constexpr resumable_thing(resumable_thing const&) = delete;

  constexpr resumable_thing& operator=(resumable_thing const&) = delete;

  constexpr resumable_thing(resumable_thing&& other) noexcept : _coroutine(other._coroutine)
  {
    other._coroutine = nullptr;
  }

  constexpr resumable_thing& operator=(resumable_thing&& other) & noexcept
  {
    if (&other != this) {
      _coroutine       = other._coroutine;
      other._coroutine = nullptr;
    }
    return *this;
  }

  std::coroutine_handle<promise_type> _coroutine = nullptr;
};

struct Task {
  constexpr bool await_ready() const noexcept { return false; }
  constexpr void await_suspend(std::coroutine_handle<>) const noexcept {}
  constexpr int  await_resume() const noexcept { return 1; }
};

resumable_thing<void> f_void() noexcept
{
  co_await Task{};
  co_return;
}

resumable_thing<int> f_int() noexcept
{
  co_await Task{};
  co_return 1;
}

resumable_thing<int> f_suspend_always() noexcept
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

resumable_thing<int> f_suspend_never() noexcept
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
