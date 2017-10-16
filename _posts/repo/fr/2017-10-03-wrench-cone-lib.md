---
layout: post_page
lang: fr
ref: repo
post_url: wrench_cone_lib
title: wrench-cone-lib
permalink: fr/git-repositories/wrench_cone_lib
---

Cette librairie est un outil simple et efficace pour faire le calcul du cone des torseurs en multi-contacts.

Elle permet de calculer les rayons ($\mathcal{V}$-representation) du cône et trouve les demi-plans associés ($\mathcal{H}$-representation) en utilisant un algorithme de double description.

Cette lib est basée sur [cdd (en)](https://www.inf.ethz.ch/personal/fukudak/cdd_home/) de Fukuda et son [wrapper threadsafe](https://github.com/vsamy/eigen-cdd) pour Eigen.
<!--more-->

La definition du **C**ône des **T**orseurs de **C**ontacts (CWC pour **C**ontact **W**rench **C**one) a été écrite dans [ce papier](https://scaron.info/papers/journal/caron-tro-2016.pdf). Je vais ré-expliquer rapidement sa définition et donner plus de détails sur le fonctionnement de la librairie.

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

Le cone des torseurs de contacts est la représentation polyédrale des contraintes de frottements. Il est calculé via les cônes de frottements propres à chaque point de contact $\mathbf{p}_i$.

Comme expliqué [ici](https://scaron.info/teaching/contact-stability.html), 'if the CWC exists, the motion may be feasible' (Si le CWC exists, le mouvement peut être possible).

## Fonctionnement de la lib
Il y a seulement deux classes dans cette librairie. `ContactSurface` caractérise une surface impliquée dans le calcul du CWC. `WrenchCone` calcule la representation polyédrale du torseur de contact.

### Surface de contact
Une surface de contact $S$ dépend de plusieurs variables.
Elle doit contenir : ses informations de translation et rotation dans les coordonnées du monde, son coefficient de frottement statique $\mu$, the nombre de génératrices du cône de frottement $N_G$ and le nombre de points $N_p$ lui appartenant. 

Par convention, la normale à la surface est l'axe z du repère de la surface.

La linéarisation du cône de frottement est calculée implicitement et dépend des différentes variables de la surface. Pour un cône 3D, $N_G$ devrait être au minimum égal à 3. Plus grand est ce nombre, plus précise sera l'approximation du cône.

La fonction `rectangularSurface()` (en C++) et `rectangular_surface()` (en Python) permet de construire de manière simple une surface rectangulaire contenant 4 points (un à chaque coin).

Par défaut, le coefficient de frottement statique est de 0.7 et le nombre de génératrices est de 4.

### WrenchCone
La classe WrenchCone a seulement deux fonctions : `getRays()` et `getHalfspaces()` (en Python `get_rays()` et `get_halfspaces()`).
La première calcule les rayons ($\mathcal{V}$-représentation) du cône des torseurs

La force au contact le long d'une des génératrices du cône de frottement est tout simplement

$$
\begin{equation}
    \mathbf{f}_g = g\lambda
\end{equation}
$$

où $\mathbf{g}$ est une génératrice du cône et $\lambda \geq 0$ est la magnitude de la force.
Pour un cône au point de contact $\mathbf{p}_i$, la matrice des torseurs est donnée par

$$
G_i=
\begin{bmatrix}
    \mathbf{g}_1 & ... & \mathbf{g}_{N_G} \\
    \mathbf{p}_i\times\mathbf{g}_1 & ... & \mathbf{p}_i\times\mathbf{g}_{N_G} \\
\end{bmatrix}
$$

Pour une surface $S$ à $N_p$ points avec $N_G$ génératrices de cône, la force de contact peut se décomposer en $N_p*N_G$ variables.

$$
G^S=
\begin{bmatrix}
    G_1 & ... & G_{N_p} \\
\end{bmatrix}
$$

En prenant $N_S$ le nombre de surfaces, la $\mathcal{V}$-représentation du cône des torseurs est la concatenation de toutes les surfaces, ainsi, la fonction retourne la matrice

$$
\begin{equation}
G=
\begin{bmatrix}
    G^1 & ... & G^{N_S} \\
\end{bmatrix}
\end{equation}
$$

Finalement, la seconde méthode renvoie la $\mathcal{H}$-représentation en utilisant la méthode de double description fournie par cdd.

## La lib en bref
### Option de compilation
Tout d'abord, il est important de savoir qu'il existe une option de compilation appelée `PLUCKER_NOTATION` si il y a un besoin d'avoir les matrices et vecteurs avec les notations de Plücker.
Le torseur devient alors

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

### Utilisateurs Python
La lib est complètement utilisable depuis Python, donc toutes les fonctionnalités C++ sont aussi en Python.
Les noms des fonctions diffèrent de leur version C++ pour respecter la norme PEP8.
Par example, la fonction `getRays()` devient `get_rays()`.

J'ai aussi ajouté quelques bindings spécifiques pour faciliter son utilisation en Python.
Lorsqu'une fonction contient un paramètre tel que `Eigen::Vector3d`, vous pouvez soit passé une liste `[1.,2.,3.]` soit un numpy array `numpy.array([1.,2.,3.])`.
La fonction qui a besoin d'un `std::vector<Eigen::Vector3d>` peut être soit une liste de liste soit une liste de numpy array.
Les numpy matrix ne sont pas bindées car peu utilisées.

## Exemples
Il y a un exemple C++ [ici](https://github.com/vsamy/wrench-cone-lib/blob/integration/apps/WrenchConeLibPerf/example.cpp), et un exemple Python [ici](https://github.com/vsamy/wrench-cone-lib/blob/integration/share/script/pyWrenchConeLibPerf/pyPerf.py) et [ici](https://github.com/stephane-caron/pymanoid/blob/master/examples/wrench_cone.py).