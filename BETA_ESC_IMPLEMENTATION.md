# Implementation of BETA_ESC Parameter for Redshift-Dependent Escape Fraction

## Overview

This modification adds a new parameter `BETA_ESC` to the py21cmfast cosmic reionization simulation package. This parameter introduces redshift dependence to the escape fraction calculation, changing it from:

```
fesc = min[1, fesc_10 * (M/1e10)^alpha_esc]
```

to:

```
fesc = min[1, fesc_10 * (M/1e10)^alpha_esc * ((1+z)/8)^beta_esc]
```

## Changes Made

### 1. Python Interface (`src/py21cmfast/wrapper/inputs.py`)

- Added `BETA_ESC` field to the `AstroParams` class with default value `0.0`
- Added documentation explaining the parameter's purpose and behavior
- The parameter follows the same pattern as other escape fraction parameters

### 2. C Structures

#### `src/py21cmfast/src/_inputparams_wrapper.h`
- Added `float BETA_ESC` to the `AstroParams` struct

#### `src/py21cmfast/src/scaling_relations.h`
- Added `double beta_esc` to the `ScalingConstants` struct

### 3. Parameter Initialization (`src/py21cmfast/src/scaling_relations.c`)

- Added initialization of `consts->beta_esc = astro_params_global->BETA_ESC`
- Added `beta_esc = 0.0` to the special case where escape fraction is disabled
- Updated debug logging to include the new parameter

### 4. Escape Fraction Calculations

#### `src/py21cmfast/src/HaloBox.c`
- Modified the escape fraction calculation to include redshift dependence:
  ```c
  double redshift_factor = pow((1.0 + consts->redshift) / 8.0, consts->beta_esc);
  fesc = fmin(consts->fesc_10 * pow(halo_mass / 1e10, consts->alpha_esc) * redshift_factor, 1);
  ```
- Applied the same modification to mini-halo escape fractions

#### `src/py21cmfast/src/hmf.c`
- Added `beta_esc` field to the `parameters_gsl_MF_integrals` struct
- Modified `nion_fraction()` and `nion_fraction_mini()` functions to include redshift dependence in log-space:
  ```c
  double redshift_factor = log(pow((1.0 + p.redshift) / 8.0, p.beta_esc));
  ```
- Updated all struct initializations to include the new parameter

### 5. Debugging and Logging

#### `src/py21cmfast/src/debugging.c`
- Added `BETA_ESC` to the debug output format string and parameter list

### 6. Template Files

Updated the following template configuration files to include the new parameter:
- `src/py21cmfast/templates/park19.toml`
- `src/py21cmfast/templates/Munoz21.toml`
- `src/py21cmfast/templates/Qin20.toml`

All set `BETA_ESC = 0.0` by default to maintain backward compatibility.

## Usage

### Python Example

```python
from py21cmfast import AstroParams

# Default behavior (no redshift dependence)
astro_params_default = AstroParams()  # BETA_ESC = 0.0

# With redshift dependence
astro_params_custom = AstroParams(
    F_ESC10=-1.0,      # 10% escape fraction at 10^10 M_sun
    ALPHA_ESC=-0.5,    # Mass dependence
    BETA_ESC=0.3       # Redshift dependence: increases with redshift
)
```

### Physical Interpretation

- `BETA_ESC = 0.0`: No redshift dependence (default, backward compatible)
- `BETA_ESC > 0.0`: Escape fraction increases at higher redshift
- `BETA_ESC < 0.0`: Escape fraction decreases at higher redshift

The normalization factor `((1+z)/8)` is chosen so that at z=7 (1+z=8), the redshift factor equals 1.

## Backward Compatibility

All changes are fully backward compatible:
- Default value of `BETA_ESC = 0.0` produces no redshift dependence
- Existing simulations and parameter sets will work unchanged
- Template files include the new parameter with default values

## Testing

The implementation includes:
- Syntax validation of Python changes
- C struct consistency checks
- Template file updates with default values

## Files Modified

1. `src/py21cmfast/wrapper/inputs.py` - Python interface
2. `src/py21cmfast/src/_inputparams_wrapper.h` - C parameter struct
3. `src/py21cmfast/src/scaling_relations.h` - Scaling constants struct
4. `src/py21cmfast/src/scaling_relations.c` - Parameter initialization
5. `src/py21cmfast/src/HaloBox.c` - Main escape fraction calculation
6. `src/py21cmfast/src/hmf.c` - Halo mass function integrals
7. `src/py21cmfast/src/debugging.c` - Debug output
8. Template configuration files (3 files)

The implementation is consistent across all parts of the codebase and maintains the existing code style and patterns.