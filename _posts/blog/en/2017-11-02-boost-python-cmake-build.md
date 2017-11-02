---
layout: post_page
lang: en
ref: blog
post_url: boost-python-cmake-build
title: Building Python bindings with CMake and Boost
permalink: en/blog/boost-python-cmake-build
---

This is a short explanation that explains how to build a boost python binding with CMake. You may or may not use [JRL CMake](https://github.com/jrl-umi3218/jrl-cmakemodules) macros or [PID](https://github.com/lirmm/pid-workspace) macros.
<!--more-->

## The lib to bind
Let's say you have a lib called `libMyLib.so` you want to bind. 
The CMake project name is defined as *MyLib*.

Let's bind the functions.

## Bindings
The binding file must be `.cpp` file, let's call it `bindings.cpp` located in a *src* folder.
The library name for python is set as **pyMyLib**.
Here is a snippet that gives the minimum needed code.

```c++
#include <boost/python.hpp>
// Include the headers of MyLib

BOOST_PYTHON_MODULE(pyMyLib)
{
    Py_Initialize();

    // Write the bindings here
}
```

If you want to use numpy with C++ (only available from boost 1.63), here is the code

```c++
#include <boost/python.hpp>
#include <boost/python/numpy.hpp>
// Include the headers of MyLib

namespace np = boost::python::numpy;

BOOST_PYTHON_MODULE(pyMyLib)
{
    Py_Initialize();
    np::initialize();

    // Write the bindings here
}
```

Don't forget to write the `__init__.py` file in the same folder as the bindings. To be able to use `__init__.py` with both python *2.7* and *3* you have to add a **.** when using the keyword **from**.

```python
from .pyMyLib import # Add class or functions
# ...
```

## Raw CMake
Let's now write the CMake that will perform the build and the installation.

First of all you need to find the Python package and Boost packages. The variable *PY_VERSION* should return either *2.7.x* either *3.x*. Here it is assumed that the boost version is *1.64.0* (change it if needed).

```cmake
find_package(PythonLibs ${PY_VERSION} REQUIRED)
find_package(Boost 1.64.0 REQUIRED COMPONENTS system python)
```

Then you need to make the compiler aware of the header files, create the library and link.

```cmake
# include directories
include_directories(${PROJECT_SOURCE_DIR}/src)
include_directories(${PYTHON_INCLUDE_DIRS})
include_directories(${Boost_INCLUDE_DIRS})

# create the lib
add_library(pyMyLib SHARED src/bindings.cpp)

# link
target_link_libraries(pyMyLib ${Boost_LIBRARIES} ${PROJECT_NAME})
```

It is **very important** that the library name (here **pyMyLib**) and the python module name (in BOOST_PYTHON_MODULE(**pyMyLib**)) are the **same**.

You now need to install the `__init__.py` and the lib.

```cmake
# Copy the __init__.py file
configure_file(__init__.py ${CMAKE_CURRENT_BINARY_DIR}/src/__init__.py COPYONLY)

# Suppress prefix "lib" because Python does not allow this prefix
set_target_properties(pyMyLib PROPERTIES PREFIX "")

install(TARGETS pyMyLib __init__.py DESTINATION "${PYTHON_INSTALL_PATH}")
```

## JRL CMake
This is pretty much the same as above. You have some macro you can use to facilitate the writings.

```cmake
# find boost and python
set(BOOST_COMPONENTS system python)
SEARCH_FOR_BOOST()
FINDPYTHON()

# Compile and install python file
PYTHON_INSTALL_BUILD(pyMyLib __init__.py "${PYTHON_INSTALL_PATH}")
```

## PID
It is a bit simpler for PID since it handle everything itself. 
In the global *CMakeLists.txt*

```cmake
get_PID_Platform_Info(PYTHON PY_VERSION)
find_package(PythonLibs ${PY_VERSION} REQUIRED)
declare_PID_Package_Dependency(PACKAGE boost EXTERNAL VERSION 1.64.0)
```

In the *CMakeLists.txt* of the *src* folder

```cmake
declare_PID_Component(
    MODULE_LIB
    NAME pyMyLib
    DIRECTORY pyMyLib
)

declare_PID_Component_Dependency(
    COMPONENT pyMyLib
    NATIVE MyLib
)

get_PID_Platform_Info(PYTHON PY_VERSION)

if(PY_VERSION VERSION_LESS 3.0)
    #using python2 to manage python wrappers
    declare_PID_Component_Dependency(
        COMPONENT pyMyLib 
        EXTERNAL boost-python 
        PACKAGE boost
    )
else()
    #using python3 to manage python wrappers
    declare_PID_Component_Dependency(
        COMPONENT pyMyLib 
        EXTERNAL boost-python3 
        PACKAGE boost
    )
endif()
```