# MEMS Portfolio — Electrostatically Actuated Cantilever: Squeeze-Film Damping & Thermal Qualification

This repository documents my progression through a set of self-directed extensions to COMSOL's **Electrostatically Actuated Cantilever** application model (COMSOL Application Library, model 444), used as a starting point to build an original MEMS simulation portfolio.

The base model provides a static electromechanical pull-in demo (Electrostatics + Solid Mechanics, coupled via Electromechanical Forces, with a Moving Mesh/ALE formulation in the air gap). The goal of this project is to extend that base model with additional physics that the original app does not include, turning it into a small-signal resonator characterization tool and, eventually, an environmental-qualification tool relevant to space-deployed MEMS.

## Status

| Enhancement | Status |
|---|---|
| **1 — Squeeze-film damping & dynamic frequency response** | ✅ Complete |
| **2 — Thermal-electromechanical space-environment qualification** | 🔄 In progress |

---

## Enhancement 1 — Squeeze-Film Damping and Dynamic Frequency Response

**Physics added:** a Thin-Film Flow interface solving the (modified) Reynolds equation in the gas gap beneath the cantilever, coupled to Solid Mechanics via COMSOL's predefined **Solid–Thin-Film Damping** multiphysics interface. This captures the gas damping the base model omits entirely, and sets the actual Q-factor of the device as a resonator.

### What was built

1. Added the predefined **Solid–Thin-Film Damping** interface (Structural Mechanics → Fluid-Structure Interaction), which auto-generated the Thin-Film Flow interface and the Structure–Thin-Film Flow Interaction coupling node.
2. Assigned **Thin-Film Flow** to the boundary on the underside of the cantilever facing the substrate (boundary selection, not a domain — the Reynolds equation formulation treats the gas layer as infinitely thin relative to its lateral extent).
3. Configured **Fluid-Film Properties**:
   - Fluid type: Gas (modified Reynolds equation)
   - Reference pressure `p_amb`, defined as a global parameter and swept from atmospheric (1×10⁵ Pa) down toward vacuum (0.1 Pa) on a log-spaced list, to move the system from the overdamped/heavily-damped regime toward underdamped resonance.
4. Noted the model's physical validity limit: the modified Reynolds equation includes a rarefaction/slip-flow correction valid up to roughly Kn ≈ 1; below that, deep-vacuum results should be interpreted with the continuum-based fluid model's limitations in mind rather than taken at face value.
5. Set up the study sequence: **Stationary** (DC electromechanical operating point) → **Eigenfrequency** (undamped/lightly-damped reference modes, swept across the same pressure list) → **Frequency Domain, Perturbation** (small-signal response around the Stationary operating point, driven by an explicit harmonic Boundary Load on the cantilever, disabled for the Stationary step and enabled only for the perturbation step).

### Debugging notes (kept here since they're part of the actual engineering record)

- **"Multiple moving frame specifications on the same selection" (Domain 2):** adding the predefined Solid–Thin-Film Damping interface silently created a *second*, redundant Solid Mechanics interface (`solid2`) covering the same domain as the original `solid` interface. Fixed by repointing the Structure–Thin-Film Flow Interaction coupling to the original Solid Mechanics interface and deleting the duplicate, after confirming the fixed-constraint and electromechanical coupling still referenced the surviving interface.
- **"Unknown model parameter f0_est, property: plist (frequencies)":** the frequency-range expression referenced a parameter, `f0_est`, that had never actually been defined in Global Definitions. Fixed by reading the fundamental flexural eigenfrequency off the Eigenfrequency study results and defining `f0_est` explicitly as a global parameter with units.
- **Frequency Domain, Perturbation returning all-zero fields:** root cause was twofold —
  1. `f0_est` had been entered with an incorrect order of magnitude (~14 Hz instead of ~25,000 Hz), so the swept frequency range never came close to the real resonance.
  2. No harmonic excitation had actually been added to the perturbation step, so the linearized problem was homogeneous and the only consistent solution was identically zero.

  Fixed by (a) correcting `f0_est` against the real part of the fundamental mechanical eigenvalue (~24,800–25,600 Hz across the pressure sweep), (b) increasing the number of eigenfrequencies requested and setting a search point near the expected mechanical frequency so the solver reliably found the mechanical bending mode rather than the Thin-Film Flow's own purely-imaginary fluid relaxation modes, and (c) adding an explicit small Boundary Load on the cantilever, active only during the Frequency Domain, Perturbation step, to actually drive the system.

### Results produced

- Q-factor vs. ambient pressure, extracted from the −3 dB (half-power, amplitude = peak/√2) bandwidth of the tip displacement frequency response at each swept pressure.
- Resonance frequency shift across the pressure sweep, comparing the fluid-coupled resonance against the pressure-independent Eigenfrequency-only reference.
- Identification of the overdamped regime at atmospheric pressure (no resolvable resonance peak) versus the underdamped regime at low pressure, and the point at which the fluid model's Kn ≈ 1 validity limit is approached.

---

## Enhancement 2 — Thermal-Electromechanical Space-Environment Qualification *(in progress)*

**Physics being added:** Heat Transfer in Solids, coupled into Solid Mechanics via the Thermal Expansion multiphysics node, with temperature-dependent material properties, to evaluate how pull-in voltage and resonance frequency shift across a space-relevant thermal cycle.

### Planned steps

1. Add **Heat Transfer in Solids** and the **Thermal Expansion** coupling node to the existing Solid Mechanics interface.
2. Replace fixed Young's modulus and CTE values with **temperature-dependent interpolation functions** built from literature data for the structural material.
3. Apply a uniform prescribed temperature (isothermal soak) at the anchor as the thermal boundary condition, representing substrate/chassis thermal control rather than a full transient gradient.
4. Optionally add an **Initial Stress and Strain** subnode to capture fabrication-induced residual stress, since this is itself temperature-dependent.
5. Study sequence: Stationary (Heat Transfer) → Stationary (Solid Mechanics + Electrostatics, inheriting thermal strain) → Parametric Sweep over ambient temperature across a representative range (e.g., -55 °C to +85 °C).
6. Target result: pull-in voltage vs. temperature and resonance frequency vs. temperature, framed around whether the actuator stays within its designed operating window across the mission thermal environment, or drifts toward unintended pull-in — a realistic failure mode for space-deployed MEMS switches.

*This section will be updated with implementation notes, debugging record, and results as the work progresses — following the same format as Enhancement 1 above.*

---

## Base Model Reference

- COMSOL Application Library: [Electrostatically Actuated Cantilever](https://www.comsol.com/model/electrostatically-actuated-cantilever-444)
