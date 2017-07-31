---
layout: post_page
lang: fr
ref: blog
post_url: mpc-example-python
title: Exemple Python de Copra
permalink: fr/blog/mpc-example-python
---

Ceci un exmple d'utilisation de [Copra](https://github.com/vsamy/Copra) library.
Pour mieux comprendre la librarie, vous pouvez voir [ici]({{site.url}}/en/git-repository/mpc).
C'est un exemple simple écrit en C++ et basé sur le problème de locomotion des robots.
L'exemple est basé sur ce [papier](https://hal.inria.fr/inria-00390462/document).
<!--more-->

Cet exemple cherche à résoudre un problème récurrent de marche dynamique en robotique.
En locomotion humanoïde, le modèle de contrôle prédictif est utilisé pour rechercher une trajectoire adéquate du centre de mass (CoM) \\(s\\) du robot en considérant le point de moment-zéro (ZMP). 
Le ZMP est un point sur le sol qui doit rester dans le polygone défini par les pieds du robot.
Supposant que le CoM du robot reste fixe sur la verticale, la position \\(z\\) du ZMP est

$$
    z = s - \frac{h_{CoM}}{g}\ddot{s}
$$

Cette équation peut s'écrire comme un système linéaire.

$$
    \mathbf{\dot{x}} = 
    \left\[
        \begin{array}{ccc}
            0 & 1 & 0 \\\\
            0 & 0 & 1 \\\\
            0 & 0 & 0
        \end{array}
    \right\]\mathbf{x} +
    \left\[
        \begin{array}{c}
            0 \\\\
            0 \\\\
            1
        \end{array}
    \right\]u
$$

avec \\(\mathbf{x} = \left\[s\ \dot{s}\ \ddot{s}\right\]^T\\) la variable d'état
et \\(u = \dddot{x}\\) la variable de contrôle.
On peut alors discrétiser le système.

$$
    \mathbf{x}\_{k+1} = 
    \left\[
        \begin{array}{ccc}
            1 & T & \frac{T^2}{2} \\\\
            0 & 1 & T \\\\
            0 & 0 & 1
        \end{array}
    \right\]\mathbf{x}\_k +
    \left\[
        \begin{array}{c}
            \frac{T^3}{6} \\\\
            \frac{T^2}{2} \\\\
            T
        \end{array}
    \right\]u
$$

Le ZMP doit aussi être contraint par deux valeurs de référence \\(z\_{min}\\) et \\(z\_{max}\\)
ce qui nous amène à

$$
    z\_{min} \leq \[1\ 0\ -\frac{h\_{CoM}}{g}\]\mathbf{s} \leq z\_{max}
$$

qui peut s'écrire

$$
    \left\\{
        \begin{array}{rcl}
            -\[1\ 0\ -\frac{h\_{CoM}}{g}\]\mathbf{s} & \leq & -z\_{min} \\\\
            \[1\ 0\ -\frac{h\_{CoM}}{g}\]\mathbf{s}  & \leq & z\_{max}
        \end{array}
    \right.
$$

Finalement, il y a deux fonctions de coût i) un coût en trajectoire \\(\mathbf{z}^{ref}\\) avec un poids \\(Q\\) et ii) un coût en contrôle avec un poids \\(R\\).

### Le code
Tout d'abord, les headers

```python
from minieigen import *
import copra
```

puis il faut créer le système

```python
A = Matrix3d.Identity
A[0, 1] = T
A[0, 2] = T * T / 2.
A[1, 2] = T

B = Vector3d(T* T * T / 6., T * T / 2, T)

d = Vector3d.Zero
x_0 = Vector3d.Zero
ps = copra.NewPreviewSystem(A, B, d, x_0, nrStep)
```

Puis on créé les contraintes sur le ZMP

```python
E2 = MatrixXd.Zero(1, 3)
E2[0, 0] = 1
E2[0, 0] = h_CoM / g
E1 = -E2
f1 = MatrixXd.Zero(1, 1)
f2 = MatrixXd.Zero(1, 1)
f1[0, 0] << z_min 
f2[0, 0] << z_max

TrajConstr1 = copra.NewTrajectoryConstraint(E1, f1)
TrajConstr2 = copra.NewTrajectoryConstraint(E2, f2)
```

On construit les fonctions coût

```python
M = MatrixXd.Zero(1, 3)
M[0, 0] = 1
M[0, 0] = h_CoM / g

trajCost = copra.NewTrajectoryCost(M, -z_ref);
trajCost.weight(Q);
trajCost.autoSpan(); # Make the dimension consistent (z_ref size is nrSteps)

Eigen::<double, 1, 1> N, p;
N = MatrixXd(1, 1)
p = VectorXd(1)
N[0, 0] = 1;
p[0] = 0;

controlCost = copra.ControlCost(N, -p);
controlCost.weight(R);
```

On peut alors créer le mpc et résoudre

```python
controller = copra.LMPC(ps)
controller.addConstraint(TrajConstr1)
controller.addConstraint(TrajConstr2)
controller.addCost(trajCost)
controller.addCost(controlCost)

controller.solve();
```

Et enfin, récupérer les résultats

```python
trajectory = controller.trajectory()
jerk = controller.control()
com_pos = VectorXd(nrSteps)
com_vel = VectorXd(nrSteps)
com_acc = VectorXd(nrSteps)
zmp_pos = VectorXd(nrSteps)
for i in xrange(nr_steps):
    com_pos[i] = trajectory[3 * i]
    com_vel[2 * i] = trajectory[3 * i + 1]
    com_acc[2 * i] = trajectory[3 * i + 2]
    zmp_pos[i] = com_pos[i] - (h_CoM / g) * com_acc[i]
```
