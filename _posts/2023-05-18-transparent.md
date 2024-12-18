---
layout: post
title:  "transparent，为关联容器增加查找成员"
date:   2023-05-18 08:00:01 +0800
categories: jekyll update
tags:
  container 
---

folly::transparent 模板类可以用比较器实例化，从而支持异型查找:

    #include <set>
    struct S {
      template<class T, class U>
      bool operator()(T& a, U& b) const { return a < b; }
    };
    struct Man {
      int id_;
      auto operator<=>(const Man&) const = default;
    };
    struct ManComp {
      bool operator()(const Man &l,const int& r)const
      {
        return l.id_ < r;
      }
      bool operator()(const int& l, const Man& r)const
      {
        return l < r.id_;
      }
      bool operator()(const Man& l, const Man& r)const
      {
        return l.id_ < r.id_;
      }
    };
    void f() {
      std::set<Man,ManComp> m;
      m.find(11);//将报错
    }
    void g()
    {
    std::set<Man,folly::transparent<ManComp>> m;
      m.find(11);
    }

上述f()中的m.find(11)将报错，因为find的参数只能接受Man类型，而int类型并不能隐式转化为Man，而g()将成功，因为比较器为folly::transparent一个实例，m.find()的查找重载成员将增加一成员模板函数，理论上模板函数的参数只要能和Key也就是Man比较就可。故而使用folly::transparent后成功.
