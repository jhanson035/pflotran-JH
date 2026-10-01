# PFLOTRAN CK1: BSA and porosity–permeability functions

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

### Symbols and parameters

| Symbol / keyword | Meaning |
| --- | --- |
| `S` | Updated BSA, in `m^2/m^3` bulk. |
| `S0` | Initial BSA, floored by `SPECIFIC_SURFACE_AREA_EPSILON`. |
| `phi`, `phi0` | Current base porosity and initial porosity, as fractions. |
| `v`, `v0` | Current and initial mineral volume fractions. |
| `A` | Porosity exponent; alias for `SURFACE_AREA_POROSITY_POWER`. Default: `2/3`. |
| `B` | Mineral volume-fraction exponent; alias for `SURFACE_AREA_VOL_FRAC_POWER`. Default: `2/3`. |
| `SPECIFIC_SURFACE_AREA` | Surface area per mineral mass, read in `m^2/kg` by default; required by the mass-based functions below. |

The `A` and `B` aliases belong to mineral kinetics. They are separate from the porosity–permeability exponents described later.

### CF_PRECIPITATION

```text
SURFACE_AREA_FUNCTION CF_PRECIPITATION
A 0.6666666666666667
```

```text
S = S0 * (phi / phi0)^A
```

Use this to scale reference BSA with porosity alone. `B` is not accepted for this function.

### CF_PRIMARY_DISSOLUTION

```text
SURFACE_AREA_FUNCTION CF_PRIMARY_DISSOLUTION
A 0.6666666666666667
B 0.6666666666666667
```

```text
S = S0 * (phi / phi0)^A * (v / v0)^B
```

This scales BSA with porosity and the fraction of the initial mineral volume remaining. Set the initial mineral volume fraction and surface area consistently with the mineral inventory.

### CF_SECONDARY_DISSOLUTION

```text
SURFACE_AREA_FUNCTION CF_SECONDARY_DISSOLUTION
A 0.6666666666666667
B 0.6666666666666667
```

```text
S = S0 * (phi / phi0)^A * v^B
```

This uses the current mineral volume fraction directly, without dividing by its initial value. Supply a nonzero reference surface area or an appropriate area epsilon if the mineral starts absent and should later react through this function.

For all three CF functions, the implementation uses `max(phi, 0)` and floors `phi0` at `1e-16`. The dissolution functions use `max(v, 0)`; primary dissolution also floors `v0` at `1e-16`. These guards are part of the implemented equations. Choose exponents that remain meaningful when porosity or mineral volume approaches zero.

The CF names select area formulas. They do not independently restrict reaction direction; precipitation and dissolution still depend on the kinetic rate and affinity calculation.

### MINERAL_MASS_CAPPED

```text
SURFACE_AREA_FUNCTION MINERAL_MASS_CAPPED
SPECIFIC_SURFACE_AREA 100.d0 m^2/kg
SURFACE_AREA_POROSITY_THRESHOLD 0.05d0
```

```text
S = 0                         if phi < threshold
S = specific_area * rho_m * v otherwise
```

`rho_m` is mineral density, calculated from molar mass divided by molar volume. Both keywords shown after the function selector are required. The threshold is a porosity fraction, not a percentage.

At exactly the threshold, the mass-based expression still applies. This is a cutoff rather than a smooth taper: setting BSA to zero suppresses the mineral's area-dependent kinetic rate. The area is recalculated on later updates, so it can become nonzero again if porosity recovers. Explicit `A` or `B` values are rejected for this function. `UPDATE_POROSITY` is required.

### TARGET_REPLACEMENT_MASS

```text
SURFACE_AREA_FUNCTION TARGET_REPLACEMENT_MASS
SPECIFIC_SURFACE_AREA 100.d0 m^2/kg
TARGET_REPLACEMENT_MINERAL Calcite
```

Place these lines in the block for the **replacement mineral**, using another kinetic mineral's name as the target. Both `SPECIFIC_SURFACE_AREA` and `TARGET_REPLACEMENT_MINERAL` are required. The target must resolve to a kinetic mineral and cannot be the replacement mineral itself.

```text
S = 0                         if v - v0 > v_target0
S = specific_area * rho_m * v otherwise
```

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

```text
scale = ((1 - phi0) / (1 - phi))^a * (phi / phi0)^b
k = k0 * max(min_scale, scale)
```

| Parameter | Meaning |
| --- | --- |
| `POROSITY_PERMEABILITY_EXPONENT_A` | Exponent `a` on the solid-fraction ratio. Required; no default. |
| `POROSITY_PERMEABILITY_EXPONENT_B` | Exponent `b` on the porosity ratio. Required; no default. |
| `PERMEABILITY_MIN_SCALE_FACTOR` | Dimensionless lower bound on `k/k0`; default `0`. |

Here `phi` is current base porosity, `phi0` is initial porosity, and `k0` is initial permeability. Both porosities must satisfy `0 < phi < 1`; otherwise the update reports an error. The minimum scale does not bypass this validity check. The example uses `a = 2` and `b = 3`; both must be supplied even when using those values.

The same scale multiplies every diagonal permeability component and, when a full tensor is enabled, every off-diagonal component. At the initial porosity the raw scale is one. There is no upper cap on the scale.

### CRITICAL_POROSITY_POWER

The existing critical-porosity law remains the default. It can now also be selected explicitly:

```text
POROSITY_PERMEABILITY_FUNCTION CRITICAL_POROSITY_POWER
PERMEABILITY_POWER 3.d0
PERMEABILITY_CRITICAL_POROSITY 0.05d0
PERMEABILITY_MIN_SCALE_FACTOR 1.d-6
```

```text
scale = ((phi - phi_c) / (phi0 - phi_c))^n
k = k0 * max(min_scale, scale)
```

The raw scale is zero unless both current base porosity and initial porosity exceed `phi_c`. Defaults are `n = 1`, `phi_c = 0`, and `min_scale = 0`. The generalized Kozeny–Carman law does not use `PERMEABILITY_POWER` or `PERMEABILITY_CRITICAL_POROSITY`.

## Using both features

Choose a BSA function separately for each kinetic mineral and a porosity–permeability function separately for each material. With both update flags enabled, the property-update routine updates mineral porosity, then mineral surface area, then permeability. Thus mineral reactions can change both reactive area and permeability through the evolving base porosity.

Implementation: [mineral functions](src/pflotran/reaction_mineral.F90), [mineral defaults](src/pflotran/reaction_mineral_aux.F90), [material input](src/pflotran/material.F90), and [property updates](src/pflotran/realization_subsurface.F90).

The examples were checked against the source parser and equations; they are not complete simulation decks and were not run as simulations. Existing license and copyright notices are in [LICENSE](LICENSE) and [COPYRIGHT](COPYRIGHT).
