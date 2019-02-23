---
layout: post_page
lang: fr
ref: blog
post_url: copra-example-python
title: Exemple Python de Copra
categories: [fr, blog]
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

```python
import numpy as np
import pyCopra as copra
```

puis il faut créer le système

```python
A = np.identity(3)
A[0, 1] = T
A[0, 2] = T * T / 2.
A[1, 2] = T

B = np.array([[T* T * T / 6.], [T * T / 2.], [T]])

d = np.zeros((3,))
x_0 = np.zeros((3,))
ps = copra.NewPreviewSystem(A, B, d, x_0, nrStep)
```

Puis on créé les contraintes sur le ZMP

```python
E2 = np.array([[1., 0., -h_CoM / g]])
E1 = -E2
f1 = np.array([z_min])
f2 = np.array([z_max])

traj_constr_1 = copra.NewTrajectoryConstraint(E1, f1)
traj_constr_2 = copra.NewTrajectoryConstraint(E2, f2)
```

On construit les fonctions coût

```python
M = np.array([[1., 0., -h_CoM / g]])

traj_cost = copra.NewTrajectoryCost(M, -z_ref);
traj_cost.weight(Q);
traj_cost.autoSpan(); # Make the dimension consistent (z_ref size is nrSteps)

N = np.array([[1.]])
p = np.array([0.])

control_cost = copra.ControlCost(N, -p);
control_cost.weight(R);
```

On peut alors créer le copra et résoudre

```python
controller = copra.LMPC(ps)
controller.add_constraint(traj_constr_1)
controller.add_constraint(traj_constr_2)
controller.add_cost(traj_cost)
controller.add_cost(control_cost)

controller.solve();
```

Et enfin, récupérer les résultats

```python
trajectory = controller.trajectory()
jerk = controller.control()
com_pos = np.array((nrSteps,))
com_vel = np.array((nrSteps,))
com_acc = np.array((nrSteps,))
zmp_pos = np.array((nrSteps,))
for i in range(nr_steps):
    com_pos[i] = trajectory[3 * i]
    com_vel[2 * i] = trajectory[3 * i + 1]
    com_acc[2 * i] = trajectory[3 * i + 2]
    zmp_pos[i] = com_pos[i] - (h_CoM / g) * com_acc[i]
```
