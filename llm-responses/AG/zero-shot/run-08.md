## Revised DCS Model

```fsp
//*****************************************************************************
// Environment
//*****************************************************************************

const N = 2
set VisitorAction = {reqEnter_Hall,reqEnter_A,reqOut}
set LockAction    = {lock_Hall,lock_A,unlock_Hall,unlock_A}
set AllowAction   = {allow_Hall,allow_A}
set DenyAction    = {deny_Hall,deny_A}
set GrantAction   = {grant_Hall,grant_A}
set DoorAction    = {LockAction,AllowAction,DenyAction,GrantAction,resRoomStatus}
set MonitorAction = {notEnter,arrive_Hall[1..N],arrive_A[1..N],leave}


//-----------------------------------------------------------------------------
// Visitor
//-----------------------------------------------------------------------------

VISITOR = VISIT,
VISIT =
(
	reqEnter_Hall  -> CHECK_HALL
    |reqEnter_A    -> CHECK_A
    |reqOut        -> CHECK_OUT
),

CHECK_HALL =
(
	allow_Hall -> grant_Hall -> arrive_Hall[1..N] -> VISIT
	|deny_Hall -> notEnter -> VISIT
),

CHECK_A =
(
	allow_A -> grant_A -> arrive_A[1..N] -> VISIT
	|deny_A -> arrive_Hall[1..N] -> VISIT
),

CHECK_OUT =
(
	leave -> VISIT
).


//-----------------------------------------------------------------------------
// Capacity models
//-----------------------------------------------------------------------------

HALL = HALL[0],

HALL[n:0..N] =
(
	when(n!=N)
		grant_Hall -> arrive_Hall[n+1] -> HALL[n+1]

	|when(n!=N)
		deny_A -> arrive_Hall[n+1] -> HALL[n+1]

	|when(n!=0)
		reqEnter_A -> HALL[n-1]
).

ROOM_A = ROOM_A[0],

ROOM_A[n:0..N] =
(
	when(n!=N)
		grant_A -> arrive_A[n+1] -> ROOM_A[n+1]

	|when(n!=0)
		reqOut -> ROOM_A[n-1]
).


//-----------------------------------------------------------------------------
// Door protocols
//-----------------------------------------------------------------------------

DOOR_HALL = UNLOCKED_HALL,

UNLOCKED_HALL =
(
	allow_Hall -> grant_Hall -> UNLOCKED_HALL
	|lock_Hall -> LOCKED_HALL
),

LOCKED_HALL =
(
	deny_Hall  -> LOCKED_HALL
	|unlock_Hall -> UNLOCKED_HALL
).


DOOR_A = UNLOCKED_A,

UNLOCKED_A =
(
	allow_A -> grant_A -> UNLOCKED_A
	|lock_A -> LOCKED_A
),

LOCKED_A =
(
	deny_A  -> LOCKED_A
	|unlock_A -> UNLOCKED_A
).


//-----------------------------------------------------------------------------
// System composition
//-----------------------------------------------------------------------------

||Environment  =
(
	VISITOR
	||HALL
	||ROOM_A
	||DOOR_HALL
	||DOOR_A
).


//*****************************************************************************
// Requirement
//*****************************************************************************

const TH_HALL = 2
const TH_A    = 2

ltl_property LIMIT_HALL = [](!arrive_Hall[TH_HALL])
ltl_property LIMIT_A    = [](!arrive_A[TH_A])

controllerSpec Specification =
{
	safety       = {
					LIMIT_HALL,
					LIMIT_A
				   }

	controllable = {
					LockAction,
					GrantAction
				   }
}


//*****************************************************************************
// Controller
//*****************************************************************************

controller ||Controller  = (Environment)~{Specification}.
```

## Explanation of the Changes

The original model is unrealizable because the uncontrollable `allow_*` actions immediately force the corresponding `arrive_*` events. Once `allow_Hall` or `allow_A` occurs, the controller has no opportunity to prevent a violating arrival, even if the room is already at the safety threshold.

To recover synthesizability, a permission/confirmation synchronization layer was added:

* New controllable actions:

  * `grant_Hall`
  * `grant_A`

* Revised flow:

  * `allow_* -> grant_* -> arrive_*`

* The occupancy models (`HALL`, `ROOM_A`) now synchronize on `grant_*` before permitting the arrival transition.

* The door processes also synchronize on the same `grant_*` events, ensuring that arrivals only happen after explicit controller authorization.

This preserves:

* all original safety properties,
* all original process names and action structure,
* the distinction between `allow` and `deny`,
* the original occupancy accounting logic.

The controller can now block `grant_Hall` or `grant_A` when an arrival would violate `LIMIT_HALL` or `LIMIT_A`, making the specification synthesizable again.
