---
layout: post_page
lang: fr
ref: repo
post_url: pygen_converter
title: pygen-converter
categories: [fr, repo]
---

pygen-converter est un outils permettant une conversion implicite entre la librairie C++ Eigen et la librairie Python Numpy.
C'est une lib avec uniquement des headers qui offre tous les bindings pour les types standards de Eigen (*e.g.* **MatrixXd**, **Vector3i**, **ArrayXXf**, ...) et non-standard ou définie par l'utilisateur.
<!--more-->

## De quoi c'est composé?
Il n'y a qu'un seul header `<pygen/converters.h>` et il est uniquement composé de fonctions et structures templatées.

### Les structures
Les structures parlent d'elles-mêmes.
`python_list_to_eigen_vector`, `python_list_to_eigen_matrix` permet la conversion de liste Python vers des vecteurs/matrices Eigen.
`numpy_array_to_eigen_vector`, `numpy_array_to_eigen_matrix` permet la conversion de Numpy ndarray vers des vecteurs/matrices Eigen.
`eigen_vector_to_numpy_array`, `eigen_matrix_to_numpy_array` permet la conversion des vecteurs/matrices Eigen vers des Numpy ndarray.
Toutes ces structures attendent comme typename un type Eigen Matrix ou Array c'est-à-dire des **Eigen::Matrix\<Scalar, _rows, _cols\>** et **Eigen::Array\<Scalar, _rows, _cols\>** où **Scalar** est un type de base (*int*, *std::complex\<float\>*, etc..), **_rows** vaut le nombre de lignes ou **Eigen::Dynamic** pour les matrices à lignes dynamiques et **_cols** vaut le nombre de colonnes ou **Eigen::Dynamic** pour matrices à colonnes dynamiques.

### Les fonctions
Les fonctions parlent aussi d'elles-mêmes.
`convertAllVector`, `convertAllRowVector`, `convertAllMatrix` convertissent les types globaux des matrices, des vecteurs et des vecteurs lignes de Eigen.
`convertAllRowArray`, `convertAllColumnArray`, `convertAllArray` convertissent la même chose mais pour les Eigen Array.
Les fonctions attendent un **Scalar** comme typename (*int*, *std::complex\<float\>*, etc..).

`convertVector` et `convertMatrix` sont deux autres fonctions qui permettent de gérer les types utilisateurs (non-standard à Eigen)
Il faut donner comme typename **Eigen::Matrix\<Scalar, _rows, _cols\>** ou **Eigen::Array\<Scalar, _rows, _cols\>**.

### Conversion simplifiée
Il y a une enum struct appelée `Converters` qui peut être utilisé avec la fonction `convert` pour gérer plusieurs types de conversions.
La structure est composée de bitflag qui sont :
 * None (Pas de  conversion)
 * Matrix (Conversion de tous les types matrices)
 * Vector (Conversion de tous les types vecteurs-colonne)
 * RowVector (Conversion de tous les types vecteurs-ligne)
 * Array (Conversion de tous les types de tableau)
 * ColumnArray (Conversion de tous les types de tableau-colonne)
 * RowArray (Conversion de tous les types de tableau-ligne)
 * NoStandard (Conversion de types non standard Vector6, Matrix6, Array6)
 * NoRowMatrixConversion (Matrix \| Vector)
 * AllMatrixConversion (Matrix \| Vector \| RowVector)
 * NoRowArrayConversion (Array \| ColumnArray)
 * AllArrayConversion (Array \| ColumnArray \| RowArray)
 * All (AllMatrixConversion \| AllArrayConversion \| NoStandard)

Par exemple, pour obtenir les conversions de toutes les matrices et vecteurs de type **double**, il faut juste appeler

```c++
pygen::convert<double>(pygen::Converters::NoRowMatrixConversion);
```

Dans le cas où la conversion des liste Python vers un type Eigen n'est pas souhaitée, il faut appeler

```c++
pygen::convert<double>(pygen::Converters::NoRowMatrixConversion, false);
```

## Utilisation
Le code minimal pour binder une librairie C++ est le suivant

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

    // Code pour le binding
}
```

La façon la plus simple d'ajouter des conversion et d'utiliser directement la méthode `convert` comme ci-dessus. Il y a cependant quelques difficultés qu'il faut connaitre.

### Conversions spécifiques
Pour ajouter des conversions spécifiques, il suffit d'appeler

```c++
pygen::convertMatrix<Eigen::Matrix<double, 5, 4> >();
pygen::convertVector<Eigen::Matrix<double, 5, 1> >(false);
```

Il est possible de passer un booléen comme argument pour activer ou non la conversion des listes Python vers des type Eigen (*true* par défaut).

### Binding des structures
Admettons qu'il faille binder une structure C++ contenant des types Eigen comme celle-ci

```c++
struct StructToBind {
    double val;
    Eigen::MatrixXd mat;
    Eigen::Vector3f vec;
};
```

Pour créer les bindings :

```c++
pygen::convertMatrix(Eigen::MatrixXd);
pygen::convertVector(Eigen::Vector3f);

py::class_<StructToBind>("StructToBind")
    .def_readwrite("val", &ToBind::val)
    .add_property("mat", 
        py::make_getter(&ToBind::mat, 
            py::return_value_policy<py::copy_non_const_reference>()),
        py::make_setter(&ToBind::mat))
    .add_property("vec", 
        py::make_getter(&ToBind::vec, 
            py::return_value_policy<py::copy_non_const_reference>()),
        py::make_setter(&ToBind::vec));
```

### Binding de fonctions spécifiques
Il y a deux types de fonctions qui posent problème lors du binding.
Les fonction qui retourne une **const&** de type Eigen et celles qui contiennent des références vers des types Eigen.

#### Gérer les retours par const&
Les fonctions qui retournent un **const&** de type Eigen ont besoin d'être bindées avec une politique particulière.
Voici une class complètement inutile à binder.

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

Le binding devient

```c++
py::class_<ClassToBind>("ClassToBind", py::init<>())
    .def("get_mat", &ClassToBind::getMat, 
        py::return_value_policy<py::copy_const_reference>());
```

#### Gérer les fonctions contenant des références de type Eigen
Les fonctions qui retournent une référence est un réel problème.
Il faut les wrapper.
Pour les fonctions telles que `getMatRef`, la meilleure façon est de d'ajouter un getter et un setter comme. De cette façon, l'utilisateur peut récupérer la matrice via le getter et la modifier via le setter.

Pour gérer `generateRandomMat` il faut wrapper la class.

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

et la binder avec 

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

Si `generateRandomMat` est une méthode statique, alors seule la fonction a besoin d'être wrappée.

```c++
static Eigen::MatrixXd wrapGenerateRandomMat(Eigen::MatrixXd mat)
{
    ClassToBind::generateRandomMat(mat);
    return mat;
}

// In the cpp bindings
py::class_<ClassToBind>("ClassToBind")
    .def("generate_random_matrix", py::make_function(wrapGenerateRandomMat));
```

### Erreur commune et slicing
L'utilisateur Python va peut-être essayé de modifier les valeurs internes du vecteur dans *StructToBind*. Hélas, ceci ne va rien modifier ce qui est contre-intuitif.

```ipython
In [2]: a.vec[1]
Out[2]: 5.0

In [3]: a.vec[1] = 8

In [4]: a.vec[1]
Out[4]: 5.0
```

C'est dû au fait que seule des copy passent entre python et c++. Le 3ème appel donne 1) Copie et converti a.vec en numpy ndarray, 2) set la 1ère valeur du vecteur copié à 8.

Pour permettre le slicing, il faut procurer des functions spécifiques

```c++
struct  WrapSTB : public StructToBind {
void set_vec(const Eigen::VectorXd& nv) {
    vec = nv;
}

void set_vec_slice(int start, Eigen::VectorXd nv) {
    if (nv.size() > vec.size() - start)
        throw std::runtime_error("Bad dimension !!!!");
    vec.segment(start, nv.size()) = nv;
}

void set_vec_splice_from_val(int index, double val) {
    if (index >= vec.size())
        throw std::domain_error("out of bounds");
    vec(index) = val;
}
};

py::class_<WrapSTB>("StructToBind", py::no_init)
 .def_readonly("vec", &WrapSTB::vec,    
    py::return_value_policy<py::copy_non_const_reference>())
 .def("set_vec", &WrapSTB::set_vec)
 .def("set_vec", &WrapSTB::set_vec_slice)
 .def("set_vec", &WrapSTB::set_vec_splice_from_val);
```

## Pours et contres
La lib gère toutes les conversions Numpy<->Eigen de manière très simple.
Du point de bue de l'utilisateur, c'est très pratique d'avoir ce genre de conversions automatiques. Les bindings sont par contre plus durs à mettre en place.

Le plus gros désaventage et qu'elle fait des copies.
De ce fait, les performances en prennent un coup et il n'y a pas d'accès direct aux matrices Eigen.

D'un autre côté, elle offre la possibilité aux utilisateurs Python d'utiliser directement la library Numpy.

Pour parer aux problèmes de performances et de gestion des références, il est préférable d'utiliser [minieigen](https://github.com/eudoxos/minieigen).
Elle permet l'accès directe aux matrices Eigen dans Python.
La contrepartie est la nécessité d'utiliser une nouvelle librairie. Les utilisateurs Python ont généralement une meilleure connaissance de la librairie Numpy.