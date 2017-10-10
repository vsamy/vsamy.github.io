---
layout: post_page
lang: fr
ref: repo
post_url: wrench_cone_lib
title: wrench-cone-lib
permalink: fr/git-repository/wrench-cone-lib
---

Cette librairie est un outil simple et efficace pour faire le calcul du cone des torseurs comprennent plusieurs surfaces de contacts.

Elle permet de calculer les rayons (representation par vertice) de polyhèdre et trouve les demi-plans associés (representation par demi-plans) en utilisant un algorithme de double description.

Cette lib est basée sur [cdd (en)](https://www.inf.ethz.ch/personal/fukudak/cdd_home/) de Fukuda et une petite [lib](https://github.com/vsamy/eigen-cdd) wrappée avec eigen et threadsafe.
<!--more-->

La definition du **C**one des **T**orseurs de **C**ontacts (CWC pour **C**ontact **W**rench **C**one) a été écrite dans [ce papier](https://scaron.info/papers/journal/caron-tro-2016.pdf). Je vais ré-expliquer rapidement ça définition et donner plus de détails sur le fonctionnement de la librairie.

## Définition
Le torseur des contacts au point $O$ est défini par

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

avec $\mathbf{f}_i$ la force au contact $i$ et $\mathbf{r}_i = \overrightarrow{OC}$ est le vecteur de translation du point $O$ au contact $C$ dans les coordonnées monde.

Le cone des torseurs de contacts est sa représentation polyhédrale où la force $\mathbf{f}_i$ est le cone de frottement.

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

where $g$ is a cone generator and $\lambda \geq 0$ is the force magnitude. 
For one cone at contact point $\mathbf{p}_i$, the wrench matrix is given by

$$
G_i=
\begin{bmatrix}
    \mathbf{g}_1 & ... & \mathbf{g}_{N_G} \\
    \mathbf{p}_i\times\mathbf{g}_1 & ... & \mathbf{p}_i\times\mathbf{g}_{N_G} \\
\end{bmatrix}
$$

For a surface of $N_p$ points with $N_G$ cone generators, the contact force can be decomposed in $N_p*N_G$ variables.

$$
G^S=
\begin{bmatrix}
    G_1 & ... & G_{N_p} \\
\end{bmatrix}
$$

Letting $N_S$ be the number of surfaces, the vertex representation of the wrench cone is the concatenation of all those vectors, thus, the function returns the matrix

$$
\begin{equation}
G=
\begin{bmatrix}
    G^1 & ... & G^{N_S} \\
\end{bmatrix}
\end{equation}
$$

Finally, the latter method return the $\mathcal{H}$-representation using a double-description method with cdd.

## Examples
You can find a C++ example [here]({{site.url}}/en/blog/wcl-example-cpp), and Python example [here]({{site.url}}/en/blog/wcl-example-python).
