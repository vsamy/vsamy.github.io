---
layout: post_page
lang: en
ref: repo
post_url: wrench_cone_lib
title: wrench-cone-lib
permalink: en/git-repository/wrench_cone_lib
---

This library is little tool for computing a wrench cone of several contact surfaces.

It can compute the rays (the vertex representation) of the polyhedron and can find the halfspaces (the halfspace representation) by using the double description algorithm.

This lib is based on Fukuda's [cdd](https://www.inf.ethz.ch/personal/fukudak/cdd_home/) lib and a simple threadsafe eigen wrapper [lib](https://github.com/vsamy/eigen-cdd)
<!--more-->

The definition of the **C**ontact **W**rench **C**one (CWC) has been written in [this paper](https://scaron.info/papers/journal/caron-tro-2016.pdf). Here, i re-explain what it is and give more details of how works the library.

## Definition
The contact wrench at point $O$ is defined by

$$
\begin{equation}
    \mathbf{w}_O \overset{\text{def}}{=} 
    \begin{bmatrix}
        \mathbf{f} \\
        \mathbf{\tau}_O
    \end{bmatrix}
    \overset{\text{def}}{=} 
    \sum\limits_{\text{contact i}}
    \begin{bmatrix}
        \mathbf{f}_i \\
        \mathbf{r}_i\times\mathbf{f}_i
    \end{bmatrix}
\end{equation}
$$

with $\mathbf{f}_i$ the force at contact $i$ and $\mathbf{r}_i = \overrightarrow{OC}$ is the translation vector from point $O$ to contact $C$ in the world coordinates.

The contact wrench cone is its polyhedral representation where the force $\mathbf{f}_i$ is the friction cone.

As explain [here](https://scaron.info/teaching/contact-stability.html), 'if the CWC exists, the motion *may be* feasible'.

## Fonctionning of the lib
There are only two classes in the library. `ContactSurface` characterizes the surface involved in the computation of the CWC. `WrenchCone` computes the polyhedral representation of the contact wrench.

### Contact surface
A contact surface $S$ depends on several variables.
It holds the translation and rotation information of the surface in the world frame coordinates, the static friction coefficient $\mu$, the number of generators of the friction cone $N_G$ and the number of points $N_p$ that belongs to it.

By convention, the normal of the surface is the z-axis of the surface frame.

The linearization of the friction cone is done at run time and depends of the surface's variables. For a 3D friction cone, this number should be at least superior or equal to 3. The higher this number is, the better the cone approximation is.

The functions `rectangularSurface()` (in C++) and `rectangular_surface()` (in python) allows to build a simple rectangular surface with 4 points (one at each corner).

By default, the static friction coefficient is 0.7 and the number of cone generators is 4.

### WrenchCone
The WrenchCone class has only two functions `getRays()` and `getHalfspaces()` (in python `get_rays()` and `get_halfspaces()`).
The former compute the rays (the polyhedral vertex representation) of the wrench cone.

The force at the contact along a cone generator is simply given by

$$
\begin{equation}
    \mathbf{f}_g = \mathbf{g}\lambda
\end{equation}
$$

where $\mathbf{g}$ is a cone generator and $\lambda \geq 0$ is the force magnitude. 
For one cone at contact point $\mathbf{p}_i$, the wrench matrix is given by

$$
G_i=
\begin{bmatrix}
    \mathbf{g}_1 & ... & \mathbf{g}_{N_G} \\
    \mathbf{p}_i\times\mathbf{g}_1 & ... & \mathbf{p}_i\times\mathbf{g}_{N_G} \\
\end{bmatrix}
$$

For a surface $S$ of $N_p$ points with $N_G$ cone generators, the contact force can be decomposed in $N_p*N_G$ variables.

$$
G^S=
\begin{bmatrix}
    G_1 & ... & G_{N_p} \\
\end{bmatrix}
$$

Letting $N_S$ be the number of surfaces, the vertex representation of the wrench cone is the concatenation of all the surfaces, thus, the function returns the matrix

$$
\begin{equation}
G=
\begin{bmatrix}
    G^1 & ... & G^{N_S} \\
\end{bmatrix}
\end{equation}
$$

Finally, the latter method return the $\mathcal{H}$-representation (Halfspaces representation) using a double-description method with cdd.

## The lib in short
### Compiler option
First of all, it is important to know that there is a compiler option called `PLUCKER_NOTATION` if you want the matrices and vectors to be in Plücker notation.
The wrench then becomes

$$
\begin{equation}
    \mathbf{w}_O \overset{\text{def}}{=} 
    \begin{bmatrix}
        \mathbf{\tau}_O \\
        \mathbf{f}
    \end{bmatrix}
    \overset{\text{def}}{=} 
    \sum\limits_{\text{contact i}}
    \begin{bmatrix}
        \mathbf{r}_i\times\mathbf{f}_i \\
        \mathbf{f}_i
    \end{bmatrix}.
\end{equation}
$$

### Python users
The lib is completely bind to Python so all C++ functions exist in the Python side.
The functions' name differ from their respective C++ version to respect the PEP8 norm.
For example, the function `getRays()` becomes `get_rays()`.

Also, i have added specific bindings to facilitate the use of the function.
When a function has parameters such as `Eigen::Vector3d`, you can pass either a list `[1.,2.,3.]` or a numpy array `numpy.array([1.,2.,3.])`.
Function that asks for a `std::vector<Eigen::Vector3d>` can be a list of list or a list of numpy array.
I didn't bind numpy matrix because it is not very used.

## Examples
You can find a C++ example [here](https://github.com/vsamy/wrench-cone-lib/blob/integration/apps/WrenchConeLibPerf/example.cpp), and Python example [here](https://github.com/vsamy/wrench-cone-lib/blob/integration/share/script/pyWrenchConeLibPerf/pyPerf.py) and [here](https://github.com/stephane-caron/pymanoid/blob/master/examples/wrench_cone.py).
