## Revised DCS Model

```fsp
//*****************************************************************************
// Environment
//*****************************************************************************

const N = 2
set VisitorAction = {reqEnter_Hall,reqEnter_A,reqOut}
set LockAction    = {lock_Hall,lock_A,unlock_Hall,unlock_A}

// New controllable permission actions inserted before uncontrollable arrivals
set PreArriveAction = {permit_Hall, permit_A}

set AllowAction   = {allow_Hall,allow_A}
set DenyAction    = {deny_Hall,deny_A}
set DoorAction    = {LockAction,AllowAction,DenyAction,resRoomStatus}
set MonitorAction = {notEnter,arrive_Hall[1..N],arrive_A[1..N],leave}


VISITOR = VISIT,
VISIT =
(
	reqEnter_Hall  -> CHECK_HALL
    |reqEnter_A    -> CHECK_A
    |reqOut        -> CHECK_OUT
),
CHECK_HALL =
(
	allow_Hall -> permit_Hall -> arrive_Hall[1..N] -> VISIT
  | deny_Hall  -> notEnter -> VISIT
),
CHECK_A =
(
	allow_A -> permit_A -> arrive_A[1..N] -> VISIT
  | deny_A  -> permit_Hall -> arrive_Hall[1..N] -> VISIT
),
CHECK_OUT  = (leave -> VISIT).


HALL = HALL[0],
HALL[n:0..N] =
(
	when(n!=N) allow_Hall -> permit_Hall -> arrive_Hall[n+1] -> HALL[n+1]
  | when(n!=N) deny_A    -> permit_Hall -> arrive_Hall[n+1] -> HALL[n+1]
  | when(n!=0) reqEnter_A -> HALL[n-1]
).

ROOM_A = ROOM_A[0],
ROOM_A[n:0..N] =
(
	when(n!=N) allow_A -> permit_A -> arrive_A[n+1] -> ROOM_A[n+1]
  | when(n!=0) reqOut -> ROOM_A[n-1]
).


DOOR_HALL = UNLOCKED_H,
UNLOCKED_H =
(
	allow_Hall -> UNLOCKED_H
  | lock_Hall -> LOCKED_H
),
LOCKED_H =
(
	deny_Hall -> LOCKED_H
  | unlock_Hall -> UNLOCKED_H
).

DOOR_A = UNLOCKED_A,
UNLOCKED_A =
(
	allow_A -> UNLOCKED_A
  | lock_A -> LOCKED_A
),
LOCKED_A =
(
	deny_A -> LOCKED_A
  | unlock_A -> UNLOCKED_A
).


||Environment  = (VISITOR||HALL||ROOM_A||DOOR_HALL||DOOR_A).

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
					LIMIT_A,
				   }
	controllable = {LockAction, PreArriveAction}
}

//*****************************************************************************
// Controller
//*****************************************************************************

controller ||Controller  = (Environment)~{Specification}.
```

### Explanation of changes

* Added two new **controllable** actions:

  * `permit_A` before every `arrive_A[i]`
  * `permit_Hall` before every `arrive_Hall[i]`

* These act as explicit **permission/synchronization points** before the previously uncontrollable arrival events, exactly matching the Additional Control Record’s recommendation to insert extra control before `arrive_A.2` and `arrive_Hall.2`.

* The original uncontrollable actions (`arrive_A`, `arrive_Hall`) were preserved unchanged; only a new controllable step was inserted immediately before them.

* Safety properties (`LIMIT_HALL`, `LIMIT_A`) were left unchanged.

* Added `PreArriveAction` to the controller’s controllable set so synthesis can use these new decision points.
