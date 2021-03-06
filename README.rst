=====================
LIsp Flavoured Tensor
=====================

Requirements
============

* Python 2.7
* `ply`__
* `islpy`__

.. __: https://pypi.python.org/pypi/ply
.. __: https://pypi.python.org/pypi/islpy


Quickstart
==========

A very simple matrix dot product example.

matmul.model

.. code::

    A :: (in 8 8)
    B :: (out 8 8)

    B := (((reduce 1 +)"2) (A (*"1 2) A))

matmul.c

.. code:: c

    #include <stdio.h>
    #ifdef USE_OPENCL
    #include "matmul_cl.h"
    #else
    #include "matmul_c.h"
    #endif

    int
    main(){
      float A[8][8];
      float B[8][8];
      struct matmul_state state = {.A = A, .B = B};

      for(int i=0; i<8; i++)
        for(int j=0; j<8; j++)
          A[i][j] = i*8+j;

      matmul(&state);

      for(int i=0; i<8; i++) {
        for(int j=0; j<8; j++)
          printf("%5.0f ", B[i][j]);
        printf("\n");
      }

      return 0;
    }

compile and run

.. code:: bash

    $ cat matmul.model
    $ python -m lift --emit opencl --sizes '{ kernel[i] -> grid[2,2]; kernel[i] -> block[2,2]; kernel[i] -> tile[2,2,2]}' matmul matmul.model
    $ gcc -DUSE_OPENCL -o matmul.elf -Wl,--format=binary matmul.cl -Wl,--format=default matmul.c matmul_cl.c -lOpenCL
    $ ./matmul.elf
     1120  1148  1176  1204  1232  1260  1288  1316
     2912  3004  3096  3188  3280  3372  3464  3556
     4704  4860  5016  5172  5328  5484  5640  5796
     6496  6716  6936  7156  7376  7596  7816  8036
     8288  8572  8856  9140  9424  9708  9992 10276
    10080 10428 10776 11124 11472 11820 12168 12516
    11872 12284 12696 13108 13520 13932 14344 14756
    13664 14140 14616 15092 15568 16044 16520 16996


Shell
=====

Besides compilation, we can also run them in the Python shell, but
much slower and consume lots more memory than you might expected.

.. code:: pycon

    >>> SOURCE = """
    ... A :: (in 8 8)
    ... B :: (out 8 8)
    ... B := (((reduce 1 +)"2) (A (*"1 2) A))
    ... """
    >>> A = Array((8,8),range(64))
    >>> exec load_source(SOURCE)
    >>> B
    Array((8, 8), [1120.0, 1148.0, 1176.0, 1204.0, 1232.0, 1260.0, 1288.0, 1316.0, 2912.0, 3004.0, 3096.0, 3188.0, 3280.0, 3372.0, 3464.0, 3556.0, 4704.0, 4860.0, 5016.0, 5172.0, 5328.0, 5484.0, 5640.0, 5796.0, 6496.0, 6716.0, 6936.0, 7156.0, 7376.0, 7596.0, 7816.0, 8036.0, 8288.0, 8572.0, 8856.0, 9140.0, 9424.0, 9708.0, 9992.0, 10276.0, 10080.0, 10428.0, 10776.0, 11124.0, 11472.0, 11820.0, 12168.0, 12516.0, 11872.0, 12284.0, 12696.0, 13108.0, 13520.0, 13932.0, 14344.0, 14756.0, 13664.0, 14140.0, 14616.0, 15092.0, 15568.0, 16044.0, 16520.0, 16996.0])


Ranks
=====

We use the same ranks as `J`__.

.. __: http://www.jsoftware.com/help/learning/07.htm

.. code:: pycon

    >>> from lift import *
    >>>
    >>> SOURCE = """
    ... A :: (in 3)
    ... B :: (out)
    ... B := ((reduce 1 +) A)
    ... """
    >>> A = Array((3,), [1,2,3])
    >>> exec load_source(SOURCE)
    >>> B
    Array((), [6.0])
    >>>
    >>> SOURCE = """
    ... A :: (in 2 3)
    ... B :: (out 3)
    ... B := ((reduce 1 +) A)
    ... """
    >>> A = Array((3,2), [1,2,3,4,5,6])
    >>> exec load_source(SOURCE)
    >>> B
    Array((3,), [5.0, 7.0, 9.0])
    >>>
    >>> SOURCE = """
    ... A :: (in 2 3)
    ... B :: (out 2)
    ... B := (((reduce 1 +)"1) A)
    ... """
    >>> A = Array((3,2), [1,2,3,4,5,6])
    >>> exec load_source(SOURCE)
    >>> B
    Array((2,), [6.0, 15.0])
    >>>
    >>> SOURCE = """
    ... A :: (in 2)
    ... B :: (in 2)
    ... C :: (out 2)
    ... C := (A + B)
    ... """
    >>> A = Array((2,),[1,2])
    >>> B = Array((2,),[3,4])
    >>> exec load_source(SOURCE)
    >>> C
    Array((2,), [4, 6])
    >>>
    >>> SOURCE = """
    ... A :: (in 2)
    ... B :: (in 2)
    ... C :: (out 2 2)
    ... C := (A (+"0 1) B)
    ... """
    >>> A = Array((2,),[1,3])
    >>> B = Array((2,),[3,4])
    >>> exec load_source(SOURCE)
    >>> C
    Array((2, 2), [4, 5, 6, 7])


Gradient
========

We have automatic differentiation in LiFT. The example here is taken
from `A Step by Step Backpropagation Example`__ .

.. __: https://mattmazur.com/2015/03/17/a-step-by-step-backpropagation-example/


.. code:: pycon

    >>> from lift import *
    >>> SOURCE = """
    ... Input :: (in 2)
    ... W1 :: (in 2 2)
    ... B1 :: (in)
    ... W2 :: (in 2 2)
    ... B2 :: (in)
    ... Output :: (out 2)
    ...
    ... sigmoid"0 := (1 / ((exp (0 - y)) + 1))
    ... dot"1 1 := ((reduce 1 +) (x * y))
    ...
    ... Hidden := (sigmoid ((W1 dot Input) + B1))
    ... Output := (sigmoid ((W2 dot Hidden) + B2))
    ...
    ... Target :: (in 2)
    ... Loss :: (out)
    ...
    ... Loss := ((reduce 1 +) (0.5 * ((Target - Output) ** 2)))
    ...
    ... dW1 :: (grad Loss W1)
    ... dW2 :: (grad Loss W2)
    ...
    ... nW1 :: (out 2 2)
    ... nW2 :: (out 2 2)
    ...
    ... nW1 := (W1 - (0.5 * dW1))
    ... nW2 := (W2 - (0.5 * dW2))
    ... """
    >>> W1 = Array((2,2), [0.15,0.20,0.25,0.30])
    >>> B1 = Array((), [0.35])
    >>> W2 = Array((2,2), [0.40,0.45,0.50,0.55])
    >>> B2 = Array((), [0.60])
    >>> Input = Array((2,), [0.05,0.10])
    >>> Target = Array((2,), [0.01,0.99])
    >>> exec load_source(SOURCE)
    >>> Output
    Array((2,), [0.7513650695523157, 0.7729284653214625])
    >>> Loss
    Array((), [0.2983711087600027])
    >>> nW1
    Array((2, 2), [0.1497807161327628, 0.19956143226552567, 0.24975114363236958, 0.29950228726473915])
    >>> nW2
    Array((2, 2), [0.35891647971788465, 0.4086661860762334, 0.5113012702387375, 0.5613701211079891])
