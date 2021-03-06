---
layout: post_page
lang: fr
ref: blog
post_url: copra-example-cpp
title: Exemple C++ de Copra
permalink: fr/blog/copra-example-cpp
---

Ceci un exmple d'utilisation de [Copra](https://github.com/vsamy/Copra) library.
Pour mieux comprendre la librarie, vous pouvez voir [ici]({{site.url}}/en/git-repository/copra).
C'est un exemple simple écrit en C++ et basé sur le problème de locomotion des robots.
L'exemple est basé sur ce [papier](https://hal.inria.fr/inria-00390462/document).
<!--more-->

Cet exemple cherche à résoudre un problème récurrent de marche dynamique en robotique.
En locomotion humanoïde, le modèle de contrôle prédictif est utilisé pour rechercher une trajectoire adéquate du centre de mass (CoM) $s$ du robot en considérant le point de moment-zéro (ZMP). 
Le ZMP est un point sur le sol qui doit rester dans le polygone défini par les pieds du robot.
Supposant que le CoM du robot reste fixe sur la verticale, la position $z$ du ZMP est

$$
    z = s - \frac{h_{CoM}}{g}\ddot{s}
$$

Cette équation peut s'écrire comme un système linéaire.

$$
    \mathbf{\dot{x}} = 
    \left[
        \begin{array}{ccc}
            0 & 1 & 0 \\
            0 & 0 & 1 \\
            0 & 0 & 0
        \end{array}
    \right]\mathbf{x} +
    \left[
        \begin{array}{c}
            0 \\
            0 \\
            1
        \end{array}
    \right]u
$$

avec $\mathbf{x} = \left[s\ \dot{s}\ \ddot{s}\right]^T$ la variable d'état
et $u = \dddot{x}$ la variable de contrôle.
On peut alors discrétiser le système.

$$
    \mathbf{x}_{k+1} = 
    \left[
        \begin{array}{ccc}
            1 & T & \frac{T^2}{2} \\
            0 & 1 & T \\
            0 & 0 & 1
        \end{array}
    \right]\mathbf{x}_k +
    \left[
        \begin{array}{c}
            \frac{T^3}{6} \\
            \frac{T^2}{2} \\
            T
        \end{array}
    \right]u
$$

Le ZMP doit aussi être contraint par deux valeurs de référence $z_{min}$ et $z_{max}$
ce qui nous amène à

$$
    z_{min} \leq [1\ 0\ \text{-}\frac{h_{CoM}}{g}]\mathbf{s} \leq z_{max}
$$

qui peut s'écrire

$$
    \left\{
        \begin{array}{rcl}
            \text{-}[1\ 0\ \text{-}\frac{h_{CoM}}{g}]\mathbf{s} & \leq & \text{-}z_{min} \\
            [1\ 0\ \text{-}\frac{h_{CoM}}{g}]\mathbf{s}  & \leq & z_{max}
        \end{array}
    \right.
$$

Finalement, il y a deux fonctions de coût i) un coût en trajectoire $\mathbf{z}^{ref}$ avec un poids $Q$ et ii) un coût en contrôle avec un poids $R$.

### Le code
Tout d'abord, les headers

```c++
// stl headers
#include <iostream>
#include <memory>
#include <utility>

// copra headers
#include <copra/constraints.h>
#include <copra/costFunctions.h>
#include <copra/LMPC.h>
#include <copra/PreviewSystem.h>
```

puis il faut créer le système

```c++
Eigen::Matrix3d A(Eigen::Matrix3d::Identity());
A(0, 1) = T;
A(0, 2) = T * T / 2.;
A(1, 2) = T;

Eigen::Vector3d B(T* T * T / 6., T * T / 2, T);

Eigen::Vector3d d(Eigen::Vector3d::Zero());
Eigen::Vector3d x_0(Eigen::Vector3d::Zero());
auto ps = std::make_shared<copra::PreviewSystem>(A, B, d, x_0, nrStep);
```

Puis on créé les contraintes sur le ZMP

```c++
Eigen::<double, 1, 3> E1, E2;
E2 << 1, 0, h_CoM / g;
E1 = -E2;
Eigen::<double, 1, 1> f1, f2;
f1 << z_min; 
f2 << z_max;

auto TrajConstr1 = std::make_shared<copra::TrajectoryConstraint>(E1, f1);
auto TrajConstr2 = std::make_shared<copra::TrajectoryConstraint>(E2, f2);
```

On construit les fonctions coût

```c++
Eigen::<double, 1, 3> M;
M << 1, 0, -h_CoM / g;

auto trajCost = std::make_shared<copra::TrajectoryCost>(M, -z_ref);
trajCost->weights(Q);
trajCost->autoSpan(); // Make the dimension consistent (z_ref size is nrSteps)

Eigen::<double, 1, 1> N, p;
N << 1;
p << 0;

auto controlCost = std::make_shared<copra::ControlCost>(N, -p);
controlCost->weights(R);
```

On peut alors créer le copra et résoudre

```c++
copra::LMPC controller(ps);
controller.addConstraint(TrajConstr1);
controller.addConstraint(TrajConstr2);
controller.addCost(trajCost);
controller.addCost(controlCost);

controller.solve();
```

Et enfin, récupérer les résultats

```c++
Eigen::VectorXd trajectory(controller.trajectory());
Eigen::VectorXd jerk(controller.control());
Eigen::VectorXd comPos(nrSteps);
Eigen::VectorXd comVel(nrSteps);
Eigen::VectorXd comAcc(nrSteps);
Eigen::VectorXd zmpPos(nrSteps);
for (int i = 0; i < nrSteps; ++i) {
    comPos(i) = trajectory(3 * i);
    comVel(2 * i) = trajectory(3 * i + 1);
    comAcc(2 * i) = trajectory(3 * i + 2);
    zmpPos(i) = comPos(i) - (h_CoM / g) * comAcc(i);
}
```
