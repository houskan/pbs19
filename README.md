# Melting Fluid Simulation 
2019 - Roger Barton, Alessia Paccagnella and Niklaus Houska
<div align="center">
<img src="assets/README/1_Bunny.gif" alt="Bunny1" width="49%"/>
  <img src="assets/README/2_Bunny.gif" alt="Bunny2" width=49%"/>
</div>

## Table of Contents
* [About the Project](#about-the-project)
* [What Has Been Implemented](#what-has-been-implemented)
* [References](#references)

## About the Project
This repository contains the source code for the graded semester project for the [Physically-Based Simulation course 2019](https://cgl.ethz.ch/teaching/simulation20/fame.php) at ETH Zurich. 

We provide an implementation of a 2D PIC/FLIP fluid simulation with implicit viscosity solving. Additionally, we implemented heat diffusion and set viscosity based on temperature by clamped inverse square to achieve a nice melting effect. 

<div align="center">
<img src="assets/README/visco_1.gif" alt="Viscosity 1" width="49%"/>
  <img src="assets/README/visco_1k.gif" alt="Viscosity 1000" width=49%"/>
  <img src="assets/README/visco_10k.gif" alt="Viscosity 10k" width="49%"/>
  <img src="assets/README/visco_100m.gif" alt="Viscosity 100m" width=49%"/>
</div>
Real-time fluid simulation with varying viscosity: 1 (Top left), 1000 (Top Right), 10'000 (Bottom Left), 100'000'000 (Bottom Right)



## What has been implemented

- 2D PIC/FLIP fluid simulation
- Implicit **viscosity** solving to allow for high viscosities in real-time based on [2] and [5]
- Adaptive timestep (CLF Condition)
- **Pressure** solving via Gauss-Seidel
  - Ghost pressure at air/fluid boundary
  - Fixed iteration count (more efficient than residual computation)
- Enforce normal dirichlet boundary conditions
- Smart particle **reseeding** based on [6]
    - Add a particle to cell if less than 3  
    - Remove particles if more than 12 in cell
    - Only add particles to non-boundary cells and cells with low velocity (0.5 * dx)
- Implicit **temperature** solver based on [1]
  - Create a *symmetric positive definite* matrix and solve it using the *conjugate gradient* method
  - This matrix and its solver can be precomputed once, see `FluidDomain::buildTemperatureMatrix`
  - Link fluid and temperature simulations by making particles transport some of the temperature, see `FluidSolver::transferTemperatureGridToParticles`
    - Efficiently tuned by `GuiData::m_particleTemperatureTransfer`
    - Works best when slightly below the average particles per grid cell (1/8 by default)
- Viscosity based on temperature by clamped inverse square, (see `FluidSolver::updateViscosity`)
- **Interaction**, click to add particles or temperature. Use an image as a brush (based on ascii art export from gimp)

Probably the most interesting function is `FluidSolver::stepPICFLIP` which gives an overview of a single step.

## Optimization
*Course notes*

Note: scaling the grid size by 2x the area means **more than** 2x slowdown -> performance does not scale linearly.

- Used *Eigen* to store grid/position data 
  - This gave a small speedup relative to ArrayT, possibly because of vectorization/alignment done by Eigen
- Main performance bottleneck are the Eigen solvers as well as the Gauss-Seidel pressure solver in `FluidSolver::solvePoissonCorrectVelocity`
- If the solver becomes instable, often with larger grid sizes, the *iterations for the pressure solver* need to be increased. Can be set in the UI.
- Eigen has been setup to run on all cores, all our solvers are `Eigen::ConjugateGradient` with the correct parameters to utilize this parallelism according to https://eigen.tuxfamily.org/dox/TopicMultiThreading.html
  - A potential to improve performance even more would be to use an *adaptive domain* based on min/max particle positions with some padding. However, in most of our scenarios most of the grid is used and this alternate method would incur more overhead.
- We also implemented a way of changing the floating point precision (`doubleT`), however, this has little effect on the frame rate
- Particle advection is done with openmp to make this completely negligible

Below is a profile in CLion for a ~15s high viscosity simulation with a 128x64 grid (using CMake `Release` mode and `g++` 7.4.0  on a i9-9980HK)

![image-20191216220217022](assets/README/image-20191216220217022.png)

*note: libgomp.so is related to eigen parallelisation through opemp*

## References

[1] https://www.cc.gatech.edu/~turk/my_papers/melt.pdf  
[2] https://cs.uwaterloo.ca/~c2batty/papers/BattyBridson08.pdf  
[3] https://www.cs.ubc.ca/~rbridson/fluidsimulation/fluids_notes.pdf  
[4] https://github.com/kbladin/Fluid_Simulation  
[5] https://github.com/rlguy/FLIPViscosity3D  
[6] https://pdfs.semanticscholar.org/a1bb/ba8ad75b4ffdaebfe56ce1aec35414247d14.pdf
