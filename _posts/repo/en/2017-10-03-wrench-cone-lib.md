---
layout: post_page
lang: en
ref: repo
post_url: wrench_cone
title: WrenchConeLib
permalink: en/git-repository/wrench-cone-lib
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

whith $\mathbf{f}_i$ is the force at contact $i$ and $\mathbf{r}_i = \overrightarrow{OC}$ is the translation vector from point $O$ to contact $C$ in the world coordinates.

The contact wrench cone is its polyhedral representation where the force $\mathbf{f}_i$ is the friction cone.

As explain [here](https://scaron.info/teaching/contact-stability.html), 'the CWC characterizes all feasible motions'.

## Fonctionning of the lib
There are only two classes in the library. `ContactSurface` characterizes the surface involved in the computation of the CWC. `WrenchCone` computes the polyhedral representation of the contact wrench.

### Contact surface
A contact surface $S$ depends on several variables.
It holds the translation and rotation information of the surface in the world frame coordinates, the static friction coefficient $\mu$, the number of generators of the friction cone $N^S_G$ and the points belonging to the surface in the surface coordinates. $N^S_p$ is the number of points.

By convention, the normal of the surface is the z-axis of the surface frame.

The linearization of the friction cone is done at run time and depends of the number of generators. For a 3D friction cone, this number should be at least superior or equal to 3. The higher this number is, the better the cone approximation is.

The functions `rectangularSurface()` (in C++) and `rectangular_surface()` (in python) allows to build a simple rectangular surface with 4 points (one at each corner).

By default, the static friction coefficient is 0.7 and the number of cone generators is 4.

### WrenchCone
The WrenchCone class has only two functions `getRays()` and `getHalfspaces()` (in python `get_rays()` and `get_halfspaces()`).
The former compute the rays (the polyhedral vertex representation) of the wrench cone.

The force at the contact along a cone generator is simply given by

$$
\begin{equation}
    \mathbf{f}_g = g\lambda
\end{equation}
$$

where $g$ is a cone generator and $\lambda \geq 0$ is the force magnitude. For a surface of 4 points with 4 cone generators, the contact force can be decomposed in $4*4=16$ variables.

Letting $M$ be the number of surfaces and $\mathbf{p}^S_i$ be the translation from $O$ to $P_i$ the i-th point of surface $S$ in world coordinates, the vertex representation of the wrench cone is the concatenation of all those vectors, thus, the function returns the matrix

$$
\begin{equation}
G=
\begin{bmatrix}
    \mathbf{g}^{1,1}_1 & ... & \mathbf{g}^{1,1}_{N^1_G} & \mathbf{g}^{1,2}_1 & ... & \mathbf{g}^{1,N^1_p}_1 & ... & \mathbf{g}^{N_S,N^M_p}_{N^S_G} \\
    \mathbf{p}^1_1\times\mathbf{g}^{1,1}_1 & ... & \mathbf{p}^1_1\times\mathbf{g}^{1,1}_{N^1_G} & \mathbf{p}^1_2\times\mathbf{g}^{1,2}_1 & ... & \mathbf{p}^1_{N^1_p}\times\mathbf{g}^{1,N^1_p}_1 & ... & \mathbf{p}^M_{N^S_p}\times\mathbf{g}^{N_S,N^M_p}_{N^S_G}
\end{bmatrix}
\end{equation}
$$