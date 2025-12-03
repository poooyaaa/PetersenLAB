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
