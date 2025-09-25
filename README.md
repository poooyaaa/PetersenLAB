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

---

## Background
We bridge continuum electrochemical equations with particle-based simulations in nanoscale pores. Instead of always prescribing the electric field from a continuum solver, we periodically **measure** the charge density \( \rho_c(\mathbf{x}) \) from particle positions and solve the field self-consistently. This provides a complementary view on screening and equilibrium structure and leverages GPU parallelism to generate many realizations of ion positions.

---

## Governing Equations
The electrostatic potential $\Phi(x)$ is governed by Poisson’s equation:

$$
-\varepsilon \nabla^2 \Phi(x) = \rho_c(x)
$$

Equivalently,

$$
\mathbf{E}(x) = -\nabla \Phi(x), \qquad \nabla \cdot \mathbf{E}(x) = \rho_c(x).
$$

Charge density from local ion concentrations:

$$
\rho_c(x) = e_0 \big(c_{+}(x) - c_{-}(x)\big),
$$

where $e_0$ is the elementary charge, $c_{+}$ is the cation concentration, and $c_{-}$ is the anion concentration.

---

## Charge Density from Particle Simulations
Discretize the domain into voxels $\mathcal{V}_j$ with volume $\Delta V$. From instantaneous particle positions,
form voxel counts $N^{+}_j$ and $N^{-}_j$:

$$
c_{+}(x_j) \approx \frac{N^{+}_j}{\Delta V}, \qquad
c_{-}(x_j) \approx \frac{N^{-}_j}{\Delta V}.
$$

Substituting gives a piecewise-constant estimator for $\rho_c$.

**Statistical rescaling.** If the real system contains $N_{\mathrm{real}}$ ions but we simulate $N_{\mathrm{sim}}\gg N_{\mathrm{real}}$ independent Brownian trajectories, then

$$
\rho_c^{\mathrm{real}}(x) = \frac{N_{\mathrm{real}}}{N_{\mathrm{sim}}}\,\rho_c^{\mathrm{sim}}(x).
$$


---

## Field Solution Strategy
At selected coupling times $t_k$:

1. **Aggregate positions:** collect particle coordinates $\{x_i(t_k)\}$.
2. **Form $\rho_c(x)$:** bin particles to voxels.
3. **Solve Poisson:** compute $\Phi$ and $\mathbf{E}=-\nabla\Phi$; store $(\Phi,\mathbf{E})$.
4. **Update field:** inject the new field into the particle simulator.
5. **Advance particles:** integrate the next block of Brownian steps with electrostatic forces.

Coupling does not occur every micro-step; choose a stride $n_{\mathrm{BD}}\in \mathbb{N}$ (particles advance $n_{\mathrm{BD}}$ steps between field updates).
