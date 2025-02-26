# Calling Fortran Using Ctypes

## Background


Python's [ctypes](https://docs.python.org/3/library/ctypes.html) interface provides an avenue to directly from Python calling libraries that provide functions that can be called from C. This provides an avenue to call Fortran code that is callable from, C without the need to modify the Fortran code.  An excellen summary of the various approaches to make calls to a shared library written in Fortran from other language is given [here](https://foreign-fortran.readthedocs.io/en/latest/index.html).

Cmake is a software build system that is the defacto standard for C++ projects and the [scikit-build](https://scikit-build.readthedocs.io/en/latest/) package provides a bridge between the setuptools Python module and Cmake. 

Thus the aim of this tutorial is to illustrate how a build system can be created for a python package that provides a wrapper to fortran code and how pyhton can interface to the fortran code via ctypes, without relying on `f2py` which comes with its own challenges when the aim is to provide a python package that includes Fortran code. 


Specific this tutorial illustrates the steps taken to create [https://github.com/inlab-geo/pyrf96](https://github.com/inlab-geo/pyrf96).  the strategy is to first organise the source code, create the build system and then write the wrapper on the python side.


### Prerequisits

This tutorial assumes that `python`, `cmake`, `gcc` and `gfortran` are installed and configured and that `virtualenvwrapper` has been installed to facilitate the creation of a virtualenv.


### Initial considerations

Given the aim is to create a python package names have to be chosen for the python package and the fortran source code. We chose `rf96` as the basename and in the following the python package will be called `pyrf96` and the fortran library will have the basename `rf96`.

The first step is to begin the directory structure that is create a directory `pyrf96` with a subdirectory `rf96`

```
mkdir pyrf96
cd pyrf96
mkdir rf96
cd rf96
```

## Creating the build system

### Organising the fortran source code

This assumes that the fortran code base is in a form that provides a function that we wish to call from Python to solve forward problem. All fortran source files now need to be placed in `pyrf96/rf96`. It is not possible to have subdirectories inside `pyrf96/rf96`.

It is recommened to check if object files can be built for all the fortran source files using. 

```
cd pyrf96/rf96
gfortran -c *.f*
rm *.o
```

This may produce warnings for legacy fortran code which may be ignored. Compile time errors however would need be resolved before proceeding. 

### Creating a Cmake file for the fortran code

To provide the compile instruction to Cmake for the fortran code a file has to be place in `pyrf96/rf96` with the filename `CmakeLists.txt`

```
enable_language(Fortran)
file( GLOB rf96_fortran_functions *.f* )

find_package(Python REQUIRED COMPONENTS Interpreter Development.Module)
Python_add_library(rf96 MODULE WITH_SOABI ${rf96_fortran_functions})

install(TARGETS rf96 DESTINATION ./pyrf96)
```

### Creating a placeholder for the python wrapper code

The python source code is kept separate from any fortran code. Thus, we create in the top level directory a directory `src` with a subdirectory `pyrf96`. 

```
mkdir src
cd src
mkdir pyrf96
```

We now need to decide what the python function provided by the package `pyrf96` will be called here we chose `rfcalc` i.e. `pyrf96.rfcalc`

In the newly created directory `src/pyrf96` we now create a file called `__init__.py` with the following content

```
cat __init__.py
from ._pyrf96 import rfcalc  # noqa
```

We now create a second file which will later contain the python code calling the fortran function. At this stage we again just create a placeholder file as the aim is to first setup the build system.

``` 
cat _pyrf96.py

from typing import Literal
import os
import glob
import platform
import numpy
import numpy.ctypeslib
import ctypes

def rfcalc():
	pass 

__all__ = ["rfcalc"]
```

### Creating a unit test placeholder

It is good practice to have a test of some sort to verify a forward kernel does behave as intended. We thus create a at the top level at same level as src a directory `test` and place a script called `test_pyrf96.py` inside the newly created directory.

```
cat test_pyrf96.py

import numpy
import pyrf96
```

### Completing the build system

To complete the build system we need to create in the top level directory a `README.md`, a `pyproject.toml`, a `requirements.txt` and a `CmakeLists.txt` file.

The `README.md` ideally to contains information about the providence of any source code taken from other packages and installation instructions as well as usage examples.

```
cat README.md

# PyRf96
Lorem ipsum dolor sit amet, consectetur adipiscing elit...

## Installation

## Licensing

### References
```

The `pyproject.toml` file contains information about the python package we are creating and its dependencies

```
cat pyproject.toml

[build-system]
requires = ["scikit-build-core[rich,pyproject]"]
build-backend = "scikit_build_core.build"

[project]
name = "pyrf96"
version = "0.1.0"
requires-python = ">=3.9"
```

Finalle we create a `CmakeLists.txt` file at the top level directory which will be calling the Cmake file in the directory with the fortran source code.

```
cat CmakeLists.txt

Cmake_minimum_required(VERSION 3.15...3.26)
set(Cmake_VERBOSE_MAKEFILE on)
project(pyrf96)
add_subdirectory(rf96)
```

### Testing the build system

It is recommended to test the build system by installing the package into a virtual environment created for this purpose. Here we use virtualenvwrapper.

```
mkvirtualenv test
```

In the top level directory we now use pip and try to install the package
 
 ```
 pip install .
 ```
 
 This will result in the package being installed but with the python wrapper not complete we can not yet call the fortran code.
 
## Creating the Python wrapper

At this stage all the build system does is compile the fortran code into a shared object.


### Identifying the subroutines we seek to call

In the fortran source code we now need to identify the functions we want to make available to python and set a unique name for this function to bypass the name wrangling which is compiler and operating system dependent. The functions we seek to access are located in `rf96/RF90.f90` specifically we desire to call the following subroutines.

```
Subroutine RFcalc_nonoise(voro,mtype,fs,...
Subroutine RFcalc_noise(voro,mtype,...
Subroutine voro2mod(voro,npt,h,...
```

We give the subroutines unique names using Iso-C-bindings to specify the subroutine name using `bind(c,name="CHOSEN NAME")`  this is the anem of the function seen by Python.

```
Subroutine RFcalc_nonoise(voro,...,wdata)bind(c,name="rfcalc_nonoise")
Subroutine RFcalc_noise(voro,...,wdata)bind(c,name="rfcalc_noise")
Subroutine voro2mod(voro,...,qb)bind(c,name="voro2mod")
```


### Create the python code to call the function

We now define the python functions that will be calling the fortran functions in `pyrf96/src/pyrf96/_pyrf96.py`, we use the ctypes interface to load the library.

```
librf96=ctypes.cdll.LoadLibrary(glob.glob(os.path.dirname(__file__)+"/rf96*.so")[0])

```

The fortran function we have identified and set the name for using the bind statemen can now be called from pyhton. The problem however is that the python variables need to be converted into objects that can be passed into the fortran function. With fortran subroutines generally being pass by reference this means we have to create pointers and pass these to the subroutine. Specfically we have to create pointers to python variables that have been converted to ctypes. 

The file `pyrf96/src/pyrf96/_pyrf96.py` contains examples for the commonly seen function arguments by fortran subroutines. 

Integer scalar - this assumes mtype is type int

```
mtype_f = ctypes.c_int(mtype)
mtype_c = ctypes.byref(mtype_f) 
```

Real/Float scalar - this assumes mtype is type float

```
angle_f = ctypes.c_float(angle)
angle_c = ctypes.byref( angle_f) 
```

Array of Real/Float - this assumes model is numpy.ndarray

```
model_f = numpy.asfortranarray(model,dtype=numpy.float32)
model_c = model_f.ctypes.data_as(ctypes.POINTER(ctypes.c_float))
```

All function arugments have to be allocated on the python side. The function `RFcalc_noise` for example returns an array `wdata` that contains the result. This variable has to be allocated and then is passed by reference

```
wdata_f = numpy.asfortranarray(numpy.zeros(ndatar),dtype=numpy.float32)
wdata_c = wdata_f.ctypes.data_as(ctypes.POINTER(ctypes.c_float))
```

As `wdata_c` is only a pointer to `wdata_f` the results of the subroutine are on the python side available in the varaible `wdata_f`


### Making the python functions available

Python functions in `pyrf96/src/pyrf96/_pyrf96.py` need to be listed in  `pyrf/src/pyrf96/__init__.py` to be available when the package is being imported.

## Installing the Python package

The python pacakge that has been created can be installed using pip

```
 pip install .
```

Or if it is hosted on github it can be installed directly from github.

```
pip install git+https://<repo url>
```

That is in the case of pyrf96

```
pip install git+https://github.com/inlab-geo/pyrf96
```

## Implementing Test
