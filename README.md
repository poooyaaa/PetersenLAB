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
The electrostatic potential \( \Phi(x) \) satisfies Poisson’s equation:

$$
-\varepsilon \nabla^2 \Phi(x) = \rho_c(x)
$$

The electric field and Gauss’s law:

$$
E(x) = -\nabla \Phi(x), \qquad \nabla \cdot E(x) = \rho_c(x)
$$

Charge density from local ion concentrations:

$$
\rho_c(x) = e_0 \, \big( c_{+}(x) - c_{-}(x) \big)
$$

where \( e_0 \) is the elementary charge, \( c_{+} \) is the cation concentration, and \( c_{-} \) is the anion concentration.

---

## Charge Density from Particle Simulations
Discretize the domain into voxels \( V_j \) with volume \( \Delta V \).  
From instantaneous particle positions, form counts \( N^{+}_j \) and \( N^{-}_j \):

$$
c_{+}(x_j) \approx \frac{N^{+}_j}{\Delta V},
\qquad
c_{-}(x_j) \approx \frac{N^{-}_j}{\Delta V}
$$

Substitute into the expression for \( \rho_c \) above.

**Statistical rescaling:**  
If the real system contains \( N_{\text{real}} \) ions but we simulate \( N_{\text{sim}} \gg N_{\text{real}} \), then

$$
\rho_c^{real}(x) = \frac{N_{real}}{N_{sim}} \, \rho_c^{sim}(x)
$$

---

## Field Solution Strategy
At selected coupling times \( t_k \):

1. **Aggregate positions:** collect particle coordinates \( \{ x_i(t_k) \} \).
2. **Form \( \rho_c(x) \):** bin particles into voxels.
3. **Solve Poisson:** compute \( \Phi \) and \( E = -\nabla \Phi \).
4. **Update field:** inject the new field into the particle simulator.
5. **Advance particles:** integrate the next block of Brownian steps with electrostatic forces.

Coupling does not occur every micro-step.  
Instead, choose a stride \( n_{BD} \): particles advance \( n_{BD} \) steps between field updates.
