# Modern Fortran Style Guide

A practical style guide for modern Fortran (2008+), with emphasis on GPU-accelerated scientific computing using OpenACC and CUDA Fortran.

## Table of Contents

- [Naming Conventions](#naming-conventions)
- [Required Practices](#required-practices)
- [Forbidden Practices](#forbidden-practices)
- [Recommended Practices](#recommended-practices)
- [GPU Programming](#gpu-programming)
- [File Structure Template](#file-structure-template)

---

## Naming Conventions

### Files

- Module files: `module_name.f90` (lowercase with underscores)
- Preprocessed files: `module_name.F90` (uppercase extension for files needing preprocessing)
- Test files: `test_module_name.f90`
- One module per file (submodules may be separate)

### Modules

Use a short project prefix to namespace your modules and avoid collisions:

```fortran
! Good — prefixed with project abbreviation
module swe_solver           ! swe = shallow water equations
module swe_boundary
module swe_io_netcdf

! Bad
module ShallowWaterSolver   ! CamelCase
module solver               ! No prefix, collision risk
module sws                  ! Too cryptic
```

**Why prefix?**
- Avoids name collisions when combining libraries
- Makes `use` statements self-documenting (`use swe_solver` vs `use solver`)
- Easy to grep for all project modules

Choose a 2-4 letter prefix and use it consistently across the project.

### Derived Types

All derived types use the `_t` suffix with snake_case:

```fortran
! Good
type :: simulation_state_t
type :: boundary_config_t
type :: mesh_t

! Bad
type :: SimulationState    ! CamelCase, no suffix
type :: simulation_state   ! Missing _t suffix
type :: TState             ! Hungarian notation
```

### Variables and Procedures

Use snake_case with descriptive names:

```fortran
! Good
integer :: num_cells
real(wp) :: total_energy
subroutine compute_flux(state, flux)

! Bad
integer :: nCells          ! camelCase
integer :: nc              ! Too short
real(wp) :: E              ! Single letter (except loop indices)
```

**Exception:** Single-letter variables are acceptable for:
- Loop indices (`i`, `j`, `k`)
- Very local, obvious variables (`x`, `y`, `z` for coordinates)
- Mathematical formulas matching published notation

### Constants

Use UPPERCASE with underscores for parameters:

```fortran
! Good
real(wp), parameter :: GRAVITY = 9.81_wp
real(wp), parameter :: DRY_TOLERANCE = 1.0e-6_wp
integer, parameter :: MAX_ITERATIONS = 1000

! Bad
real(wp), parameter :: gravity = 9.81_wp       ! Lowercase
real(wp), parameter :: maxIterations = 1000    ! camelCase
```

---

## Required Practices

### Use Statements with `only` Clause

Always specify what you're importing:

```fortran
! Good
use iso_fortran_env, only: real64, int32
use netcdf, only: nf90_open, nf90_close, NF90_NOERR

! Bad — pollutes namespace, hides dependencies
use iso_fortran_env
use netcdf
```

### Implicit None

Always include `implicit none` in modules and programs:

```fortran
module my_module
   use iso_fortran_env, only: real64
   implicit none
   private
   ! ...
end module my_module
```

### Intent Declarations

Always declare intent for all procedure arguments:

```fortran
! Good
subroutine compute_flux(state, dt, flux)
   type(state_t), intent(in) :: state
   real(wp), intent(in) :: dt
   real(wp), intent(out) :: flux(:,:)

! Bad — intent unclear, error-prone
subroutine compute_flux(state, dt, flux)
   type(state_t) :: state
   real(wp) :: dt
   real(wp) :: flux(:,:)
```

### Private by Default

Modules should be `private` by default with explicit `public` declarations:

```fortran
module solver
   implicit none
   private

   public :: solver_t
   public :: initialize_solver
   public :: run_timestep

   type :: solver_t
      ! Public type
   end type

   type :: internal_workspace_t
      ! Stays private — implementation detail
   end type
end module
```

### Limit Procedure Arguments

Public procedures should have **6 or fewer arguments**. Group related arguments into derived types:

```fortran
! Bad — too many arguments, hard to maintain
subroutine run_simulation(nx, ny, dx, dy, dt, t_end, &
                          h, hu, hv, z, manning_n, &
                          bc_west, bc_east, bc_south, bc_north, &
                          output_interval, restart_file)

! Good — grouped into logical types
subroutine run_simulation(mesh, state, config, output)
   type(mesh_t), intent(in) :: mesh           ! nx, ny, dx, dy
   type(state_t), intent(inout) :: state      ! h, hu, hv, z
   type(config_t), intent(in) :: config       ! dt, t_end, friction, BCs
   type(output_t), intent(inout) :: output    ! interval, files
```

**Benefits:**
- Easier to add new parameters without changing signatures
- Self-documenting — related data is grouped
- Simpler call sites

**Performance note:** Derived type indirection introduces a small overhead from pointer chasing and potential cache misses. For performance-critical compute kernels called millions of times (e.g., flux calculations in RK2 substeps), explicit scalar/array arguments are acceptable. These hot kernels are typically stable and rarely modified, so the maintainability trade-off is worthwhile.

---

## Forbidden Practices

### No GOTO Statements

Use structured control flow:

```fortran
! Bad
   if (error) goto 999
   ! ...
999 continue
   print *, "Error occurred"

! Good
   if (error) then
      call handle_error("Error occurred")
      return
   end if
```

### No Arithmetic IF

Use `if-then-else` or `select case`:

```fortran
! Bad (Fortran 77)
   if (x) 10, 20, 30

! Good
   select case (sign(1, x))
   case (-1)
      ! negative
   case (0)
      ! zero
   case (1)
      ! positive
   end select
```

### No COMMON Blocks

Use module variables or derived types:

```fortran
! Bad
common /grid_data/ nx, ny, dx, dy

! Good
module grid
   implicit none
   type :: grid_t
      integer :: nx, ny
      real(wp) :: dx, dy
   end type
end module
```

### No EQUIVALENCE

Use proper type conversions or `transfer()` if absolutely necessary.

### No Fixed-Form Source

All code must be free-form (`.f90` / `.F90`). No `.f` or `.F` files.

### No Assumed-Size Arrays

Use assumed-shape arrays with explicit interfaces:

```fortran
! Bad
subroutine process(arr, n)
   real(wp) :: arr(*)    ! No bounds checking, no array intrinsics
   integer :: n

! Good
subroutine process(arr)
   real(wp), intent(in) :: arr(:)    ! Knows its own size
```

### No External Statements

Use modules for explicit interfaces:

```fortran
! Bad
external :: my_function
real(wp) :: my_function

! Good
use my_module, only: my_function
```

### No Implicit SAVE

Avoid module-level variables that retain state without explicit marking:

```fortran
! Bad — implicit save, hidden state
module bad_module
   integer :: call_count = 0    ! Retains value between calls (implicit save)
end module

! Good — explicit state management via types
module good_module
   type :: counter_t
      integer :: value = 0
   end type
end module

! Acceptable — explicit save keyword makes intent clear
module acceptable_module
   integer, save :: call_count = 0    ! Clearly marked as persistent
end module
```

If you must use module-level state, use the explicit `save` keyword to signal intent to readers. However, prefer encapsulating state in derived types passed as arguments.

---

## Recommended Practices

### Working Precision

Define a working precision parameter and use it consistently:

```fortran
module precision
   use iso_fortran_env, only: real64
   implicit none
   integer, parameter :: wp = real64    ! Working precision
end module

! Usage
use precision, only: wp
real(wp) :: energy
real(wp), parameter :: TOLERANCE = 1.0e-10_wp
```

Never use literal kind numbers:

```fortran
! Bad
real(8) :: x           ! Non-portable
real*8 :: y            ! Obsolete
double precision :: z  ! Obsolete

! Good
real(wp) :: x
```

### Prefer Allocatable Over Pointer

Use `allocatable` instead of `pointer` when possible:

```fortran
! Good — automatic cleanup, no leak risk
type :: data_t
   real(wp), allocatable :: values(:)
   real(wp), allocatable :: matrix(:,:)
end type

! Bad — manual cleanup required, leak risk
type :: data_t
   real(wp), pointer :: values(:) => null()
   real(wp), pointer :: matrix(:,:) => null()
end type
```

**Use pointers only when you need:**
- Aliasing existing data
- Linked data structures
- Polymorphic return types

### Pure and Elemental Procedures

Mark functions as `pure` when they have no side effects:

```fortran
pure function kinetic_energy(mass, velocity) result(ke)
   real(wp), intent(in) :: mass, velocity
   real(wp) :: ke
   ke = 0.5_wp * mass * velocity**2
end function
```

Use `elemental` for scalar operations that should work on arrays:

```fortran
elemental function degrees_to_radians(deg) result(rad)
   real(wp), intent(in) :: deg
   real(wp) :: rad
   rad = deg * PI / 180.0_wp
end function

! Works on scalars and arrays
angle_rad = degrees_to_radians(45.0_wp)           ! Scalar
angles_rad = degrees_to_radians(angles_deg)       ! Array
```

### Block Constructs for Scope

Use `block` to limit variable scope:

```fortran
subroutine process_data(data, result)
   real(wp), intent(in) :: data(:)
   real(wp), intent(out) :: result

   integer :: i

   result = 0.0_wp
   do i = 1, size(data)
      block
         real(wp) :: normalized, contribution
         normalized = data(i) / maxval(data)
         contribution = compute_contribution(normalized)
         result = result + contribution
      end block
   end do
end subroutine
```

### Associate for Readability

Use `associate` for long expressions:

```fortran
! Good — clear and readable
associate(h => state%water_depth, &
          u => state%velocity_x, &
          v => state%velocity_y)
   flux_x = h * u
   flux_y = h * v
   kinetic = 0.5_wp * h * (u**2 + v**2)
end associate

! Bad — repetitive and verbose
flux_x = state%water_depth * state%velocity_x
flux_y = state%water_depth * state%velocity_y
kinetic = 0.5_wp * state%water_depth * &
          (state%velocity_x**2 + state%velocity_y**2)
```

**Compiler caveat:** `associate` support varies across compilers and can produce unexpected behavior (aliasing issues, optimization failures). Flang tends to have the most robust support. Test thoroughly when using `associate` in performance-critical code, and consider falling back to explicit temporaries if you encounter issues with gfortran or nvfortran.

### No Magic Numbers

Use named constants:

```fortran
! Bad — what do these mean?
if (depth < 0.001) then
   velocity = 0.0
end if
tolerance = 1.0e-8

! Good — self-documenting
real(wp), parameter :: DRY_TOLERANCE = 1.0e-3_wp
real(wp), parameter :: CONVERGENCE_TOL = 1.0e-8_wp

if (depth < DRY_TOLERANCE) then
   velocity = 0.0_wp
end if
tolerance = CONVERGENCE_TOL
```

### Avoid Deep Nesting

Maximum 3-4 levels of indentation. Use early returns and `cycle`:

```fortran
! Bad — deeply nested
do i = 1, n
   if (active(i)) then
      if (valid(i)) then
         do j = 1, m
            if (connected(i,j)) then
               ! Actual work buried here
            end if
         end do
      end if
   end if
end do

! Good — flat structure
do i = 1, n
   if (.not. active(i)) cycle
   if (.not. valid(i)) cycle

   do j = 1, m
      if (.not. connected(i,j)) cycle
      ! Work at reasonable depth
   end do
end do
```

### Allocatable Character Strings

Use allocatable strings for dynamic content:

```fortran
! Good
character(len=:), allocatable :: filename
character(len=:), allocatable :: error_msg

filename = "output_" // to_string(step) // ".nc"
error_msg = "Failed to open: " // trim(filename)

! Bad — fixed length may truncate or waste memory
character(len=256) :: filename
character(len=80) :: error_msg
```

### Memory Management

For types with allocatable components, Fortran automatically deallocates when the object goes out of scope — explicit destroy routines are generally unnecessary for non-GPU code (pointers are the exception and require manual cleanup).

Use `block` constructs to control deallocation timing:

```fortran
subroutine process_large_data(input)
   real(wp), intent(in) :: input(:)

   ! Large workspace deallocated at end of block, not end of subroutine
   block
      type :: workspace_t
         real(wp), allocatable :: buffer(:)
         real(wp), allocatable :: matrix(:,:)
      end type

      type(workspace_t) :: work
      allocate(work%buffer(1000000))
      allocate(work%matrix(1000, 1000))
      ! ... use workspace ...
   end block   ! work deallocated here

   ! Continue with other operations, memory already freed
end subroutine
```

**When explicit destroy is needed:**
- Types containing pointers (no automatic cleanup)
- GPU memory management (OpenACC `!$acc exit data`, CUDA Fortran device arrays)
- Resources requiring ordered cleanup (file handles, MPI communicators)

### Do Concurrent (Use with Caution because of compiler portability)

`do concurrent` hints that iterations are independent:

```fortran
! Good candidate — no dependencies
do concurrent (i = 1:n)
   result(i) = input(i)**2
end do

! Bad candidate — has loop-carried dependency
do concurrent (i = 2:n)
   result(i) = result(i-1) + input(i)    ! Wrong!
end do
```

**Caveats:**
- Compiler support varies
- No `exit`, `cycle`, `return`, or `goto` inside
- When in doubt, use regular `do` loop

### Documentation

Use consistent documentation comments (compatible with FORD):

```fortran
type :: simulation_t
   !! Main simulation container
   !!
   !! Holds all state and configuration for a shallow water simulation.

   type(mesh_t) :: mesh
      !! Computational mesh (structured Cartesian)
   type(state_t) :: state
      !! Conserved variables (h, hu, hv)
   real(wp) :: time = 0.0_wp
      !! Current simulation time [s]
end type
```

---

## GPU Programming

### OpenACC Basics

OpenACC uses directives that compile as comments on non-GPU compilers:

```fortran
subroutine compute_flux(h, hu, flux, nx, ny)
   real(wp), intent(in) :: h(nx, ny), hu(nx, ny)
   real(wp), intent(out) :: flux(nx, ny)
   integer, intent(in) :: nx, ny

   integer :: i, j

   !$acc parallel loop collapse(2) default(present)
   do j = 1, ny
      do i = 1, nx
         flux(i,j) = hu(i,j)**2 / h(i,j) + 0.5_wp * GRAVITY * h(i,j)**2
      end do
   end do
end subroutine
```

**Key directives:**
- `!$acc parallel loop` — parallelize the following loop
- `!$acc collapse(n)` — collapse n nested loops into one
- `!$acc default(present)` — assume data is already on GPU
- `!$acc data copyin/copy/copyout` — manage data movement

### CUDA Fortran Basics

CUDA Fortran uses explicit kernel definitions:

```fortran
attributes(global) subroutine compute_flux_kernel(h, hu, flux, nx, ny)
   real(wp), device, intent(in) :: h(nx, ny), hu(nx, ny)
   real(wp), device, intent(out) :: flux(nx, ny)
   integer, value, intent(in) :: nx, ny

   integer :: i, j

   i = (blockIdx%x - 1) * blockDim%x + threadIdx%x
   j = (blockIdx%y - 1) * blockDim%y + threadIdx%y

   if (i <= nx .and. j <= ny) then
      flux(i,j) = hu(i,j)**2 / h(i,j) + 0.5_wp * GRAVITY * h(i,j)**2
   end if
end subroutine
```

**Key patterns:**
- `attributes(global)` — kernel callable from host
- `attributes(device)` — device-only subroutine
- `device` attribute — GPU memory
- `value` intent — pass by value (required for scalars in kernels)

### Data Management Strategy

**Solver-level data management:** Copy data to GPU once at start, keep it there, copy back at end:

```fortran
subroutine run_simulation(state, config)
   type(state_t), intent(inout) :: state
   type(config_t), intent(in) :: config

   ! Copy to GPU once
   !$acc enter data copyin(state%h, state%hu, state%hv, state%z)
   !$acc enter data copyin(config)

   do while (state%time < config%t_end)
      call compute_timestep(state, config)    ! All GPU work
      call apply_boundary_conditions(state)
      call update_state(state)
   end do

   ! Copy back once
   !$acc exit data copyout(state%h, state%hu, state%hv)
   !$acc exit data delete(state%z, config)
end subroutine
```

**Unit tests:** Wrap kernel calls with explicit data regions:

```fortran
subroutine test_flux_kernel()
   real(wp) :: h(10,10), hu(10,10), flux(10,10)

   ! Initialize test data
   h = 1.0_wp
   hu = 0.5_wp

   !$acc data copyin(h, hu) copyout(flux)
   call compute_flux(h, hu, flux, 10, 10)
   !$acc end data

   ! Verify results on host
   call assert_equals(flux(5,5), expected_value, tolerance)
end subroutine
```

### Multi-GPU with MPI

Each MPI rank binds to one GPU:

```fortran
subroutine setup_gpu(comm)
   use mpi_f08
   type(MPI_Comm), intent(in) :: comm

   type(MPI_Comm) :: node_comm
   integer :: node_rank, ierr

   ! Get rank within node
   call MPI_Comm_split_type(comm, MPI_COMM_TYPE_SHARED, 0, &
                            MPI_INFO_NULL, node_comm, ierr)
   call MPI_Comm_rank(node_comm, node_rank, ierr)

   ! Bind to GPU
   !$acc set device_num(node_rank)
   ! Or for CUDA Fortran:
   ! ierr = cudaSetDevice(node_rank)
end subroutine
```

### Memory Coalescing

Ensure contiguous memory access for best GPU performance:

```fortran
! Good — contiguous access (column-major in Fortran)
!$acc parallel loop collapse(2)
do j = 1, ny
   do i = 1, nx
      a(i,j) = b(i,j) + c(i,j)
   end do
end do

! Bad — strided access
!$acc parallel loop collapse(2)
do i = 1, nx
   do j = 1, ny
      a(i,j) = b(i,j) + c(i,j)    ! Threads access non-contiguous memory
   end do
end do
```

### Reductions

**OpenACC:**
```fortran
real(wp) :: total
total = 0.0_wp
!$acc parallel loop reduction(+:total)
do i = 1, n
   total = total + values(i)
end do
```

**CUDA Fortran:** Use `atomicAdd` for simple reductions, or implement parallel reduction kernels for performance-critical code:

```fortran
attributes(global) subroutine sum_kernel(values, n, result)
   real(wp), device, intent(in) :: values(n)
   integer, value, intent(in) :: n
   real(wp), device, intent(inout) :: result

   integer :: i
   real(wp) :: local_sum

   local_sum = 0.0_wp
   do i = threadIdx%x, n, blockDim%x * gridDim%x
      local_sum = local_sum + values(i)
   end do

   call atomicAdd(result, local_sum)
end subroutine
```

---

## File Structure Template

```fortran
!! Brief module description (one line)
module module_name
   !! Extended module documentation.
   !!
   !! More details about purpose, usage, and any important notes.

   use iso_fortran_env, only: real64, int32
   use other_module, only: needed_type_t, needed_function
   implicit none
   private

   ! Public API
   public :: my_type_t
   public :: initialize
   public :: compute

   ! Constants
   real(real64), parameter :: SOME_CONSTANT = 1.0e-6_real64

   ! Types
   type :: my_type_t
      !! Type documentation
      integer :: n
         !! Number of elements
      real(real64), allocatable :: data(:)
         !! Data array
   contains
      procedure :: compute => my_type_compute
      procedure :: destroy => my_type_destroy
   end type my_type_t

contains

   subroutine initialize(self, n)
      !! Initialize the type with n elements
      type(my_type_t), intent(out) :: self
      integer, intent(in) :: n

      self%n = n
      allocate(self%data(n))
      self%data = 0.0_real64
   end subroutine initialize

   subroutine my_type_compute(self, input, output)
      !! Compute something
      class(my_type_t), intent(inout) :: self
      real(real64), intent(in) :: input
      real(real64), intent(out) :: output

      ! Implementation
      output = input * sum(self%data)
   end subroutine my_type_compute

   subroutine my_type_destroy(self)
      !! Clean up allocated memory
      class(my_type_t), intent(inout) :: self

      if (allocated(self%data)) deallocate(self%data)
   end subroutine my_type_destroy

end module module_name
```

---

## Summary

| Category | Do | Don't |
|----------|-----|-------|
| **Types** | `state_t`, `config_t` | `State`, `TConfig` |
| **Variables** | `num_cells`, `total_flux` | `nCells`, `tf` |
| **Constants** | `MAX_ITER`, `GRAVITY` | `maxIter`, `g` |
| **Imports** | `use mod, only: x, y` | `use mod` |
| **Arrays** | `arr(:)` (assumed-shape) | `arr(*)` (assumed-size) |
| **Memory** | `allocatable` | `pointer` (unless needed) |
| **Output** | Logging framework | `print *` |
| **Control** | `if/select/do` | `goto` |

---

## Resources

- [Modern Fortran Explained](https://global.oup.com/academic/product/modern-fortran-explained-9780198876571) — Metcalf, Reid, Cohen
- [Fortran Wiki](https://fortranwiki.org/)
- [fortran-lang.org](https://fortran-lang.org/) — Modern Fortran community
- [FORD](https://github.com/Fortran-FOSS-Programmers/ford) — Documentation generator
- [OpenACC](https://www.openacc.org/) — GPU directive programming
- [NVIDIA HPC SDK](https://developer.nvidia.com/hpc-sdk) — CUDA Fortran compiler

---

*This guide is open for community contributions. Feel free to adapt it for your projects.*
