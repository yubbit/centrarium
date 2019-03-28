---
layout: post
title: "Convolution over training data with circulant matrices"
date: 2017-11-17
author: yubbit
categories: ['Mathematics', 'Computer Vision']
---

Let's say we want to train a linear classifier for the following one-dimensional "image"

$$t = \begin{bmatrix}t_1 & t_2 & t_3 & t_4 & t_5\end{bmatrix}$$

We want to train the clasifier such that it returns as positive if applied over the region $$\begin{bmatrix}t_3 & t_4 & t_5\end{bmatrix}$$, and negative otherwise. In this case, we only have three valid samples in our training data: $$\begin{bmatrix}t_1 & t_2 & t_3\end{bmatrix}$$, $$\begin{bmatrix}t_2 & t_3 & t_4\end{bmatrix}$$, and $$\begin{bmatrix}t_3 & t_4 & t_5\end{bmatrix}$$.

Training our linear model on this data would then look like:

$$
\begin{bmatrix}
t_1 & t_2 & t_3 \\
t_2 & t_3 & t_4 \\
t_3 & t_4 & t_5
\end{bmatrix}
\begin{bmatrix}x_1 \\ x_2 \\ x_3\end{bmatrix}
= \begin{bmatrix}y_1 \\ y_2 \\ y_3\end{bmatrix} \\
Tx = y
$$

Training this model should take on the order of $$O(n^3)$$ time, or the time it would take to calculate the inverse of the matrix $$T$$.

---

One way of looking at this is that we're *convolving* the array $$t$$ with the parameters of the linear model $$x$$. That means that instead of performing the convolution (which would take up about $$O(n^2)$$ time), we can instead take the Fourier representations of $$x$$ and $$t$$, multiply them elementwise in the Fourier domain, and convert them back to their time domain representations to perform the same calculation in just $$O(n\log{n})$$ time.

There are also ways to significantly speed up training, but we'll need a few prerequisites before we can examine those.

---

First, we define the [Toeplitz matrix](http://ee.stanford.edu/~gray/toeplitz.pdf) as a matrix of the form:

$$
\begin{bmatrix}
t_0 & t_1 & t_2 \\
t_{-1} & t_0 & t_1 \\
t_{-2} & t_{-1} & t_0
\end{bmatrix}
$$

Notice that the matrix $$T$$ is already in this form. If we had different kernel sizes, we can just pad the edges of both $$T$$ and $$x$$ to get a Toeplitz matrix.

$$
\begin{bmatrix}
t_3 & t_4 & t_5 & 0 \\
t_2 & t_3 & t_4 & t_5 \\
t_1 & t_2 & t_3 & t_4 \\
0 & t_1 & t_2 & t_3 \\
\end{bmatrix}
\begin{bmatrix}0 \\ x_1 \\ x_2 \\ 0\end{bmatrix}
= \begin{bmatrix}y_1 \\ y_2 \\ y_3 \\ y_4\end{bmatrix}
$$

While I won't go into detail about it here, you can apply the [Levinson-Durbin recursion](https://www.mathworks.com/help/signal/ref/levinson.html) to compute the parameter values in just $$O(n^2)$$ time. But we can do better.

---

A circulant matrix is a Toeplitz matrix where $$t_k = t_{k-n}$$, or to illustrate:

$$
\begin{bmatrix}
t_0 & t_1 & t_2 \\
t_2 & t_0 & t_1 \\
t_1 & t_2 & t_0
\end{bmatrix}
$$

Circulant matrices can be produced given just the original time signal by shifting every element in a row to the right and moving the right most element to the left. Another way of reproducing them is by examining its spectral decomposition on the right-hand side of this equation: $$T = F \Lambda F^H$$. Here $$F$$ denotes the Fourier transform matrix, $$F^H$$ denotes its complex-conjugate, and $$\Lambda$$ is a diagonal matrix, where each value is given
by the Fourier representation of $$t$$.

That's a lot to take in, so here's a demonstration that it actually works in Python.

```python
import numpy as np

def main():
    t = np.array(range(0, 10))

    Lambda = np.diag(np.fft.fft(t))
    F = np.fft.fft(np.eye(t.size))
    F_H = F.conj()
    T = np.real(F.dot(Lambda).dot(F_H)) / t.size
    print(t)
    print(np.round(T).astype(int))
```

which should yield:

```python
[0 1 2 3 4 5 6 7 8 9]
[[0 1 2 3 4 5 6 7 8 9]
 [9 0 1 2 3 4 5 6 7 8]
 [8 9 0 1 2 3 4 5 6 7]
 [7 8 9 0 1 2 3 4 5 6]
 [6 7 8 9 0 1 2 3 4 5]
 [5 6 7 8 9 0 1 2 3 4]
 [4 5 6 7 8 9 0 1 2 3]
 [3 4 5 6 7 8 9 0 1 2]
 [2 3 4 5 6 7 8 9 0 1]
 [1 2 3 4 5 6 7 8 9 0]]
```

---

This property is convenient because it allows us to greatly simplify certain results by calculating them in the Fourier domain. For [example](http://www.robots.ox.ac.uk/~joao/publications/henriques_tpami2015.pdf), the calculation of the appropriate weights of a ridge regression model (that is, values of $$x$$ in this page's notation), can be reduced to a series of elementwise operations in the Fourier domain, such that:

$$\hat{x} = diag(\frac{\hat{t}^*\odot\hat{y}}{\hat{t}^*\odot\hat{t}+\lambda})$$

where $$\odot$$ represents elementwise multiplication, $$\hat{t}$$ is the DFT of $$t$$, $$\hat{t}^*$$ is the complex conjugate of the DFT of $$t$$, and $$\lambda$$ is the regularization term of the ridge regression model. This reduces the formula to a set of elementwise multiplications, from which $$x$$ can be retrieved using the inverse DFT in $$O(n\log{n})$$ time.

---

**Edited November 30, 2017**

For the 2-dimensional case, we can obtain all possible shifts on a matrix using the 2-D DFT as follows:

```python
import numpy as np

def main():
    a = np.array(range(1,10)).reshape((3,3))
    A_hat = np.diag(np.fft.fft2(a).flatten())
    dft_y = np.fft.fft(np.eye(a.shape[0]))
    dft_x = np.fft.fft(np.eye(a.shape[1]))
    dft = np.kron(dft_y, dft_x)
    A = np.real(dft.dot(A_hat).dot(dft.conj())) / a.size
    print(A)
    print(a)
```
 This should yield:

```python
[[ 1.  2.  3.  4.  5.  6.  7.  8.  9.]
 [ 3.  1.  2.  6.  4.  5.  9.  7.  8.]
 [ 2.  3.  1.  5.  6.  4.  8.  9.  7.]
 [ 7.  8.  9.  1.  2.  3.  4.  5.  6.]
 [ 9.  7.  8.  3.  1.  2.  6.  4.  5.]
 [ 8.  9.  7.  2.  3.  1.  5.  6.  4.]
 [ 4.  5.  6.  7.  8.  9.  1.  2.  3.]
 [ 6.  4.  5.  9.  7.  8.  3.  1.  2.]
 [ 5.  6.  4.  8.  9.  7.  2.  3.  1.]]

[[1 2 3]
 [4 5 6]
 [7 8 9]]
```

In this case, each row of the result, when reshaped, represents every possible shift in the image. Interestingly, the resultant matrix can be subdivided into circulant blocks, such as the region:

```python
[[ 1.  2.  3.]
 [ 3.  1.  2.]
 [ 2.  3.  1.]]
```

The resulting matrix is known as a block circulant circulant block matrix. It can be shown that most properties which apply to 1-dimensional circulant matrices can be generalized for 2-dimensional ones.

