---
layout: post_page
lang: fr
ref: repo
post_url: pygen_converter
title: pygen-converter
permalink: fr/git-repositories/pygen_converter
---

Pygen-converter est un outils permettant une conversion implicite entre la librairie C++ Eigen et la librairie Python Numpy.
C'est une lib avec uniquement des headers qui offre tous les bindings pour les types standards de Eigen (*e.g.* `MatrixXd`, `Vector3i`, `ArrayXXf`, ...) et non-standard ou définie par l'utilisateur.
<!--more-->

## De quoi c'est composé?
Il n'y a qu'un seul header `<pygen/converters.h>` et il est uniquement composé de fonctions et structures templatées.

### Structures
Les structures parlent d'elles-mêmes.
`python_list_to_eigen_vector`, `python_list_to_eigen_matrix` permet la conversion de liste Python vers des vecteurs/matrices Eigen.
`numpy_array_to_eigen_vector`, `numpy_array_to_eigen_matrix` permet la conversion de Numpy ndarray vers des vecteurs/matrices Eigen.
`eigen_vector_to_numpy_array`, `eigen_matrix_to_numpy_array` allows the conversion from Eigen vectors/matrices to Numpy ndarray.
All of these templated structures await for an Eigen matrix or array type. 
These types are of the form `Eigen::Matrix<Scalar, _rows, _cols>` or `Eigen::Array<Scalar, _rows, _cols>` where `Scalar` is a basic type (int, std::complex<float>, etc..), `_rows` is either the number of rows or Eigen::Dynamic for dynamic-sized rows and `_cols` is either the number of columns or Eigen::Dynamic for dynamic-sized columns.

### Functions
The functions are also self-explanatory. 
`convertAllVector`, `convertAllRowVector`, `convertAllMatrix` converts all Eigen global (row-)vector, matrix typedefs.
`convertAllRowArray`, `convertAllColumnArray`, `convertAllArray` converts all Eigen global row-Array, col-Array, Array typedefs.
These templated functions need an `Scalar` type (int, std::complex<float>, etc..).

`convertVector` and `convertMatrix` are two other functions that allows to handle user-defined Matrix and Vectors.
You need to provide an `Eigen::Matrix<Scalar, _rows, _cols>` or `Eigen::Array<Scalar, _rows, _cols>` as typename.

### Easy conversion
There is an enum struct called `Converters` along a function called `convert` that can be used to handle any kind of conversion for you.
The structure is made of bitflag with the following flags:
 * None (No conversion)
 * Matrix (Convert all global typedefs matrices)
 * Vector (Convert all global typedefs vectors)
 * RowVector (Convert all global typedefs row vectors)
 * Array (Convert all global typedefs arrays)
 * ColumnArray (Convert all global typedefs column arrays)
 * RowArray (Convert all global typedefs row arrays)
 * NoStandard (Convert non standard Vector6, Matrix6, Array6)
 * NoRowMatrixConversion (Matrix \| Vector)
 * AllMatrixConversion (Matrix \| Vector \| RowVector)
 * NoRowArrayConversion (Array \| ColumnArray)
 * AllArrayConversion (Array \| ColumnArray \| RowArray)
 * All (AllMatrixConversion \| AllArrayConversion \| NoStandard)

For example, to have all Matrix and Vector Conversion of Scalar `double` you just need to call

```c++
pygen::convert<double>(pygen::Converters::NoRowMatrixConversion);
```

In the case you don't want Python list to be convertible into an Eigen type, call

```c++
pygen::convert<double>(pygen::Converters::NoRowMatrixConversion, false);
```

## How-to
The minimal code to bind a C++ library is as follow

```c++
#include <boost/python.hpp>
#include <boost/python/numpy.hpp>
#include <pygen/converters.h>

namespace py = boost::python;
namespace np = boost::python::numpy;

BOOST_PYTHON_MODULE(libName)
{
    Py_Initialize();
    np::initialize();

    // Include binding code here
}
```

The easiest way to add conversion is to directly use the `convert` method has above. But there is some tricks to be aware of.

### User-defined functions
To allow user-defined conversion you can call

```c++
pygen::convertMatrix<Eigen::Matrix<double, 5, 4> >();
pygen::convertVector<Eigen::Matrix<double, 5, 1> >(false);
```

You can pass a boolean as argument if you need to allow Python list conversion (default is `true`).

### Binding structures
Let's say you need to bind a C++ structure that involves Eigen type as

```c++
struct StructToBind {
    double val;
    Eigen::MatrixXd mat;
    Eigen::Vector3f vec;
};
```

Let's create the bindings.

```c++
pygen::convertMatrix(Eigen::MatrixXd);
pygen::convertVector(Eigen::Vector3f);

py::class_<StructToBind>("StructToBind")
    .def_readwrite("val", &ToBind::val)
    .add_property("mat", 
        py::make_getter(&ToBind::mat,  py::return_value_policy<py::copy_non_const_reference>()), 
        py::make_setter(&ToBind::mat))
    .add_property("vac", py::make_getter(&ToBind::vec, 
        py::return_value_policy<py::copy_non_const_reference>()), 
        py::make_setter(&ToBind::vec));
```

### Binding specific functions
There are two types of functions that needs to be deal with.
The functions that return `const&` of Eigen type and the functions that involve reference of Eigen type.

#### Handling const& return type
Function that returns a `const&` of an Eigen type needs to be bind with a specific return policy.
Let's create a simple useless class

```c++
class ClassToBind {
public:
    ClassToBind() : mat_(Eigen::MatrixXd::Random(3, 4)) {}

    const Eigen::MatrixXd& getMat() const noexcept {
        return mat_;
    }

    Eigen::MatrixXd& getMatRef() {
        return mat_;
    }

    void setMat(Eigen::MatrixXd& mat) {
        mat = Eigen::MatrixXd::Random(3, 4);
        mat_ = mat;
    }

private:
    Eigen::MatrixXd mat_;
};
```

The bindings become

```c++
py::class_<ClassToBind>("ClassToBind", py::init<>())
    .def("get_mat", &ClassToBind::getMat, 
        py::return_value_policy<py::copy_const_reference>());
```

#### Handling functions with reference to Eigen type
Functions that return a reference is a real problem for the lib. 
You need to wrap these.
For function like `getMatRef`, the best to do is to add the property with getter and setter as for the structure above. This way, the user can not only get the matrix but also set it.

To handle `generateRandomMat` you need to wrap it.

```c++
class WrapCTP : public ClassToBind {
public:
    void setMatRef(const Eigen::MatrixXd& mat) {
        Eigen::MatrixXd& inMat = getMatRef();
        inMat = mat;
    }

    Eigen::MatrixXd wrapSetMat(Eigen::MatrixXd mat) {
        setMat(mat);
        return mat;
    }
};
```

and then bind it with

```c++
py::class_<WrapCTP, py::bases<ClassToBind> >("ClassToBind", py::init<>())
    .def("get_mat", &WrapCTP::getMat, 
        py::return_value_policy<py::copy_const_reference>())
    .def("get_mat_ref", &WrapCTP::getMatRef, 
        py::return_value_policy<py::copy_non_const_reference>())
    .def("set_mat_ref", &WrapCTP::setMatRef, 
        py::return_value_policy<py::copy_non_const_reference>())
    .def("set_mat", &WrapCTP::wrapSetMat);
```

## Pros and cons
This lib handles all conversion Numpy<->Eigen in a very simple way.
For the user, it is quite handful to use this type of conversion.
The binding is little more complex though.

The main disadvantage of this method is that it always perform a copy.
This is bad for performance and does not facilitate the bindings since you can pass parameters by reference.

On the other hand it offers the possibility for Python users to directly used the commonly used Numpy library.

If you want to have the possibility to pass parameters by reference you should use [minieigen](https://github.com/eudoxos/minieigen).
It offers the possibility to access directly Eigen class in Python.
The drawback of minieigen is to have another lib to deals with and Python user's generally prefer Numpy.