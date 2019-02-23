---
layout: post_page
lang: fr
ref: blog
post_url: boost-python-cmake-build
title: Compiler des binding Python avec CMake et Boost
categories: [fr, blog]
---

Ceci une courte notice pour expliquer comment compiler des binding python avec Boost et CMake. Il possible (ou non) d'utiliser les macros du [CMake du JRL](https://github.com/jrl-umi3218/jrl-cmakemodules) ou celles de [PID](https://github.com/lirmm/pid-workspace).
<!--more-->

## La lib à binder
Imaginons qu'on veuille binder une librairie appelée `libMyLib.so`.
Le nom du projet CMake est alors *MyLib*.

Commençons par binder les fonctions.

## Bindings
Le fichier pour binder doit être un fichier `.cpp`, il est appelé ici `bindings.cpp` et est situé dans le dossier *src*.
On choisit comme nom pour la librairie python **pyMyLib**
Voici le code minimal à avoir

```c++
#include <boost/python.hpp>
// Inclure les headers de MyLib

BOOST_PYTHON_MODULE(pyMyLib)
{
    Py_Initialize();

    // Ecrire les bindings ici
}
```

Si il y a besoin d'utiliser Numpy avec C++ (à partir de Boost 1.63), quelques ajouts au code est nécessaire

```c++
#include <boost/python.hpp>
#include <boost/python/numpy.hpp>
// Inclure les headers de MyLib

namespace np = boost::python::numpy;

BOOST_PYTHON_MODULE(pyMyLib)
{
    Py_Initialize();
    np::initialize();

    // Ecrire les bindings ici
}
```

Il faut aussi écrire le fichier `__init__.py` dans le même dossier que les bindings. 
Pour pouvoir utiliser `__init__.py` avec les versions *2.7* et *3* de Python, il faut ajouter un **.** quand le mot-clé **from** est employé.

```python
from .pyMyLib import # Ajouter les class ou fonctions
# ...
```

## CMake basique
On peut maintenant écrire le CMake qui compilera et installera les bindings

Tout d'abord il faut rechercher les package Python et Boost.
La variable *PY_VERSION* devrait valoir soit *2.7.x* soit *3.x*.
On assume ici que la version de Boost désirée est *1.64.0* (à modifier si nécessaire).

```cmake
find_package(PythonLibs ${PY_VERSION} REQUIRED)
find_package(Boost 1.64.0 REQUIRED COMPONENTS system python)
```

Ensuite, il faut donner au compilateur des dossier des headers, créer la librairie et linker.

```cmake
# ajout des headers
include_directories(${PROJECT_SOURCE_DIR}/src)
include_directories(${PYTHON_INCLUDE_DIRS})
include_directories(${Boost_INCLUDE_DIRS})

# creation de la lib
add_library(pyMyLib SHARED src/bindings.cpp)

# link
target_link_libraries(pyMyLib ${Boost_LIBRARIES} ${PROJECT_NAME})
```

Il est **primordial** que le nom de la lib (ici **pyMyLib**) et le nom du module Python (dans BOOST_PYTHON_MODULE(**pyMyLib**)) **coïncident**

Il faut maintenant installer `__init__.py` et la lib.

```cmake
# Copie le fichier __init__.py
configure_file(__init__.py ${CMAKE_CURRENT_BINARY_DIR}/src/__init__.py COPYONLY)

# Supprime le prefix "lib" car Python ne permet pas ce prefix
set_target_properties(pyMyLib PROPERTIES PREFIX "")

install(TARGETS pyMyLib __init__.py DESTINATION "${PYTHON_INSTALL_PATH}")
```

## JRL CMake
Ça ne change que très peu ici. Il est possible d'utiliser quelques macros.

```cmake
# rechercher boost et python
set(BOOST_COMPONENTS system python)
SEARCH_FOR_BOOST()
FINDPYTHON()

# Compile et installe __init__.py
PYTHON_INSTALL_BUILD(pyMyLib __init__.py "${PYTHON_INSTALL_PATH}")
```

## PID
PID s'occupe de presque tout faire, il faut juste lui dire ce dont il a besoin.
Dans le *CMakeLists.txt* global

```cmake
get_PID_Platform_Info(PYTHON PY_VERSION)
find_package(PythonLibs ${PY_VERSION} REQUIRED)
declare_PID_Package_Dependency(PACKAGE boost EXTERNAL VERSION 1.64.0)
```

Dans le *CMakeLists.txt* du dossier *src*

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
    # python2
    declare_PID_Component_Dependency(
        COMPONENT pyMyLib 
        EXTERNAL boost-python 
        PACKAGE boost
    )
else()
    # python3
    declare_PID_Component_Dependency(
        COMPONENT pyMyLib 
        EXTERNAL boost-python3 
        PACKAGE boost
    )
endif()
```