# PFLOTRAN-JH: BSA and porosity–permeability functions

This guide covers the added mineral bulk surface area (BSA) functions and the selectable porosity–permeability relationship in this source tree.

Examples are input fragments to merge into an existing PFLOTRAN model. Retain your database, species, mineral declarations, kinetic rates, initial conditions and material assignments. Numerical values below illustrate syntax and are not calibrated parameters.

## Mineral bulk surface area (BSA)

Select a function inside each mineral's block under `CHEMISTRY / MINERAL_KINETICS`. BSA is mineral surface area per bulk volume, in `m^2/m^3`.

```text
CHEMISTRY
  UPDATE_POROSITY
  MINERAL_KINETICS
    Calcite
      RATE_CONSTANT 1.d-10
      SURFACE_AREA_FUNCTION CF_PRIMARY_DISSOLUTION
      A 0.6666666666666667
      B 0.6666666666666667
    /
  /
END
```

Use your mineral's database name instead of `Calcite`. Define its initial volume fraction and surface area in the existing mineral constraint. For the CF functions, that initial surface area supplies the reference BSA. Include `UPDATE_POROSITY` so porosity-dependent functions respond to mineral reactions. Selecting a surface-area function enables its surface-area updates; the old `UPDATE_MINERAL_SURFACE_AREA` keyword is obsolete.

In the equations below, $S$ is the updated BSA and $S_0$ is the initial BSA, floored by `SPECIFIC_SURFACE_AREA_EPSILON`. Both have units of $\mathrm{m^2/m^3}$ bulk. Porosity $\phi$ is current base porosity and $\phi_0$ is initial porosity. Mineral volume fractions $v$ and $v_0$ are current and initial values.

### CF_PRECIPITATION

```text
SURFACE_AREA_FUNCTION CF_PRECIPITATION
A 0.6666666666666667
```

$$
S = S_0 \left(\frac{\phi}{\phi_0}\right)^A
$$

The exponent $A$ corresponds to `A` or `SURFACE_AREA_POROSITY_POWER` and defaults to $2/3$.

Use this to scale reference BSA with porosity alone. `B` is not accepted for this function.

### CF_PRIMARY_DISSOLUTION

```text
SURFACE_AREA_FUNCTION CF_PRIMARY_DISSOLUTION
A 0.6666666666666667
B 0.6666666666666667
```

$$
S = S_0 \left(\frac{\phi}{\phi_0}\right)^A
        \left(\frac{v}{v_0}\right)^B
$$

The porosity exponent $A$ is set by `A` or `SURFACE_AREA_POROSITY_POWER`. The mineral volume-fraction exponent $B$ is set by `B` or `SURFACE_AREA_VOL_FRAC_POWER`. Both default to $2/3$.

This scales BSA with porosity and the fraction of the initial mineral volume remaining. Set the initial mineral volume fraction and surface area consistently with the mineral inventory.

### CF_SECONDARY_DISSOLUTION

```text
SURFACE_AREA_FUNCTION CF_SECONDARY_DISSOLUTION
A 0.6666666666666667
B 0.6666666666666667
```

$$
S = S_0 \left(\frac{\phi}{\phi_0}\right)^A v^B
$$

As above, $A$ corresponds to `A` or `SURFACE_AREA_POROSITY_POWER`, and $B$ to `B` or `SURFACE_AREA_VOL_FRAC_POWER`; both default to $2/3$.

This uses the current mineral volume fraction directly, without dividing by its initial value. Supply a nonzero reference surface area or an appropriate area epsilon if the mineral starts absent and should later react through this function.

For all three CF functions, the implementation uses `max(phi, 0)` and floors `phi0` at `1e-16`. The dissolution functions use `max(v, 0)`; primary dissolution also floors `v0` at `1e-16`. These guards are part of the implemented equations. Choose exponents that remain meaningful when porosity or mineral volume approaches zero.

The CF names select area formulas. They do not independently restrict reaction direction; precipitation and dissolution still depend on the kinetic rate and affinity calculation.

### MINERAL_MASS_CAPPED

```text
SURFACE_AREA_FUNCTION MINERAL_MASS_CAPPED
SPECIFIC_SURFACE_AREA 100.d0 m^2/kg
SURFACE_AREA_POROSITY_THRESHOLD 0.05d0
```

$$
S = \begin{cases}
0, & \phi < \phi_{\mathrm{threshold}}, \\
a_s\rho_m v, & \phi \geq \phi_{\mathrm{threshold}}.
\end{cases}
$$

Here $a_s$ is `SPECIFIC_SURFACE_AREA`, read in $\mathrm{m^2/kg}$ by default, and $\phi_{\mathrm{threshold}}$ is `SURFACE_AREA_POROSITY_THRESHOLD`. Both keywords are required. Mineral density $\rho_m$ is calculated from molar mass divided by molar volume. The threshold is a porosity fraction, not a percentage.

At exactly the threshold, the mass-based expression still applies. This is a cutoff rather than a smooth taper: setting BSA to zero suppresses the mineral's area-dependent kinetic rate. The area is recalculated on later updates, so it can become nonzero again if porosity recovers. Explicit `A` or `B` values are rejected for this function. `UPDATE_POROSITY` is required.

### TARGET_REPLACEMENT_MASS

```text
SURFACE_AREA_FUNCTION TARGET_REPLACEMENT_MASS
SPECIFIC_SURFACE_AREA 100.d0 m^2/kg
TARGET_REPLACEMENT_MINERAL Calcite
```

Place these lines in the block for the **replacement mineral**, using another kinetic mineral's name as the target. Both `SPECIFIC_SURFACE_AREA` and `TARGET_REPLACEMENT_MINERAL` are required. The target must resolve to a kinetic mineral and cannot be the replacement mineral itself.

$$
S = \begin{cases}
0, & v-v_0 > v_{\mathrm{target},0}, \\
a_s\rho_m v, & v-v_0 \leq v_{\mathrm{target},0}.
\end{cases}
$$

Here $a_s$ is `SPECIFIC_SURFACE_AREA` in $\mathrm{m^2/kg}$ and $\rho_m$ is the replacement mineral's density. The value $v_{\mathrm{target},0}$ is the initial volume fraction of the mineral named by `TARGET_REPLACEMENT_MINERAL`.

The cutoff compares growth in the replacement mineral's volume fraction with the target's **initial** volume fraction. It does not track the target's current remaining mass or enforce a stoichiometric replacement ratio. Equality still uses the mass-based area expression. The cutoff acts when the area is updated, so it does not impose an exact within-step cap on precipitated volume. Explicit `A` or `B` values are rejected.

## Porosity–permeability relationship

Enable reaction-driven property updates in `CHEMISTRY`:

```text
CHEMISTRY
  UPDATE_POROSITY
  UPDATE_PERMEABILITY
END
```

Merge these flags into your existing chemistry block. `UPDATE_PERMEABILITY` requires `UPDATE_POROSITY`. Select the relationship and its parameters inside each relevant `MATERIAL_PROPERTY` block.

### GENERALIZED_KOZENY_CARMAN

```text
MATERIAL_PROPERTY rock
  ID 1
  POROSITY 0.30d0
  PERMEABILITY
    PERM_ISO 1.d-12
  /
  POROSITY_PERMEABILITY_FUNCTION GENERALIZED_KOZENY_CARMAN
  POROSITY_PERMEABILITY_EXPONENT_A 2.d0
  POROSITY_PERMEABILITY_EXPONENT_B 3.d0
  PERMEABILITY_MIN_SCALE_FACTOR 1.d-6
END
```

The implemented relation is:

$$
f(\phi) = \left(\frac{1-\phi_0}{1-\phi}\right)^a
           \left(\frac{\phi}{\phi_0}\right)^b,
\qquad
k = k_0\max\left(f_{\min}, f(\phi)\right).
$$

The exponent $a$ on the solid-fraction ratio corresponds to `POROSITY_PERMEABILITY_EXPONENT_A`. The exponent $b$ on the porosity ratio corresponds to `POROSITY_PERMEABILITY_EXPONENT_B`. Both are required and have no default. These material parameters are separate from the `A` and `B` aliases in mineral kinetics.

The lower bound $f_{\min}$ is `PERMEABILITY_MIN_SCALE_FACTOR`, a dimensionless minimum for $k/k_0$ that defaults to zero. Here $\phi$ is current base porosity, $\phi_0$ is initial porosity, and $k_0$ is initial permeability. Both porosities must lie strictly between zero and one; otherwise the update reports an error. The minimum scale does not bypass this validity check. The example uses $a=2$ and $b=3$; both must be supplied even when using those values.

The same scale multiplies every diagonal permeability component and, when a full tensor is enabled, every off-diagonal component. At the initial porosity the raw scale is one. There is no upper cap on the scale.

### CRITICAL_POROSITY_POWER

The existing critical-porosity law remains the default. It can now also be selected explicitly:

```text
POROSITY_PERMEABILITY_FUNCTION CRITICAL_POROSITY_POWER
PERMEABILITY_POWER 3.d0
PERMEABILITY_CRITICAL_POROSITY 0.05d0
PERMEABILITY_MIN_SCALE_FACTOR 1.d-6
```

$$
f(\phi) = \begin{cases}
\left(\dfrac{\phi-\phi_c}{\phi_0-\phi_c}\right)^n,
  & \phi>\phi_c\ \text{and}\ \phi_0>\phi_c, \\
0, & \text{otherwise},
\end{cases}
\qquad
k = k_0\max\left(f_{\min}, f(\phi)\right).
$$

The exponent $n$ corresponds to `PERMEABILITY_POWER` and defaults to one. The critical porosity $\phi_c$ corresponds to `PERMEABILITY_CRITICAL_POROSITY` and defaults to zero. The lower bound $f_{\min}$ is `PERMEABILITY_MIN_SCALE_FACTOR` and defaults to zero. As above, $\phi$ is current base porosity, $\phi_0$ is initial porosity, and $k_0$ is initial permeability. The generalized Kozeny–Carman law does not use `PERMEABILITY_POWER` or `PERMEABILITY_CRITICAL_POROSITY`.

Implementation: [mineral functions](src/pflotran/reaction_mineral.F90), [mineral defaults](src/pflotran/reaction_mineral_aux.F90), [material input](src/pflotran/material.F90), and [property updates](src/pflotran/realization_subsurface.F90).

The examples were checked against the source parser and equations; they are not complete simulation decks and were not run as simulations. Existing license and copyright notices are in [LICENSE](LICENSE) and [COPYRIGHT](COPYRIGHT).
