.. _install_instructions:


Software Overview
===================

.. image:: https://raw.githubusercontent.com/JeremyBYU/polylidar/major_changes_refactor/assets/polylidar_3D_architecture.jpg

Polylidar3D is a non-convex polygon extraction algorithm which takes as input either unorganized 2D point sets, unorganized 3D point clouds (e.g., airborne LiDAR point clouds), organized point clouds (e.g., range images), or user provided meshes. 
The non-convex polygons extracted represent flat surfaces in an environment, while interior holes represent obstacles on said surfaces. The picture above provides an overview of Polylidar3D's data input, frontend, backend, and output. 
The frontend transforms input data into a half-edge triangular mesh. This representation provides a common level of abstraction such that the the back-end core algorithms may efficiently operate on. 
The back-end is composed of four core algorithms: mesh smoothing, dominant plane normal estimation, planar segment extraction, and finally polygon extraction. 

Currently this repo (named Polylidar3D) has all the front-end modules and the plane/polygon extraction of the back-end core algorithms. 
The GPU accelerated mesh smoothing procedures for organized points clouds are found in a separate repo titled `OrganizedPointFilters <https://github.com/JeremyBYU/OrganizedPointFilters>`_. 
This must be installed if you desire fast mesh smoothing for organized point clouds (i.e., denoising). 
The dominant plane normal estimation procedure is general and implemented in a separate repo titled `Fast Gaussian Accumulator (FastGA) <https://github.com/JeremyBYU/FastGaussianAccumulator>`_. 
This must be installed if you don't know the dominant plane normals in your data input (very likely for organized point clouds and meshes). 
These modules themselves are written in C++ as well with Python bindings; see the respective repos for installation instructions. 
One day I will try to ease installation burden and automatically pull these dependencies into the build process.


Install Instructions
====================


If you just want to install the python bindings and have a supported architecture, use pip: `pip install polylidar`. Binary wheels have been created for Windows, Linux, and MacOS.
If this fails see below to install from source.

Build Project Library
------------------------------------

This instructions are for people who want to modify Poylidar3D's code and are focused on the C++ library. Building and subsequent installation is entirely through CMake now. You must have CMake 3.14 or higher installed and a C++ compiler with C++ 14 or higher. No built binaries are included currently.
There are several config options which can be specified here during step (2):

.. code:: text

    PL_BUILD_BENCHMARKS:BOOL=OFF // PL - Build Benchmarks
    PL_BUILD_EXAMPLES:BOOL=ON // PL - Build Examples
    PL_BUILD_PYMODULE:BOOL=ON // PL -Build Python Module
    PL_BUILD_TESTS:BOOL=ON // PL - Build Tests
    PL_BUILD_WERROR:BOOL=OFF // PL - Add Werror flag to build (turns warnings into errors)
    PL_USE_ROBUST_PREDICATES:BOOL=OFF // PL - Use Robust Geometric Predicates
    PL_BUILD_FASTGA=OFF // PL - Build FastGA with Example"
    PL_WITH_OPENMP:BOOL=ON // PL - Build with OpenMP Support


1. ``mkdir cmake-build && cd cmake-build`` - create build folder directory
2. ``cmake .. -DCMAKE_BUILD_TYPE=Release -DFETCHCONTENT_QUIET=OFF -DPYTHON_EXECUTABLE=$(python3 -c "import sys; print(sys.executable)")`` - For windows also add ``-DCMAKE_GENERATOR_PLATFORM=x64``. This will take 10-20 minutes while dependencies are being downloaded from Github. Sorry, take a break! 
3. ``cmake --build . --config Release -j4`` - Build Polylidar3D, change ``-j4`` to how many processors/threads you have. 
4. ``cd .. && ./cmake-build/polylidar-simple`` - Simple test program.

.. note::
    There have been some recent issues with CMake not identifying the activated python virtual environment, see `Issue <https://github.com/JeremyBYU/polylidar/issues/5>`_ . To fix this add ``-DPYTHON_EXECUTABLE=$(python3 -c "import sys; print(sys.executable)")`` to force pybind11 to use your virtualenv python environment.
    I have also noticed that things just "work again" if you use CMake 3.15 or higher as well as upgrade ``pybind11`` to version 2.6.0. However setting the python executable is simpler and less intrusive.

Build and Install Python Extension
------------------------------------

The basic setup here is that CMake will build the python extension (.so or .dll) into a standalone folder. Then you use `pip` to install from that folder.

1. Install `conda <https://conda.io/projects/conda/en/latest/>`_ or create a python virtual environment (`Why? <https://medium.freecodecamp.org/why-you-need-python-environments-and-how-to-manage-them-with-conda-85f155f4353c>`_). I recommend ``conda`` for Windows users. Activate your environment.
2. ``conda install shapely`` - Only for Windows users because ``conda`` handles windows binary dependency correctly.
3. ``cd cmake-build`` - Ensure you are in cmake build directory.
4. ``cmake --build . --target python-package --config Release -j4`` - Build python extension, will create folder ``cmake-build/lib/python_package``
5. ``cd lib/python_package`` - Change to standalone python package. 
6. ``pip install -e .`` - Install the python package in ``develop/edit`` mode into python virtual environment.
7. ``cd ../../../ && pip install -r dev-requirements.txt`` - Move back to main folder and install optional dependencies to run python examples.

.. warning::
    Polylidar3D uses Open3D to visualize 3D geometries (pointclouds, polygons, meshes, etc.). A recent version of Open3D has a serious performance regression which causes severe slowdown during visualization. 
    This regression is on versions 0.10 and 0.11 of Open3D. Regression details: `Link <https://github.com/intel-isl/Open3D/pull/2523>`_ , `Issue1 <https://github.com/intel-isl/Open3D/issues/2472>`_ , `Issue2 <https://github.com/intel-isl/Open3D/issues/2157>`_.
    I recommend that you stick with 0.9.0, build from master, or wait for 0.12.

Download Example/Fixture Data
------------------------------

You can download example data `here <https://drive.google.com/file/d/1T5u7Cn8H_rWZpugcr_h3VrRx_onTpcX7/view?usp=sharinghttps://drive.google.com/file/d/1T5u7Cn8H_rWZpugcr_h3VrRx_onTpcX7/view?usp=sharing>`_ . Everything should be placed in the ```fixtures``` folder.

C++ Projects
-------------

CMake Integration
^^^^^^^^^^^^^^^^^^

To integrate with a *different* CMake Project do the following:

1. Add as a submodule into your repo: ``git submodule add https://github.com/JeremyBYU/polylidar thirdparty/polylidar``
2. Add the following to your CMakeLists.txt file:

.. code:: text

    add_subdirectory("thirdparty/polylidar")
    ... .
    target_link_libraries(MY_BINARY polylidar)


Robust Geometric Predicates
---------------------------

Delaunator (the 2D triangulation library used for 2D Point Sets and Unorganized 3D Point Clouds) does not use `robust geometric predicates <https://github.com/mikolalysenko/robust-arithmetic-notes>`_ for its orientation and incircle tests; `reference <https://github.com/mapbox/delaunator/issues/43>`_. 
This means that the triangulation can be incorrect when points are nearly colinear or cocircular. A library developed by Jonathan Richard Shewchuk provides very fast adaptive precision floating point arithmetic for `geometric predicates <https://www.cs.cmu.edu/~quake/robust.html>`_.  
This library is released in the public domain and an updated version of it is maintained at this `repository <https://github.com/danshapero/predicates>`_. I have included this source code in the folder ``polylidar/predicates`` .  

If you desire to have robust geometric predicates built into Delaunator you must build with the CMake option. For example ``cmake .. -DCMAKE_BUILD_TYPE=Release -DFETCHCONTENT_QUIET=OFF -DPL_USE_ROBUST_PREDICATES=ON``.