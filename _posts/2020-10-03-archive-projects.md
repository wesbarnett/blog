---
title: "Molecular Dynamics-related repositories archived"
description: "News on archiving several projects"
layout: post
toc: true
comments: true
hide: false
search_exclude: false
categories: [linux,gromacs,fortran,julia]
---

I recently archived several projects on Github that I no longer have the attention to
maintain. These all relate to [GROMACS](http://www.gromacs.org/) and
[LAMMPS](https://lammps.sandia.gov/), Molecular Dynamics programs, that I used in
graduate school and in my postdoctoral research.

Here's a list of the archived repositories:

* [dcdfort](https://github.com/wesbarnett/dcdfort) - a modern Fortran library used for
  analyzing the output of LAMMPS simulations.
* [libgmxfort](https://github.com/wesbarnett/libgmxfort) - a modern Fortran library used
  for analyzing xtc files, which are output from GROMACS simulations
* [libgmxcpp](https://github.com/wesbarnett/libgmxcpp) - similiar to libgmxfort but in
  C++
* [MolecularDynamics.jl](https://github.com/wesbarnett/MolecularDynamics.jl) - Julia
  library for analyzing the output of GROMACS simulations. I actually didn't end up
  using it much. Julia has changed a whole lot since I wrote this!

I had also written a few tutorials for getting started in GROMACS a few years ago.
Another research group that I am not affiliated with asked to host them after I could no
longer support them. [They are located
here](https://www.svedruziclab.com/tutorials/gromacs/).

Most of my graduate and research work used Fortran at the time given it's ability to
parallelize calculations easily with OpenMP. If I was still in the academic world I
would probably focus more on learning Julia and see how far that would get me since the
language is now stable.

I hope that the above libraries were useful for those who analyze Molecular Dynamics
output and need to use Fortran. I'm actually surprised so many ended up using the
Fortran libraries. My guess is it made it easier to interact with legacy coce.

I learned a lot about creating a system library, how to link Fortran to C libraries, as
well as cmake, ninja, make, and other build tools. In addition I learned a whole lot
about object-oriented programming in both Fortran and C++.

I'd like to encourage anyone to fork these libraries and continue development if they
seem useful.

Today my work primarily revolves around machine learning using Python. I don't see that
changing any time soon, although I'll probably continue to keep my eye Julia.
