---
layout: post_page
lang: en
ref: blog
post_url: copra-example-python
title: Python example of Copra
permalink: en/blog/copra-example-python
---

This is an how-to-use article of [Copra](https://github.com/vsamy/Copra) library.
To understand more on the library, you can visit [here]({{site.url}}/en/git-repository/copra).
It is a simple example written in python and base on robot locomotion problem.
The example is based on this [paper](https://hal.inria.fr/inria-00390462/document).
<!--more-->

This example aims to solve a common problem in humanoid robot dynamic walk.
In dynamic walk, a model predictive control is used to find a trajectory for the Center of Mass (CoM) $s$ of the robot considering its Zero-momentum Point (ZMP). 
The ZMP is a point on the ground which must stay inside the polygon defined by the foot of the robot. 
Supposing the CoM of the robot does not move vertically, the position $z$ of the CoP is
$$
    z = s - \frac{h_{CoM}}{g}\ddot{s}
$$

This equation can be written as a linear system.

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

with $\mathbf{x} = \left[s\ \dot{s}\ \ddot{s}\right]^T$ the state variable 
and $u = \dddot{x}$ the control variable.
From it, we can easily perform the discretization.

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

The ZMP should also be constrained between two references values $z_{min}$ and $z_{max}$
which leads to

$$
    z_{min} \leq [1\ 0\ \text{-}\frac{h_{CoM}}{g}]\mathbf{s} \leq z_{max}
$$

which is splitted into

$$
    \left\{
        \begin{array}{rcl}
            \text{-}[1\ 0\ \text{-}\frac{h_{CoM}}{g}]\mathbf{s} & \leq & \text{-}z_{min} \\
            [1\ 0\ \text{-}\frac{h_{CoM}}{g}]\mathbf{s}  & \leq & z_{max}
        \end{array}
    \right.
$$

And finally we have two cost functions i) A trajectory cost $\mathbf{z}^{ref}$ with a weight $Q$ and ii) a control cost with weight $R$.

The code
--------
First of all, the headers

```python
import numpy as np
import pyCopra as copra
```

then we need to create the system

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

Then we create the ZMP constraint 

```python
E2 = np.array([[1., 0., -h_CoM / g]])
E1 = -E2
f1 = np.array([z_min])
f2 = np.array([z_max])

traj_constr_1 = copra.NewTrajectoryConstraint(E1, f1)
traj_constr_2 = copra.NewTrajectoryConstraint(E2, f2)
```

Build the cost function

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

Create the copra and solve

```python
controller = copra.LMPC(ps)
controller.add_constraint(traj_constr_1)
controller.add_constraint(traj_constr_2)
controller.add_cost(traj_cost)
controller.add_cost(control_cost)

controller.solve();
```

Finally, get the results

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
