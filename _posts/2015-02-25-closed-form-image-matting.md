---
layout: post
title: "A Closed-From Solution to Natural Image Matting"
date: 2015-02-25
comments: True
categories: "vision"
description: "This article is a summary for PAMI 2013 paper - A closed-form solution to natural image matting."
---

This article is a summary for PAMI 2013 paper - A closed-form solution to natural image matting.

<!--more-->

<hr class="soft"/>

### Introduction

#### Targets
* Extracting a foreground object from an imaeg based on limited input.

#### Challenges
* The problem is ill-posed
  * At each pixel we must estimate the foreground and the background colors, as well as the foreground opacity from a single color measurement.

#### The Matting Equation
* For an input image I, it is assumed to be composited of foreground F and background B. The color of i-th pixel is assumed to be a linear combination of the corresponding foreground and background colors:

$$
\begin{equation}
    I_i = \alpha_i \cdot F_i\ +\ (1 - \alpha_i) \cdot B_i
\end{equation}
$$

* The variables on the right hand side are unknown in the image, so considering about the total 3 color channels, the total number of unknowns here would be 7.

### Overview of this paper's method

#### 1. Derivation

* __Local Smoothness Assumption__
  * The matting problem is severely underconstrained, so in this paper the author made an assumption taht both __F and B are approximately constant__ oevr __a small window around each pixel__, in order to derive their solution for the gray-scale case

###### 1.1 Start from grey image

* As a result of the local smoothness assumption goes, we can express $\alpha$ as a linear function of the iamge I:

$$
\begin{align}
    \alpha_i \ &\approx \ a \cdot I_i \ +\ b,  \forall{ i \in w} \\
    & \textit{where $a = \frac{1}{ F - B}$ and $b = \frac{B}{F - B}$ }
\end{align}
$$

* So the goal is to find the alpha that minimize the cost function

$$
\begin{equation}
J(\alpha, a, b) = \sum_{j \in I} {( \sum_{i \in w_j } {(\alpha_j - a_j \cdot I_j - b_j)^2 + \epsilon \cdot {a_j}^2 } })
\end{equation}
$$

* In their implementation, they typically use windows of 3 x 3 pixels. And since windows are placed around each pixel, the window overlaps, which enables the propagation of information between neighboring pixels.

* The cost function is quadratic in $\alpha$, a, and b, with 3N unknowns for an imaeg with N pixels

* **Theorem 1** [ *which eliminaets constant a and b* ]

  * Define J($\alpha$) as $J(\alpha) = min_{a,b} J(\alpha, a, b)$
  * Then, $J(\alpha) = \alpha^{T} \cdot L \cdot \alpha$
  * Where the laplacian matrix L is an N x N matrix, whose (i, j)th entry is:

$$
\begin{equation}
\sum_{k(i,j) \in w_k} {( \delta_{ij} - \frac{1}{\mid w_k \mid} \cdot (1 + \frac{1}{ \frac{\epsilon}{\mid w_k \mid} + \sigma_k^2} \cdot (I_i - \mu_k) \cdot (I_j - \mu_k) ) ) }
\end{equation}
$$

*   * where $\delta_{ij}$ is the Kronecker delta, $\mu_k$ and $\sigma_k^2$ are the mean and variance of the intensities in the window $w_k$ around k, and $\mid w_k \mid$ is the number of pixels in this window

* **Proof of theorem 1?(Not quite understand)**

###### 1.2 Extend to color image

* To extend this method into color image, they simply replace the linear model with a 4D linear model

$$
\begin{align}
\alpha_i &\approx \sum_c \alpha^c \cdot I_i^c + b, \forall i \in w, \textit{where c sums over color channels}
\end{align}
$$

* This model relaxes the previous assumption that F and B are constant over each window. Instead, it assumes that in a small window each of F and B is a linear mixture of two colors.

  * In other words,  $F_i = \beta_i^F \cdot F_1 + (1 - \beta_i^F) \cdot F_2$, and $B_i = \beta_i^B \cdot B_1 + (1 - \beta_i^B) \cdot B_2$


* And similarly to grey-scale case, we could define the cost function as following:

$$
\begin{align}
J(\alpha, a, b) = \sum_{j \in I}{( \sum_{i \in w_j } { (\alpha_j - \sum_c{ a_j^c \cdot I_j^c} - b_j)^2} + \epsilon \cdot \sum_c{ {a_j^c}^2 }
)}
\end{align}
$$

* Thus, we could further get $J(\alpha) = \alpha^{T} \cdot L \cdot \alpha$, where L is defined as following:

$$
\begin{equation}
\sum_{k(i,j) \in w_k} {( \delta_{ij} - \frac{1}{\mid w_k \mid} \cdot (1 + (I_i - \mu_k) \cdot (\Sigma_k + \frac{\epsilon}{ \mid w_k \mid } \cdot I_3 )^{-1} \cdot ( I_j -\mu_k) ) }
\end{equation}
$$

* where $\Sigma_k$ is a 3 x 3 covariance matrix, $\mu_k$ is a 3 x 1 mean vector of the colors in a window $w_k$, and I_3 is the 3 x 3 identity matrix

###### Matting Laplacian

* The matrix L in previous section is refered as a matting laplacian.
  * Elements in each row of L sum to zero, therefore **the null space of L includes the constant vector?(Not quite understand)**

* Since the edge contrasts in the different color channels are different, by scaling the color channels appropriately, **this model is able to actually cancel the background edge?(Not quite understand)**

#### 2. User Interaction & Constraint

* User-supplied constraint on the matte may be provided via a scribble-based GUI or a trimap

* To extract an alpha matte matching we are going to solve

$$
\begin{align}
\alpha = &argmin\ \alpha^T \cdot L \cdot \alpha + \lambda \cdot (\alpha^T - b_S^T) \cdot D_S \cdot (\alpha - b_S), \\
\end{align}
$$

* where $\lambda$ is some large number $D_S$ is a diagonal matrix whose diagonal elements are one for constrained pixels and zero for all other pixels, and $b_S$ is the vector containing the specified alpha values for the constrained pixels and zero for all others


* Since the above is quadratic in alpha, the global minimum may be found by differentiating and setting the derivatives to zero, which amounts to **solving the sparse linear system:**

$$
\begin{align}
(L + \lambda \cdot D_S) \cdot \alpha = \lambda \cdot b_S
\end{align}
$$

###### 3. Optimization

* In order to solve the sparse linear system efficiently, they proposed a coarse-to-fine scheme.
  1. They dowmsample the image and the constraints, then solve it at a lower resolution
  2. The recovered alpha matte is then interpolated to the finer resolution
    2.1 Alpha values are thresholded and pixels with alpha close to zero or one are clamped and considered as contrained in the finer resolution
    2.2 Constrained pixels may be eliminated from the system, reducing the system size.[so there is no need to enforce local linear modes in constrained areas]
  3. When computing the matting laplacian matrix, sum only windows w_k that contain at least one unconstrained pixel
  4. **Clamping alpha values to zero or one is also useful in avoiding oversmoothed alpha mattes?(Not quite understand)**
  5. Side-effect: Long-thin structures may be lost

###### 4.Reconstrctuing F and B

* One approach for recontructing F and B is to solve $\alpha_i \approx \sum_c{\alpha^c \cdot I_i^c + b}$, with the optimal a and b given $\alpha$ using least squares.
* In order to extract F and B from a and b, there is an additional matting parameter taht should be recovered ($\beta$ )
* For complex foreground and background patterns, F and B can be solved using the compositing equation, introducing some explicit smoothness priors on F and B.

* Specifically, we minimize a system of the form

$$
\begin{align}
min &\sum_{i \in I}{\sum_c{ (\alpha_i \cdot F_i^c + (1 - \alpha_i) \cdot B_i^c - I_i^c)^2
+ \mid \alpha_{i_x} \mid \cdot ((F_{i_x}^c)^2 + (B_{i_x}^c)^2) + \mid \alpha_{i_y} \mid \cdot ( (F_{i_y}^c)^2 + (B_{i_y}^c)^2 ) }} \\
&\textit{where $F_{i_x}^c, F_{i_y}^c, B_{i_x}^c,B_{i_y}^c$ are the x and y derivatives of $F^c, B^c$, } \\
&\textit{and $\alpha_{i_x}, \alpha_{i_y}$ are the matte derivatives}
\end{align}
$$

* For a fixed $\alpha$, the cost function is quadratic, and its minimum may be found by solving a sparse set of linear equations

### Analysis

#### 1. Parameter Analysis

* In this section we will mainly discuss the effects of regularization term and the window size

* __1.1 Effect of regularization weight__

  * $\epsilon$ is the weight of the regularization term on a in cost function. There are two reasons for having it:
    * Numerical stability
    * Minimizing the norm of a biases the solution toward smoother alpha mattes


  * Use small $\epsilon$ value so that the sharpness of recovered matte matches the profile of the edge in the input image, but the matte also captures the image noise

* __1.2 Effect of window size__
  * Using wider windows is more stable when the color line model holds, but the chance of encountering windows that deviate from the color line model grows when the windows are larger

  * Increase computation time since the resulting system is less sparse


### Reference
[**[1]**](http://www.wisdom.weizmann.ac.il/~levina/papers/Matting-Levin-Lischinski-Weiss-PAMI.pdf)  A Closed-Form Solution to Natural Image Matting, Anat Levin, Dani Lischinski, and Yair Weiss, MIT, 2013 PAMI

[**[2]**](http://cs.brown.edu/courses/cs129/results/final/valayshah/) Natural Image Matting - Brown University CS129 Final Project
