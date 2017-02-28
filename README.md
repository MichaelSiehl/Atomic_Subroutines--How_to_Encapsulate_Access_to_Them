# Atomic_Subroutines--How_to_Encapsulate_Access_to_Them
Fortran 2008 coarray programming with unordered execution segments (user-defined ordering) - Atomic Subroutines: Encapsulate access to atomic_define and atomic_ref

# Overview
This GitHub repository aims to show how to encapsulate access to the both Fortran 2008 atomic subroutines atomic_define and atomic_ref and thus, how to keep the parallel logic codes and syntax apart from the application's main logic codes.
<br />
Assume the following type definition and coarray declaration in a module:<br />
!  Type Definition:<br />
type, public :: ImageStatus_CA<br />
&nbsp;&nbsp;private<br />
&nbsp;&nbsp;integer(atomic_int_kind) :: m_atomic_intImageActivityFlag = 0<br />
end type OOOPimsc_adtImageStatus_CA<br />
!<br />
! Coarray Declaration:<br />
type (ImageStatus_CA), public, codimension[*], save :: ImageStatus_CA_1<br />
