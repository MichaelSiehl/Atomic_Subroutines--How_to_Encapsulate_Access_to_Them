# Atomic Subroutines: How to Encapsulate Access to Them

Fortran 2008 coarray programming with unordered execution segments (user-defined ordering) - Atomic Subroutines: Encapsulate access to atomic_define and atomic_ref

# Overview

This GitHub repository aims to show how to encapsulate access to the both Fortran 2008 atomic subroutines atomic_define and atomic_ref and thus, how to keep the parallel logic codes and syntax apart from the application's main logic codes.

Assume the following type definition and coarray declaration in a module:

```fortran
type, public :: ImageStatus_CA
  private
  integer(atomic_int_kind) :: m_atomic_intImageActivityFlag = 0
end type OOOPimsc_adtImageStatus_CA

! Coarray Declaration:
type (ImageStatus_CA), public, codimension[*], save :: ImageStatus_CA_1
```

# How to Encapsulate Access to atomic_define

Encapsulating access to atomic_define is very straightforward: We can easily use a traditional style setter routine to do so. See the following code example:

```fortran
subroutine Set_atomic_intImageActivityFlag_CA (Object_CA, intImageActivityFlag, intImageNumber)
  type (ImageStatus_CA), codimension[*], intent (inout) :: Object_CA
  integer, intent (in)                                  :: intImageActivityFlag
  integer, intent (in)                                  :: intImageNumber

  call atomic_define (Object_CA[intImageNumber] % m_atomic_intImageActivityFlag, intImageActivityFlag)

end subroutine Set_atomic_intImageActivityFlag_CA
```

# How to Encapsulate Access to atomic_ref

The first reflex to encapsulate access to atomic_ref may be to use a traditional style getter routine to do so. But the problem with atomic_ref is it's use in conjunction with a spin-wait loop synchronization and the required SYNC MEMORY statement after the atomic data transfer took place. Since we want to keep the call to atomic_ref apart form the spin-wait loop (also because of the spin-wait loop may not be part of the parallel logic code), and also want to keep the SYNC MEMORY statement at a place of it's own in the parallel logic code, we use a checker routine instead of a getter routine. See the following code example:

```fortran
logical function Check_atomic_intImageActivityFlag_CA (Object_CA, intCheckImageActivityFlag)
  ! in order to hide the sync memory statement herein, this Checker routine does not allow
  ! to access the member directly, but instead does only allow to check the atomic member
  ! for specific values (this Checker is intentioned for synchronizations)
  type (ImageStatus_CA), codimension[*], intent (inout) :: Object_CA
  integer, intent (in)                                  :: intCheckImageActivityFlag
  integer                                               :: intImageActivityFlag

  Check_atomic_intImageActivityFlag_CA = .false.

  call atomic_ref (intImageActivityFlag, Object_CA % m_atomic_intImageActivityFlag)

  if (intCheckImageActivityFlag == intImageActivityFlag) then
    call subSyncMemory (Object_CA) ! executing sync memory
    Check_atomic_intImageActivityFlag_CA = .true.
  end if

end function Check_atomic_intImageActivityFlag_CA
```

Then, to make use of this Checker routine, we could use a spin-wait loop from the main logic code (well, the following code snippet is rather part of the parallel logic code), something like this (the code uses an enumeration):

```fortran
do ! check the ImageActivityFlag in local PGAS memory until it has
  ! value Enum_ImageActivityFlag % ExecutionFinished
  if (Check_atomic_intImageActivityFlag_CA (ImageStatus_CA_1, &
    Enum_ImageActivityFlag % InitializeSegmentSynchronization)) then
    ! initialize the execution segment synchronization on the executing image
    …
    ...
  end if

  if (Check_atomic_intImageActivityFlag_CA (ImageStatus_CA_1, &
    Enum_ImageActivityFlag % ExecutionFinsihed)) then
    ! exit the loop
    ...
  end if

end do
...
```

That way, the logic codes (parallel or main logic codes) remain without any direct call to atomic_ref and without any direct use of the SYNC MEMORY statement.

(As an aside: In practice, spin-wait loop synchronizations must be implemented much more sophisticated then shown here).
