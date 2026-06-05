## Revised DCS Model

```fsp
//*****************************************************************************
// Environment
//*****************************************************************************

const N = 2
set VisitorAction = {reqEnter_Hall,reqEnter_A,reqOut}
set LockAction    = {lock_Hall,lock_A,unlock_Hall,unlock_A}

/* New controllable permission actions inserted before uncontrollable arrivals */
set PreArriveAction = {preArrive_Hall, preArrive_A}

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
/* additional controllable synchronization before arrive_Hall */
CHECK_HALL = (
      allow_Hall -> preArrive_Hall -> arrive_Hall[1..N] -> VISIT
    | deny_Hall  -> notEnter -> VISIT
),
/* additional controllable synchronization before arrive_A */
CHECK_A    = (
      allow_A -> preArrive_A -> arrive_A[1..N] -> VISIT
    | deny_A  -> preArrive_Hall -> arrive_Hall[1..N] -> VISIT
),
CHECK_OUT  = (leave -> VISIT).


HALL = HALL[0],
HALL[n:0..N] = (
      when(n!=N) allow_Hall -> preArrive_Hall -> arrive_Hall[n+1] -> HALL[n+1]
    | when(n!=N) deny_A     -> preArrive_Hall -> arrive_Hall[n+1] -> HALL[n+1]
    | when(n!=0) reqEnter_A -> HALL[n-1]
).

ROOM_A = ROOM_A[0],
ROOM_A[n:0..N] = (
      when(n!=N) allow_A -> preArrive_A -> arrive_A[n+1] -> ROOM_A[n+1]
    | when(n!=0) reqOut  -> ROOM_A[n-1]
).


DOOR_HALL = UNLOCKED_H,
UNLOCKED_H = (
      allow_Hall -> UNLOCKED_H
    | lock_Hall  -> LOCKED_H
),
LOCKED_H = (
      deny_Hall   -> LOCKED_H
    | unlock_Hall -> UNLOCKED_H
).

DOOR_A = UNLOCKED_A,
UNLOCKED_A = (
      allow_A -> UNLOCKED_A
    | lock_A  -> LOCKED_A
),
LOCKED_A = (
      deny_A   -> LOCKED_A
    | unlock_A -> UNLOCKED_A
).


||Environment = (VISITOR || HALL || ROOM_A || DOOR_HALL || DOOR_A).

//*****************************************************************************
// Requirement
//*****************************************************************************
const TH_HALL = 2
const TH_A    = 2

ltl_property LIMIT_HALL = [](!arrive_Hall[TH_HALL])
ltl_property LIMIT_A    = [](!arrive_A[TH_A])

controllerSpec Specification =
{
	safety = {
		LIMIT_HALL,
		LIMIT_A
	}
	controllable = {LockAction, PreArriveAction}
}

//*****************************************************************************
// Controller
//*****************************************************************************

controller ||Controller = (Environment)~{Specification}.
```

### Explanation of changes

1. Added two **new controllable actions**:

   * `preArrive_A` before every `arrive_A[...]`
   * `preArrive_Hall` before every `arrive_Hall[...]`

2. These act as **permission/synchronization points**, exactly matching the Additional Control Record’s intent: the controller can now decide whether to permit the uncontrollable arrival after observing the lock configuration (`lock_*`, `unlock_*`).

3. Preserved:

   * all original uncontrollable actions (`arrive_A`, `arrive_Hall`)
   * original process names and branching
   * original safety properties unchanged

4. Extended `controllable` set to include `PreArriveAction`, allowing DCS to synthesize decisions at the newly inserted control points.
