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

# How to Encapsulate Access to atomic_ref:
The first reflex to encapsulate access to atomic_ref may be to use a traditional style getter routine to do so. But the problem with atomic_ref is it's use in conjunction with a spin-wait loop synchronization and the required SYNC MEMORY statement after the atomic data transfer took place. Since we want to keep the call to atomic_ref apart form the spin-wait loop (also because of the spin-wait loop may not be part of the parallel logic code), and also want to keep the SYNC MEMORY statement at a place of it's own in the parallel logic code, we use a checker routine instead of a getter routine. See the following code example:

logical function Check_atomic_intImageActivityFlag_CA (Object_CA, intCheckImageActivityFlag)<br />
&nbsp;&nbsp;! in order to hide the sync memory statement herein, this Checker routine does not allow<br />
&nbsp;&nbsp;! to access the member directly, but instead does only allow to check the atomic member<br />
&nbsp;&nbsp;! for specific values (this Checker is intentioned for synchronizations)<br />
&nbsp;&nbsp;type (ImageStatus_CA), codimension[*], intent (inout) :: Object_CA<br />
&nbsp;&nbsp;integer, intent (in) :: intCheckImageActivityFlag<br />
&nbsp;&nbsp;integer :: intImageActivityFlag<br />
&nbsp;&nbsp;!<br />
&nbsp;&nbsp;Check_atomic_intImageActivityFlag_CA = .false.<br />
&nbsp;&nbsp;!<br />
&nbsp;&nbsp;call atomic_ref (intImageActivityFlag, Object_CA % m_atomic_intImageActivityFlag)<br />
&nbsp;&nbsp;if (intCheckImageActivityFlag == intImageActivityFlag) then<br />
&nbsp;&nbsp;&nbsp;&nbsp;call subSyncMemory (Object_CA) ! executing sync memory<br />
&nbsp;&nbsp;&nbsp;&nbsp;Check_atomic_intImageActivityFlag_CA = .true.<br />
&nbsp;&nbsp;end if<br />
&nbsp;&nbsp;!<br />
end function Check_atomic_intImageActivityFlag_CA<br />

Then, to make use of this Checker routine, we could use a spin-wait loop from the main logic code (well, the following code snippet is rather part of the parallel logic code), something like this (the code uses an enumeration):<br />

...<br />
do ! check the ImageActivityFlag in local PGAS memory until it has<br />
&nbsp;&nbsp;&nbsp;&nbsp;! value Enum_ImageActivityFlag % ExecutionFinished<br />
&nbsp;&nbsp;if (Check_atomic_intImageActivityFlag_CA (ImageStatus_CA_1, &<br />
&nbsp;&nbsp;&nbsp;&nbsp;Enum_ImageActivityFlag % InitializeSegmentSynchronization)) then<br />
&nbsp;&nbsp;&nbsp;&nbsp;! initialize the execution segment synchronization on the executing image<br />
&nbsp;&nbsp;&nbsp;&nbsp;…<br />
&nbsp;&nbsp;end if<br />
&nbsp;&nbsp;!<br />
&nbsp;&nbsp;if (Check_atomic_intImageActivityFlag_CA (ImageStatus_CA_1, &<br />
&nbsp;&nbsp;&nbsp;&nbsp;Enum_ImageActivityFlag % ExecutionFinsihed)) then<br />
&nbsp;&nbsp;&nbsp;&nbsp;! exit the loop<br />
&nbsp;&nbsp;&nbsp;&nbsp;…<br />
&nbsp;&nbsp;end if<br />
&nbsp;&nbsp;!<br />
end do<br />
...<br />

