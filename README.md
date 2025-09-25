# PetersenLAB

# Coupling Particle Simulations with Continuum Electrostatics
**Computing Electric Fields from Particle-Derived Charge Densities**  
*Author: Pouya Golchin*

> This project couples GPU-accelerated Brownian particle simulations with continuum electrostatics. We compute the volumetric charge density directly from particle positions and solve Poisson’s equation for the electrostatic potential and field. This enables (i) validation against continuum electrokinetic models and (ii) quantification of electric-double-layer screening at charged walls.

---

## Table of Contents
- [Background](#background)
- [Governing Equations](#governing-equations)
- [Charge Density from Particle Simulations](#charge-density-from-particle-simulations)
- [Field Solution Strategy](#field-solution-strategy)
- [Demo Video / Preview](#demo-video--preview)
- [Getting Started](#getting-started)
- [Cite / Acknowledge](#cite--acknowledge)
- [License](#license)

---

## Background
We bridge continuum electrochemical equations with particle-based simulations in nanoscale pores. Instead of always prescribing the electric field from a continuum solver, we periodically **measure** the charge density \( \rho_c(\mathbf{x}) \) from particle positions and solve the field self-consistently. This provides a complementary view on screening and equilibrium structure and leverages GPU parallelism to generate many realizations of ion positions.

---

## Governing Equations
The electrostatic potential \( \Phi(\mathbf{x}) \) satisfies Poisson’s equation:
$$
-\varepsilon \nabla^2 \Phi(\mathbf{x}) = \rho_c(\mathbf{x}).
\tag{1}
$$

The electric field and Gauss’s law:
$$
\mathbf{E}(\mathbf{x}) = -\nabla \Phi(\mathbf{x}),
\qquad
\nabla \cdot \mathbf{E}(\mathbf{x}) = \rho_c(\mathbf{x}).
\tag{2}
$$

Charge density from local ion concentrations:
$$
\rho_c(\mathbf{x}) = e_0 \big( c_{+}(\mathbf{x}) - c_{-}(\mathbf{x}) \big),
\tag{3}
$$
where \( e_0 \) is the elementary charge, \( c_{+} \) is the cation concentration, and \( c_{-} \) is the anion concentration. Charge neutrality implies \( \rho_c = 0 \) when \( c_{+} = c_{-} \).

---

## Charge Density from Particle Simulations
Discretize the domain into voxels \( \{\mathcal{V}_j\} \) with volume \( \Delta V \). From instantaneous particle positions, form counts \( N^{+}_j \) and \( N^{-}_j \) (cations/ions) per voxel:
$$
c_{+}(\mathbf{x}_j) \approx \frac{N^{+}_j}{\Delta V},
\qquad
c_{-}(\mathbf{x}_j) \approx \frac{N^{-}_j}{\Delta V}.
\tag{4}
$$
Substitute into (3) to obtain a piecewise-constant estimator of \( \rho_c \).

**Statistical rescaling.** If the real system contains \( N_{\text{real}} \) ions but we simulate \( N_{\text{sim}} \gg N_{\text{real}} \) independent Brownian trajectories to sample spatial statistics, rescale:
$$
\rho_c^{\text{real}}(\mathbf{x}) \;=\; \frac{N_{\text{real}}}{N_{\text{sim}}}\,\rho_c^{\text{sim}}(\mathbf{x}).
\tag{5}
$$

---

## Field Solution Strategy
At selected coupling times \( t_k \):
1. **Aggregate positions:** collect particle coordinates \( \{\mathbf{x}_i(t_k)\} \).
2. **Form \( \rho_c(\mathbf{x}) \):** bin particles to voxels and compute (3) via (4).
3. **Solve Poisson:** compute \( \Phi \) and \( \mathbf{E} = -\nabla \Phi \); store \( (\Phi,\mathbf{E}) \).
4. **Update field:** inject the new field into the particle simulator.
5. **Advance particles:** integrate the next block of Brownian steps with electrostatic forces.

Coupling does **not** occur every micro-step. Choose a stride \( n_{\text{BD}} \in \mathbb{N} \) (advance particles \( n_{\text{BD}} \) steps between field updates) to balance accuracy and performance.
