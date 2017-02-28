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

# How to Encapsulate Access to atomic_define:
Encapsulating access to atomic_define is very straightforward: We can easily use a traditional style setter routine to do so. See the following code example:<br />

subroutine Set_atomic_intImageActivityFlag_CA (Object_CA, intImageActivityFlag, intImageNumber)<br />
&nbsp;&nbsp;type (ImageStatus_CA), codimension[*], intent (inout) :: Object_CA<br />
&nbsp;&nbsp;integer, intent (in) :: intImageActivityFlag<br />
&nbsp;&nbsp;integer, intent (in) :: intImageNumber<br />
&nbsp;&nbsp;!<br />
&nbsp;&nbsp;call atomic_define (Object_CA[intImageNumber] % m_atomic_intImageActivityFlag, intImageActivityFlag)<br />
&nbsp;&nbsp;!<br />
end subroutine Set_atomic_intImageActivityFlag_CA<br />
