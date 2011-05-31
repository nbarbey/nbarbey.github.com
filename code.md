---
layout: default
title: Code
---

Code
====

Most of the code I produce is hosted on github.

[linear_operators](http://nbarbey.github.com/linear_operators/)
---------------------------------------------------------------

This is a Python package originally inspired by the Scipy sparse
linear algebra package which implements a LinearOperator object.  This
operator mimics a matrix using function pointers for the matrix-vector
product. This results in a very easy and generic way to design
instrument models as it is very often required in signal processing of
spatial observatories.

The linear_operator package is mostly a developpement of this idea
along with bidings to optimization frameworks (scipy.optimize,
scikits.optimization) to perform inversion on the instrument models.


[TomograPy](http://nbarbey.github.com/TomograPy/)
-------------------------------------------------

This is a fast parallelized tomography projector / backprojector. It
originates from solar tomography application but could be use for
other applications. It uses the fast Siddon algorithm as its core. The
parallelization is done with OpenMP on the C part of the code.
