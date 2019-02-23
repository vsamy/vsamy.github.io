---
layout: post_page
lang: en
ref: blog
post_url: SharedData-static-assert-cpp
title: Sharing complex object in C++ (static assert)
categories: [en, blog]
---

In a previous [post]({% post_url en/blog/2019-02-18-shared-complex-object-cpp %}), i created a class that can be used for sharing complex objects between threads.
I also omitted some details on some *static_assert* to not overload the post. We will dive into them here.
<!--more-->

The static assert i previously presented are the followings:

```c++
static_assert(!internal::has_duplicate_type_v<Attributes...>, "All type should be different.");
static_assert(internal::are_all_copy_assignable_v<Attributes...>, "One of the type is not copy-assignable.");
static_assert(!internal::is_any_pointer_v<Attributes...>, "Pointers should not be copied here");
```

They ensures that:
 - Objects are of different types
 - Objects are copy-assigneables
 - Objects are not pointers

