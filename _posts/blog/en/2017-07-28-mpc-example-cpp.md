---
layout: post_page
lang: en
ref: blog
post_url: mpc-example-cpp
title: C++ example of Copra
permalink: en/blog/mpc-example-cpp
---

This is an how-to-use article of [Copra](https://github.com/vsamy/Copra) library.
To understand more on the library, you can visit [here]({{site.url}}/en/git-repository/mpc).
It is a simple example written in C++ and base on robot locomotion problem.
The example is based on this [paper](https://hal.inria.fr/inria-00390462/document).
<!--more-->

This example aims to solve a common problem in humanoid robot dynamic walk.
In dynamic walk, a model predictive control is used to find a trajectory for the Center of Mass (CoM) \\(s\\) of the robot considering its Zero-momentum Point (ZMP). 
The ZMP is a point on the ground which must stay inside the polygon defined by the foot of the robot. 
Supposing the CoM of the robot does not move vertically, the position \\(z\\) of the CoP is
$$
    z = s - \frac{h_{CoM}}{g}\ddot{s}
$$

This equation can be written as a linear system.

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

with \\(\mathbf{x} = \left\[s\ \dot{s}\ \ddot{s}\right\]^T\\) the state variable 
and \\(u = \dddot{x}\\) the control variable.
From it, we can easily perform the discretization.

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

The ZMP should also be constrained between two references values \\(z\_{min}\\) and \\(z\_{max}\\)
which leads to

$$
    z\_{min} \leq \[1\ 0\ -\frac{h\_{CoM}}{g}\]\mathbf{s} \leq z\_{max}
$$

which is splitted into

$$
    \left\\{
        \begin{array}{rcl}
            -\[1\ 0\ -\frac{h\_{CoM}}{g}\]\mathbf{s} & \leq & -z\_{min} \\\\
            \[1\ 0\ -\frac{h\_{CoM}}{g}\]\mathbf{s}  & \leq & z\_{max}
        \end{array}
    \right.
$$

And finally we have two cost functions i) A trajectory cost \\(\mathbf{z}^{ref}\\) with a weight \\(Q\\) and ii) a control cost with weight \\(R\\).

### The code
First of all, the headers

```c++
// stl headers
#include <iostream>
#include <memory>
#include <utility>

// mpc headers
#include <mpc/constraints.h>
#include <mpc/costFunctions.h>
#include <mpc/LMPC.h>
#include <mpc/PreviewSystem.h>
```

then we need to create the system

```c++
Eigen::Matrix3d A(Eigen::Matrix3d::Identity());
A(0, 1) = T;
A(0, 2) = T * T / 2.;
A(1, 2) = T;

Eigen::Vector3d B(T* T * T / 6., T * T / 2, T);

Eigen::Vector3d d(Eigen::Vector3d::Zero());
Eigen::Vector3d x_0(Eigen::Vector3d::Zero());
auto ps = std::make_shared<mpc::PreviewSystem>(A, B, d, x_0, nrStep);
```

Then we create the ZMP constraint 

```c++
Eigen::<double, 1, 3> E1, E2;
E2 << 1, 0, h_CoM / g;
E1 = -E2;
Eigen::<double, 1, 1> f1, f2;
f1 << z_min; 
f2 << z_max;

TrajConstr1 = std::make_shared<mpc::TrajectoryConstraint>(E1, f1);
TrajConstr2 = std::make_shared<mpc::TrajectoryConstraint>(E2, f2);
```

Build the cost function

```c++
Eigen::<double, 1, 3> M;
M << 1, 0, -h_CoM / g;

trajCost = std::make_shared<mpc::TrajectoryCost>(M, -z_ref);
trajCost->weight(Q);
trajCost->autoSpan(); // Make the dimension consistent (z_ref size is nrSteps)

Eigen::<double, 1, 1> N, p;
N << 1;
p << 0;

controlCost = std::make_shared<mpc::ControlCost>(N, -p);
controlCost->weight(R);
```

Create the mpc and solve

```c++
mpc::LMPC controller(ps);
controller.addConstraint(TrajConstr1);
controller.addConstraint(TrajConstr2);
controller.addCost(trajCost);
controller.addCost(controlCost);

controller.solve();
```

Finally, get the results

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
