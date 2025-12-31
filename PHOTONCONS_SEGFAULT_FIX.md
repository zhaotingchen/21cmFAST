# Photon Conservation Segmentation Fault Fix

## Problem Analysis

The segmentation fault was occurring in `gsl_interp_eval` called from `determine_deltaz_for_photoncons` and `adjust_redshifts_for_photoncons`. The stack trace showed:

```
[ 1] /lib/x86_64-linux-gnu/libgsl.so.27(gsl_interp_eval+0xc)
[ 2] determine_deltaz_for_photoncons+0x162
[ 3] adjust_redshifts_for_photoncons+0x343
```

**Root Cause**: Thread-safety race condition in photon conservation initialization:

1. **Race Condition on `photon_cons_allocated` flag**: The global `bool photon_cons_allocated` was checked and set without synchronization. Multiple threads could:
   - Both see `photon_cons_allocated == false` simultaneously
   - Both call `determine_deltaz_for_photoncons()` concurrently
   - Cause memory corruption or NULL pointer access

2. **Missing NULL checks**: The code didn't verify that GSL spline allocation succeeded before using the splines, leading to NULL pointer dereferences.

3. **Uninitialized spline access**: Threads could call `gsl_spline_eval()` before initialization completed, accessing NULL or partially initialized splines.

## Solution Implemented

### 1. Added OpenMP Support
- Added `#include <omp.h>` for thread synchronization

### 2. Protected Initialization with Critical Section
- Wrapped the initialization check and execution in `#pragma omp critical(photoncons_init)`
- Ensures only one thread initializes the splines, even if multiple threads call `adjust_redshifts_for_photoncons()` simultaneously

### 3. Added Safety Checks
- Added NULL pointer checks after GSL allocation in `determine_deltaz_for_photoncons()`
- Added validation checks before using splines in `adjust_redshifts_for_photoncons()`
- Ensures splines are properly initialized before evaluation

### 4. Improved Error Handling
- Added proper cleanup on allocation failure
- Added error logging for debugging

## Code Changes

### File: `src/py21cmfast/src/photoncons.c`

1. **Added OpenMP include** (line ~23):
```c
#include <omp.h>
```

2. **Protected initialization in `adjust_redshifts_for_photoncons()`** (lines ~750-760, ~790-800):
```c
// Thread-safe initialization: only one thread initializes, others wait
#pragma omp critical(photoncons_init)
{
    if (!photon_cons_allocated) {
        determine_deltaz_for_photoncons();
        photon_cons_allocated = true;
    }
}

// Safety check: ensure spline is initialized before use
if (!photon_cons_allocated || deltaz_spline_for_photoncons == NULL ||
    deltaz_spline_for_photoncons_acc == NULL) {
    LOG_ERROR("adjust_redshifts_for_photoncons: Spline not properly initialized.");
    Throw(PhotonConsError);
}
```

3. **Added NULL checks after allocation in `determine_deltaz_for_photoncons()`** (lines ~452-465, ~679-692):
```c
deltaz_spline_for_photoncons_acc = gsl_interp_accel_alloc();
if (deltaz_spline_for_photoncons_acc == NULL) {
    LOG_ERROR("determine_deltaz_for_photoncons: Failed to allocate GSL interpolation accelerator.");
    Throw(MemoryAllocError);
}

deltaz_spline_for_photoncons = gsl_spline_alloc(gsl_interp_linear, ...);
if (deltaz_spline_for_photoncons == NULL) {
    gsl_interp_accel_free(deltaz_spline_for_photoncons_acc);
    deltaz_spline_for_photoncons_acc = NULL;
    LOG_ERROR("determine_deltaz_for_photoncons: Failed to allocate GSL spline.");
    Throw(MemoryAllocError);
}
```

4. **Improved cleanup in free function** (lines ~1013-1022):
```c
// Safely free deltaz spline (handle NULL pointers)
if (deltaz_spline_for_photoncons != NULL) {
    gsl_spline_free(deltaz_spline_for_photoncons);
    deltaz_spline_for_photoncons = NULL;
}
if (deltaz_spline_for_photoncons_acc != NULL) {
    gsl_interp_accel_free(deltaz_spline_for_photoncons_acc);
    deltaz_spline_for_photoncons_acc = NULL;
}
```

## How It Works

1. **First thread** to reach initialization:
   - Enters critical section
   - Checks `photon_cons_allocated` (false)
   - Calls `determine_deltaz_for_photoncons()` to initialize splines
   - Sets `photon_cons_allocated = true`
   - Exits critical section

2. **Other threads**:
   - Wait at critical section
   - Enter after first thread completes
   - See `photon_cons_allocated == true`
   - Skip initialization
   - Proceed to use already-initialized splines

3. **Safety checks** ensure:
   - Splines are not NULL before use
   - Initialization completed successfully
   - Proper error handling if something goes wrong

## Testing Recommendations

1. **Multi-threaded testing**: Run simulations with multiple threads to verify race condition is fixed
2. **Stress testing**: Run many simulations in parallel on HPC to catch any remaining edge cases
3. **Memory checking**: Use valgrind or similar tools to verify no memory leaks or corruption
4. **Error path testing**: Verify error handling works correctly if allocation fails

## Expected Behavior

- **Before fix**: Intermittent segmentation faults when multiple threads accessed photon conservation functions
- **After fix**: Thread-safe initialization ensures only one thread initializes, all others wait and use the initialized splines safely

## Related Issues

This fix addresses the same type of thread-safety issue as the `Energy_Lya_heating` fix, but in a different part of the codebase. Both involve:
- Static/global variables shared across threads
- Initialization that must happen exactly once
- Race conditions in multi-threaded environments
