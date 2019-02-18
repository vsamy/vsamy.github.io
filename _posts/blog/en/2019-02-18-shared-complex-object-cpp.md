---
layout: post_page
lang: en
ref: blog
post_url: shared-complex-object-cpp
title: Sharing complex object in C++
permalink: en/blog/shared-complex-object-cpp
---

I present here a C++14 way of sharing data between threads using strong types and mutex.
The main advantage of this way is the ability to directly shared in a thread-safe way any class
and more specifically, to share containers.
<!--more-->

So let's begin!

## The container

We first need a way to hold data that will be shared between all threads.
This container should also be accessible in a thread-safe way.
We also want the possibility to be able to share (almost) any type.
A nice possible tool to use can be the **std::tuple** of STL library which, since c++14, allow us to get a value by type (see [here](https://en.cppreference.com/w/cpp/utility/tuple/get)).
Of course, **std::tuple** is not thread-safe, therefore, we put it as a private attribute of a class.

As **std::tuple** is a *variadic template*, we made the class itself a variadic template.
Here is the code:

```c++
template <typename... Attributes>
class SharedData {
    static_assert(!internal::has_duplicate_type_v<Attributes...>, "All type should be different.");
    static_assert(internal::are_all_copy_assignable_v<Attributes...>, "One of the type is not copy-assignable.");
    static_assert(!internal::is_any_pointer_v<Attributes...>, "Pointers should not be copied here");

private:
    std::tuple<Attributes...> m_data;
}
```

In order to get element from the tuple by type, we ensure that the tuple contains one and only one type of a kind. This is done thanks to the first **static_assert**.
Secondly, since the data will be shared between thread, they must be copy-assignable (second **static_assert**).
Finally, we don't want to deal with pointer because they lead to unsafe sharing.
More details on those static assert will be given in a next post.

## Accessing the tuple

Now we have ensured that the tuple will only have one and only one type of a kind, we need to give read/write access.
Since multithreading is aimed, a mutex will be used. Here is a chunk of code for the writing part (almost the same is done for reading):

```c++
public:
    template <typename... Args>
    void setValues(Args&&... values)
    {
        std::lock_guard<std::mutex> lock(m_mutex);
        setData(std::forward<Args>(values)...);
    }

private:
    template <typename T, typename... Args>
    void setData(T&& data, Args&&... args) noexcept(std::is_nothrow_copy_assignable_v<T>)
    {
        std::get<std::decay_t<T>>(m_values) = std::move(data);
        setData(std::forward<Args>(args)...);
    }

    void setData() noexcept {}
```

As you can see, it is not that complex. Let's take things one-by-one.
First and before any data is to be set, the mutex is locked so that all data can be safely written.
Then, we recursively call **setData** for each type until nothing (void) is left. This recursion allows the program to unpack the variadic template and to know what is the type of data that is to be set. 
If a data does not belong to the tuple, the compiler will complain due to the **std::get**.
We also gives the possibility to move object for faster assignment.

The signature of the read access functions are respectively **void getValues(Args&&... args) const** and **void getData(T& data, Args&&... args) const**.
Note that the mutex should be *mutable* to have const functions.

You can activate/deactivate the noexcept depending of whether or not **T**'s assign operator is noexcept.
Note that **\*_v\<T\>** is C++17 and is equivalent to **\*\<T\>::value**.
Note also that, move-only type are not allowed (it would make non-sense to share).

So what about checking the *noexcept*ness of the move-assign operator? 
You can add some extension, but as mention by M. Sutter and M. Stroustrup in their C++ guidelines: [Make move operations noexcept](https://github.com/isocpp/CppCoreGuidelines/blob/master/CppCoreGuidelines.md#Rc-move-noexcept).

## Defining constructors, destructors, assign operators

Here they are:

```c++
SharedData() = default;
~SharedData() = default;
SharedData(const SharedData& rhs)
{
    std::lock_guard<std::mutex> lock(rhs.m_mutex);
    m_values = rhs.m_values;
}
SharedData& operator=(const SharedData& rhs)
{
    std::unique_lock<std::mutex> lock1(m_mutex, std::defer_lock);
    std::unique_lock<std::mutex> lock2(rhs.m_mutex, std::defer_lock);
    std::lock(lock1, lock2);
    m_values = rhs.m_values;
    return *this;
}
SharedData(SharedData&& rhs) = delete;
SharedData& operator=(SharedData&& rhs) = delete;
```

The class does not manage any resources and does not need external information to be created, thus both the constructor and the destructor are defaulted.
The class can be copy-construct and copy-assign. In the case of the copy-assign operator, we need to lock both class at the same time, this is done with a *defer_lock*.
The move-contructor and move-assign operators are deleted because these are extremely risky operation to perform on a shared object. If a thread is pointing to this object and another thread performs a move operation, you can expect bad results :)

## Strong typing

Now, how could I pass two **std::vector\<int\>** to that class since it is prohibited to have two type of the same? Well, strong typing!

```c++
struct StrongType1 : std::vector<int> {
    using std::vector<int>::vector;
};

struct StrongType2 : std::vector<int> {
    using std::vector<int>::vector;
};
```

Please write better variable names :)
For more details i suggest reading Jonathan Boccara's [blog](https://www.fluentcpp.com/2016/12/08/strong-types-for-strong-interfaces/).

## How to

The test is found on [coliru](http://coliru.stacked-crooked.com/a/9da94f606120fcd1).
The code is quite simple (See below for the **SharedData.h**):

```c++
#include "SharedData.h"
#include <memory> // shared_ptr
#include <string>
#include <thread>
#include <vector>
#include <sstream>
#include <iostream>
#include <chrono>

using MySharedData = SharedData<StrongType1, StrongType2, std::string, double, int>;

int main()
{
    std::shared_ptr<MySharedData> sd(new MySharedData);

    auto setterFoo = [](std::shared_ptr<MySharedData> sd) {
        StrongType1 st1 = { 1, 2, 3 };
        StrongType2 st2 = { 8, 9, 10 };
        for (int i = 0; i < 10; ++i) {
            std::string s = "iter: " + std::to_string(i);
            double d = 29.5 + i;
            sd->setValues(st1, st2, std::move(s), d, int(3)); // Set all objects
            st1[1] += i;
            st2[2] += i;
            std::this_thread::sleep_for(std::chrono::milliseconds(40));
        }
    };

    auto getterFoo = [](std::shared_ptr<MySharedData> sd) {
        std::string s;
        StrongType2 v;
        for (int i = 0; i < 25; ++i) {
            std::stringstream ss;
            sd->getValues(s, v); // Don't need to get all object
            ss << s << ":\t";
            for (std::size_t j = 0; j < v.size(); ++j)
                ss << v[j] << " ";

            std::cout << ss.str() << std::endl;
            std::this_thread::sleep_for(std::chrono::milliseconds(20));
        }
    };

    auto modifierFoo = [](std::shared_ptr<MySharedData> sd) {
        std::string s;
        StrongType2 v;
        for (int i = 0; i < 10; ++i) {
            sd->getValues(v, s); // The order of the arguments is not important
            s += "_Modified";
            v.push_back(42);
            sd->setValues(s, v); // Don't need to set all object
            std::this_thread::sleep_for(std::chrono::milliseconds(30));
        }
    };

    std::thread t1(setterFoo, sd);
    std::thread t2(modifierFoo, sd);
    std::thread t3(getterFoo, sd);

    t1.join();
    t2.join();
    t3.join();

    return 0;
}

```

## Important notes

First of all, it is very important to **setValues** and **getValues** all at once.
Not doing so might lead to incoherent data. For example, if your data are all relative to a specific frame and a thread is getting these data one-by-one you might risk that the next write overwrite everything, thus, the hereby thread will get portion of data from the previous and current frame.

The assign-operator of the type should minimize memory allocation, the less time you spend in a critical section, the better.

One of the advantage of the design is that you can access all or a part of the sharing data because the read/write access functions are generated at compile-time.
Also, copy-assign operator of objects like **std::vector<...>** or **Eigen::Matrix<...>** mostly (if not only) copy the data of their underlying array, so it should not be much slower than a raw array and more safe (because of auto-resizing).

In the next post, I will give some more tools to handle resizing and some way to copy a subpart of a container.

## C++17 improvments
This kind of class is best use with 1 writer and several readers. 
But then, as the mutex lock all data, each thread needs to wait for the current reader to finish its operation.
Fortunately, c++17 provides a *shared_mutex* that allows several readers to run at the same time.
Then, *std::mutex* is changed with a *std::shared_mutex* and the getter is modify to be:

```c++
template <typename... Args>
void getValues(Args&&... values)
{
    std::shared_lock<std::shared_mutex> lock(m_mutex);
    setData(std::forward<Args>(values)...);
}
```

where the *shared_lock* allows several readers to lock the mutex.

## Appendix: Full code

```c++
#pragma once
#include <shared_mutex>
#include <mutex>
#include <tuple>

template <typename... Attributes>
class SharedData {
    static_assert(internal::are_all_copy_assignable_v<Attributes...>, "One of the type is not copy-assignable.");
    static_assert(!internal::has_duplicate_type_v<Attributes...>, "All type should be different.");
    static_assert(!internal::is_any_pointer_v<Attributes...>, "Pointers should not be copied here");

public:
    /*! \brief Default constructor */
    SharedData() = default;
    /*! \brief Default destructor */
    ~SharedData() = default;
    /*! \brief Copy-constructor. */
    SharedData(const SharedData& rhs)
    {
        std::lock_guard<std::shared_mutex> lock(rhs.m_mutex);
        m_values = rhs.m_values;
    }
    /*! \brief Copy-assignment. */
    SharedData& operator=(const SharedData& rhs)
    {
        std::unique_lock<std::shared_mutex> lock1(m_mutex, std::defer_lock);
        std::unique_lock<std::shared_mutex> lock2(rhs.m_mutex, std::defer_lock);
        std::lock(lock1, lock2);
        m_values = rhs.m_values;
        return *this;
    }
    /*! \brief Deleted move-constructor. */
    SharedData(SharedData&& rhs) = delete;
    /*! \brief Deleted move-assignment. */
    SharedData& operator=(SharedData&& rhs) = delete;

    /*! \brief Copy the values from the shared space.
     * \tparam Args Type of the values to copy. (Order is not relevant).
     * \param values Parameters to set the values to.
     * \exception COMPILER_EXCEPTION if the Args do not belongs to the tuple.
     */
    template <typename... Args>
    void getValues(Args&&... values) const
    {
        std::shared_lock<std::shared_mutex> lock(m_mutex);
        getData(std::forward<Args>(values)...);
    }

    /*! \brief Copy the values to the shared space.
     * \tparam Args Type of the values to copy. (Order is not relevant).
     * \param values Parameters to get the values from.
     * \exception COMPILER_EXCEPTION if the Args do not belongs to the tuple.
     */
    template <typename... Args>
    void setValues(Args&&... values)
    {
        std::lock_guard<std::shared_mutex> lock(m_mutex);
        setData(std::forward<Args>(values)...);
    }

private:
    /*! \brief Function that recursively get the data one by one.
     * \tparam T Type of the parameter to get data from.
     * \tparam Args Remaining types of parameters to get data from.
     * \param data Parameter that will receive the data.
     * \param args Remaining parameters.
     */
    template <typename T, typename... Args>
    void getData(T& data, Args&&... args) const noexcept(std::is_nothrow_copy_assignable_v<T>)
    {
        data = std::get<T>(m_values);
        getData(std::forward<Args>(args)...);
    }
    /*! \brief Recursion termination function. */
    void getData() const noexcept {}

    /*! \brief Function that recursively set the data one by one.
     * \tparam T Type of the parameter to set data to.
     * \tparam Args Remaining types of parameters to set data to.
     * \param data Parameter that will be copied.
     * \param args Remaining parameters.
     */
    template <typename T, typename... Args>
    void setData(const T& data, Args... args) noexcept(std::is_nothrow_copy_assignable_v<T>)
    {
        std::get<std::decay_t<T>>(m_values) = std::move(data);
        setData(std::forward<Args>(args)...);
    }
    /*! \brief Recursion termination function. */
    void setData() noexcept {}

private:
    mutable std::shared_mutex m_mutex; /*!< A shared mutex to ensure thread-safety */
    std::tuple<Attributes...> m_values; /*!< Tuple of copied values */
};
```