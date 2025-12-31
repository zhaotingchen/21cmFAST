# Comprehensive Thread-Safety Fixes

## Summary

This document summarizes all thread-safety fixes applied to prevent race conditions in multi-threaded execution environments (HPC clusters).

## Fixed Issues

### 1. `heating_helper_progs.c`

#### Fixed Functions:
- **`Energy_Lya_heating()`** - Static arrays `dEC` and `dEI` initialization
- **`T_RECFAST()`** - Static GSL spline initialization with flag-based pattern
- **`xion_RECFAST()`** - Static GSL spline initialization with flag-based pattern
- **`spectral_emissivity()`** - Static arrays initialization with flag-based pattern

#### Changes:
- Added `#include <omp.h>`
- Added initialization flags (`lya_heating_initialized`, `trecfast_initialized`, `xionrecfast_initialized`, `spectral_emissivity_initialized`)
- Protected initialization with `#pragma omp critical` sections
- Added NULL pointer checks after GSL allocation
- Added safety checks before using splines/arrays

### 2. `photoncons.c`

#### Fixed Functions:
- **`determine_deltaz_for_photoncons()`** - Static GSL spline initialization
- **`adjust_redshifts_for_photoncons()`** - Uses splines initialized by `determine_deltaz_for_photoncons()`

#### Changes:
- Added `#include <omp.h>`
- Protected `photon_cons_allocated` flag check with `#pragma omp critical(photoncons_init)`
- Added NULL pointer checks after GSL allocation
- Added safety checks before using splines

### 3. `recombinations.c`

#### Fixed Functions:
- **`init_MHR()`** - Initializes multiple static GSL splines
- **`init_A_MHR()`** - Static GSL spline for A parameter
- **`init_C_MHR()`** - Static GSL spline for C parameter
- **`init_beta_MHR()`** - Static GSL spline for beta parameter
- **`splined_recombination_rate()`** - Uses RR_spline arrays
- **`splined_A_MHR()`**, **`splined_C_MHR()`**, **`splined_beta_MHR()`** - Use respective splines

#### Changes:
- Added `#include <omp.h>`
- Added `mhr_initialized` flag
- Protected `init_MHR()` with `#pragma omp critical(mhr_init)`
- Added NULL pointer checks after all GSL allocations
- Added safety checks in all splined functions before use
- Improved error handling with cleanup on failure

### 4. `cosmology.c`

#### Fixed Functions:
- **`init_ps()`** - Modifies static `cosmo_consts` struct

#### Changes:
- Added `ps_initialized` flag (OpenMP already included)
- Protected initialization with `#pragma omp critical(init_ps)`
- Added NULL check for GSL workspace allocation
- Improved error handling

### 5. `interp_tables.c`

#### Fixed Functions:
- **`initialise_SFRD_spline()`** - Allocation checks for `SFRD_z_table` and `SFRD_z_table_MINI`
- **`initialise_Nion_Ts_spline()`** - Allocation checks for `Nion_z_table` and `Nion_z_table_MINI`

#### Changes:
- Added `#include <omp.h>`
- Protected allocation checks with `#pragma omp critical` sections
- Note: Other initialization functions in this file follow similar patterns but may be called in contexts where thread-safety is already ensured. Additional protection can be added if race conditions are observed.

## Pattern Used

All fixes follow a consistent pattern:

1. **Add initialization flag**: `static int function_initialized = 0;`

2. **Protect initialization**:
```c
#pragma omp critical(critical_section_name)
{
    if (!function_initialized) {
        // ... initialization code ...
        function_initialized = 1;
    }
}
```

3. **Add safety checks**:
```c
if (!function_initialized || pointer == NULL) {
    LOG_ERROR("Function not initialized");
    Throw(ValueError);
}
```

4. **Add NULL checks after allocation**:
```c
ptr = gsl_spline_alloc(...);
if (ptr == NULL) {
    // cleanup and error handling
    Throw(MemoryAllocError);
}
```

## Testing Recommendations

1. **Multi-threaded testing**: Run simulations with `N_THREADS > 1` to verify fixes
2. **Stress testing**: Run many simulations in parallel on HPC
3. **Memory checking**: Use valgrind or similar tools to detect memory issues
4. **Race condition detection**: Use thread sanitizer (`-fsanitize=thread`) if available

## Remaining Potential Issues

The following areas may need additional protection if race conditions are observed:

1. **`interp_tables.c`**: Other initialization functions with `.allocated` flag checks
2. **File I/O operations**: Some file reads might benefit from additional protection
3. **Global state modifications**: Any other functions modifying global/static state

## Files Modified

- `src/py21cmfast/src/heating_helper_progs.c`
- `src/py21cmfast/src/photoncons.c`
- `src/py21cmfast/src/recombinations.c`
- `src/py21cmfast/src/cosmology.c`
- `src/py21cmfast/src/interp_tables.c`

## Expected Impact

- **Before**: Intermittent segmentation faults due to race conditions in multi-threaded environments
- **After**: Thread-safe initialization ensures only one thread initializes shared resources, all others wait and use initialized resources safely
