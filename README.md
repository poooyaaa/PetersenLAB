# Particle-based Electrokinetic RWPT (pyPAR² extension)

**Computing electric fields from particle-derived charge densities**

This repository extends a GPU-based Random Walk Particle Tracking (RWPT) code  
(**PAR²** by Rizzo et al.) from *neutral tracers* to *charged ions*.  

It also adds a **self-consistent electric field** computed from particle charge
density and wall charge, and provides a bridge to continuum electrokinetic
solvers.

---

## Table of Contents

1. [Background & Motivation](#background--motivation)
2. [Problem – why this matters, and what is missing](#problem--why-this-matters-and-what-is-missing)
3. [What this project does – unique approach](#what-this-project-does--unique-approach)
   - [From neutral tracers to charged ions](#from-neutral-tracers-to-charged-ions)
   - [Self-consistent electric field via Poisson’s equation](#self-consistent-electric-field-via-poissons-equation)
   - [Continuum vs particle-based E-field modes](#continuum-vs-particle-based-e-field-modes)
   - [GPU-based particle motion with PyTorch](#gpu-based-particle-motion-with-pytorch)
4. [Governing equations](#governing-equations)
5. [Charge density from particle simulations](#charge-density-from-particle-simulations)
6. [Field solution strategy](#field-solution-strategy)
7. [RWPT–continuum coupling: visualization](#rwptcontinuum-coupling-visualization)
8. [So what – why this is useful](#so-what--why-this-is-useful)

---

## Background & Motivation

Electrokinetic transport in charged nanochannels and porous media is hard to
simulate. The system couples

- fluid flow,  
- ion migration,  
- diffusion,  
- and electric fields  

in complex geometries.

Most existing tools fall into two groups:

- **Eulerian continuum solvers** (Poisson–Nernst–Planck, Poisson–Boltzmann, etc.)  
  These work on fixed grids and can suffer from numerical diffusion and high
  cost in 2D/3D.

- **Lagrangian RWPT solvers for neutral solutes.**  
  These are efficient and easy to parallelize, but usually do **not** include
  electrostatics or multi-species charged ions.

The **PAR²** code by Rizzo et al. is a GPU-accelerated RWPT implementation for
conservative tracers in porous media. PAR² tracks neutral particles in a given
velocity field and models hydrodynamic dispersion in a rigorous way.

However, in many electrokinetic problems, **feedback between ions and the
electric field is essential**:

- Ion distributions modify the electric field.  
- The electric field, in turn, modifies ion motion.  
- A neutral RWPT model cannot capture this loop.

This project bridges this gap by coupling a particle-based RWPT solver with
continuum electrostatics through Poisson’s equation.

---

## Problem – why this matters, and what is missing

Electrokinetic transport in charged nanochannels and porous media involves:

- charged walls,  
- overlapping electric double layers,  
- strong coupling between flow, diffusion, and electromigration.

Classical continuum approaches can be expensive and may smear sharp concentration
fronts due to numerical diffusion. Pure RWPT approaches ignore electrostatic
interactions altogether.

What is missing is a **GPU-ready, particle-based electrokinetic solver** that:

- tracks individual ions as particles,  
- computes a **self-consistent** electric field from their charge distribution
  and wall charge,  
- and can still interface cleanly with existing continuum solvers.

---

## What this project does – unique approach

This repository takes the PAR² RWPT idea and extends it in three main directions:

1. From neutral tracers to **charged ion species**.  
2. A **self-consistent electric field** computed by solving Poisson’s equation.  
3. A flexible coupling between **continuum** and **particle-based** E-field descriptions,  
   with GPU-accelerated RWPT implemented in PyTorch.

### (a) From neutral tracers to charged ions

Particles are no longer just passive solute markers.  
Each particle belongs to a **species** (cation, anion, neutral). Each species has:

- an integer charge number $z_m$,  
- a diffusion coefficient $D_m$,  
- an electrophoretic mobility

$$
\mu_e = \frac{D_m z_m q_e}{k_B T}.
$$

The code initializes a mixture of species with user-defined fractions.  
The RWPT update then includes both:

- **advection** by the fluid velocity,  
- **drift** in the electric field due to electrophoresis.

### (b) Self-consistent electric field via Poisson’s equation

The code can compute the electric field **self-consistently** from particle charges and
wall charge.

At selected time steps, it:

1. Deposits particle charges on a structured grid to build a volumetric charge
   density $\rho_c(x, y)$.
2. Adds a surface charge on the top and bottom walls, using a flexible profile:
   - uniform charge,
   - sinusoidal charge,
   - absolute sinusoidal charge.
3. Solves a Poisson problem

$$
-\varepsilon \nabla^2 \phi = \rho_c
$$

   with:
   - periodic boundary conditions in $x$,
   - Neumann boundary conditions at the walls based on the given surface charge.
4. Computes the electric field as

$$
\mathbf{E} = -\nabla \phi
$$

   on the same structured grid used for RWPT.

The Poisson solver uses high-order compact finite-difference schemes. The global
system is assembled as a sparse matrix and solved either:

- on the CPU with SciPy sparse linear algebra, or  
- on the GPU with CuPy-based iterative solvers (when available).

The updated field $\mathbf{E}(x, y)$ is then interpolated back to particle
positions and used in the next RWPT update.

### (c) Continuum vs particle-based E-field modes

The code supports two E-field modes:

- **Continuum mode** (`use_continuum_E = True`):
  - The electric field comes from an external continuum solver (e.g. Stokes /
    electrokinetic code).
  - The field is loaded from file and interpolated onto the RWPT grid.
  - No Poisson solve from particles is performed.
- **Self-consistent mode** (`use_continuum_E = False`):
  - The electric field is recomputed from particle charge density + wall charge
    by solving Poisson’s equation.

This makes it possible to:

- compare particle-based and continuum models on the same geometry,  
- start from a continuum field and later include feedback from particles,  
- test how different surface-charge patterns affect ion transport.

### (d) GPU-based particle motion with PyTorch

The RWPT part is implemented in **PyTorch**.

All particle positions and velocities are stored as tensors on the chosen device:

- CUDA GPU (if available),
- Apple MPS,
- or CPU as fallback.

The code can track up to $10^5$–$10^6$ particles over millions of time steps.  
Snapshots of the particle states are stored to disk as NumPy arrays.

Utilities are provided to:

- restart a simulation from the last snapshot,
- plot “unfolded” particle positions in a periodic channel,
- separate species and visualize them with different colors.

---
## Governing equations

The electrostatic potential $\Phi(\mathbf{x})$ is governed by **Poisson’s equation**:

$$
-\varepsilon \nabla^2 \Phi(\mathbf{x}) = \rho_c(\mathbf{x}),
$$

where

- $\varepsilon$ is the permittivity of the medium,
- $\Phi(\mathbf{x})$ is the electrostatic potential,
- $\rho_c(\mathbf{x})$ is the volumetric charge density.

Equivalently, the electric field is

$$
\mathbf{E}(\mathbf{x}) = -\nabla \Phi(\mathbf{x}), \qquad
\nabla \cdot \mathbf{E}(\mathbf{x}) = \rho_c(\mathbf{x}).
$$

To compute $\rho_c(\mathbf{x})$, we consider the local density of cations and anions in each voxel of the simulation domain. The volumetric charge density is

$$
\rho_c(\mathbf{x}) = e_0 \bigl(c^+(\mathbf{x}) - c^-(\mathbf{x})\bigr),
$$

where

- $e_0$ is the elementary charge,
- $c^+(\mathbf{x})$ is the cation concentration,
- $c^-(\mathbf{x})$ is the anion concentration.

---