---
layout: post_page
lang: fr
ref: blog
post_url: shared-complex-object-cpp
title: Partage d'objets complexes en C++
permalink: fr/blog/shared-complex-object-cpp
---

Je propose ici une méthode basée c++14 pour partager des objects complexes entre plusieurs threads et utilisant du typage fort.
Le principal avantage est de pouvoir partager des class, des structs et plus particulièrement des containers, et ce de manière thread-safe.
<!--more-->

Hop, au travail :)

## Le container

De prime abord, il nous faut un moyen de conserver les données qui seront partagées entre les différents threads.
Il faut aussi pouvoir le faire de manière thread-safe.
De plus, on voudrais pouvoir être capable de partager (quasiment) tout type d'objet.
Un des possibles outil est sans conteste le **std::tuple** de la librairie standard qui, depuis c++14, permet récupérer une valeur par type (voir [ici](https://en.cppreference.com/w/cpp/utility/tuple/get) (en anglais)).
Il est à noter que **std::tuple** n'est en aucun thread-safe et donc, on va le placer comme attribut privé d'une classe.

Comme **std::tuple** est *variadic template*, la classe elle-même est variadic template.
Le code:

```c++
template <typename... Attributes>
class ShareData {
    static_assert(!internal::has_duplicate_type_v<Attributes...>, "All type should be different.");
    static_assert(internal::are_all_copy_assignable_v<Attributes...>, "One of the type is not copy-assignable.");
    static_assert(!internal::is_any_pointer_v<Attributes...>, "Pointers should not be copied here");

private:
    std::tuple<Attributes...> m_data;
}
```

Un des prérequis pour récupérer un élément par type et que celui-ci soit unique au sein du tuple. Ceci-est assuré par le 1er **static_assert**.
Deuxièmement, Puisque les données ont besoin d'être partagées, elles doivent être *copy-assignable* (2ème **static_assert**).
Enfin, on rejette toute copy de pointer car ceux-ci sont dangereux à partager.
Plus de détails sur ces assertions statiques seront mieux détaillées dans un prochain poste.

## Accès au tuple

Maintenant que nous avons assuré que le tuple ne contient que des types uniques, il faut permettre sont accès en lecteure et écriture.
Et puisque que l'objectif est du multithreading, un *mutex* est utilisé.
Voici ci-dessous du code montrant la partie écriture (Quasi identique à la partie lecture):

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
        std::get<std::decay_t<T>(m_values) = std::move(data);
        setData(std::forward<Args>(args)...);
    }

    void setData() noexcept {}
```

Comme vous pouvez le voir, c'est pas sorcier (pin bin binbin!).
Regardons le programme étape par étape.
Tout d'abord et avant d'écrire quelconques données, le mutex est vérouillé ce qui rend l'accès *thread-safe*.
Puis, **setData** est appelée pour copier les données de manière récursive, ce qui permet au programme de déballer les types un par un jusqu'à ce qu'il ne reste plus rien (void).
Si un type n'appartenant pas au tuple est passé, le compilateur renverra une erreur de compilation due au **std::get**.
Il est aussi possible de passer un objet par *rvalue* pour une copie plus rapide.

La signature des fonctions d'accès en lecture sont respectivement **void getValues(Args&&... args) const** et **void getData(T& data, Args&&... args) const**.
Remarquez que le mutex doit être marqué *mutable* pour que les fonctions soient *const*.

Il est aussi possible d'activer/désactiver le *noexcept* en fonction du *assign operator* du type **T**.
Notez que **\*_v\<T\>** est une notation c++17 équivalente à **\*\<T\>::value** en c++11.
Notez aussi que les *move-only objects* ne sont pas authorisés (car ça ne ferait pas sens)

Mais pourquoi vérifier le *noexcept* de l'*assign operator* et non celui du *move-assign operator* ?
Il est possible d'améliorer le principe mais si on suit les quelques lignes écrites par by Mr Sutter et Mr Stroustrup dans leur guide c++: [Make move operations noexcept](https://github.com/isocpp/CppCoreGuidelines/blob/master/CppCoreGuidelines.md#Rc-move-noexcept).

## Définition du constructor, destructor, et assign operators

Les voici:

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

La classe ne gère aucune ressource et n'a besoin d'aucune information externe pour être créée, son contructeur et destructeur sont donc par défaut.
La classe peut être *copy-construct* et *copy-assign*. 
Et dans le cas du *copy-assign*, il est préférable de vérouiller les deux classes en même temps grâce à un *defer_lock*.
Le *move-constructor* et *move-assign operator* sont supprimés car ce sont des opérations extrêmement risquéees sur des objets partagés.
Si un thread pointe vers cet objet partagé et qu'un autre thread effectue un *move*, et bien... KABOOM.

## Typage fort

Maintenant que tout est prêt, et étant donné que deux types similaires sont interdits, comment puis-je utiliser, par exemple, deux instances de **std::vector\<int\>** ? Et bien, le typage fort pardis !

```c++
struct StrongType1 : std::vector<int> {
    using std::vector<int>::vector;
};

struct StrongType2 : std::vector<int> {
    using std::vector<int>::vector;
};
```

Il est évident que le nom de ces variables laisse à désirer :)

## Fonctionnement

Le test se trouve sur [coliru](http://coliru.stacked-crooked.com/a/9da94f606120fcd1).
Un petit morceau de code pour montrer le fonctionnement (voir ci-dessous pour le **SharedData.h**):

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

## Notes importantes

Tout d'abord, il est important de toujours accéder aux données via **setValues** et **getValues** en une seule fois.
Car sinon, cela pourrait donner accès à des données incohéerentes !
Par exemple, si vous avez un thread qui écrit les données d'un seul bloc tous les *x* secondes, et que vous avez un second thread qui les lis (mais pas d'un seul bloc), vous risquez d'avoir dans votre second thread des données d'avant et d'aprés que le 1er thread soit passé.

Autre note, l'*assign operator* devrait limiter l'allocation de mémoire. Moins de temps est passé dans une section critique, meilleur le programme est.

Un des avantages de ce design est de pouvoir copier l'entièreté ou une partie des données partagées car les fonctions d'accès sont générées à la compilation.
Plus encore, les *copy-assign operator* d'objets tels que **std::vector<...>** ou encore de **Eigen::Matrix<...>** ne copie quasiment (si ce n'est uniquement) que les valeurs de leur tableau interne ce qui signifie que la copie n'est pas plus lente qu'une copie d'un simple tableau.

Dans un prochain poste, je fournirai quelques outils pour pouvoir (pré-)allouer de la mémoire pour chaque type et pouvoir copier partiellement un container.

## Amélioration en c++17
Ce genre de classe fonctionne très bien avec un thread en écriture et une multitude de threads en lecture. 
Cependant, comme le mutex bloque toutes les données tant qu'une copie n'est pas finie, chaque thread doit attendre son tour.
Heureusement, c++17 fournit un *shared_mutex* qui permet à plusieurs thread en lecture d'accéder aux données en même temps.
De ce fait, on peut modifier le **std::mutex** en **std::shared_mutex** et modifier le getter par:

```c++
template <typename... Args>
void getValues(Args&&... values)
{
    std::shared_lock<std::shared_mutex> lock(m_mutex);
    setData(std::forward<Args>(values)...);
}
```

où le *shared_lock* permet plusieurs lectures en même temps.

## Appendix: Code complet et teste

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