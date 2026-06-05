# Iteration 1

## Revised DCS Model

```fsp
//*****************************************************************************
// Environment
//*****************************************************************************

const N = 2
set VisitorAction = {reqEnter_Hall,reqEnter_A,reqOut}
set LockAction    = {lock_Hall,lock_A,unlock_Hall,unlock_A}

/*
 * Additional controllable permission actions inserted before arrive_A.2
 * and arrive_Hall.2. They synchronize with the controller's lock/unlock
 * decisions and mediate the previously uncontrollable arrivals.
 */
set PermitAction  = {permit_arrive_A2,permit_arrive_Hall2}

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
	allow_Hall ->
		(arrive_Hall[1] -> VISIT
		|permit_arrive_Hall2 -> arrive_Hall[2] -> VISIT)
	|deny_Hall -> notEnter -> VISIT
),
CHECK_A =
(
	allow_A ->
		(arrive_A[1] -> VISIT
		|permit_arrive_A2 -> arrive_A[2] -> VISIT)
	|deny_A ->
		(arrive_Hall[1] -> VISIT
		|permit_arrive_Hall2 -> arrive_Hall[2] -> VISIT)
),
CHECK_OUT  = (leave -> VISIT).


HALL = HALL[0],
HALL[n:0..N] =
(
	when(n!=N) allow_Hall ->
		(when(n+1==1) arrive_Hall[n+1] -> HALL[n+1]
		|when(n+1==2) permit_arrive_Hall2 -> arrive_Hall[n+1] -> HALL[n+1])

	|when(n!=N) deny_A ->
		(when(n+1==1) arrive_Hall[n+1] -> HALL[n+1]
		|when(n+1==2) permit_arrive_Hall2 -> arrive_Hall[n+1] -> HALL[n+1])

	|when(n!=0) reqEnter_A -> HALL[n-1]
).

ROOM_A = ROOM_A[0],
ROOM_A[n:0..N] =
(
	when(n!=N) allow_A ->
		(when(n+1==1) arrive_A[n+1] -> ROOM_A[n+1]
		|when(n+1==2) permit_arrive_A2 -> arrive_A[n+1] -> ROOM_A[n+1])

	|when(n!=0) reqOut -> ROOM_A[n-1]
).


DOOR_HALL = UNLOCKED_HALL,
UNLOCKED_HALL =
(
	allow_Hall -> UNLOCKED_HALL
	|lock_Hall -> PERMIT_HALL_LOCK
	|unlock_Hall -> PERMIT_HALL_UNLOCK
),
LOCKED_HALL =
(
	deny_Hall -> LOCKED_HALL
	|unlock_Hall -> PERMIT_HALL_UNLOCK
	|lock_Hall -> PERMIT_HALL_LOCK
),

/*
 * The permission point synchronizes with the controller's lock/unlock
 * actions before an arrive_Hall.2 event may occur.
 */
PERMIT_HALL_LOCK =
(
	permit_arrive_Hall2 -> LOCKED_HALL
),
PERMIT_HALL_UNLOCK =
(
	permit_arrive_Hall2 -> UNLOCKED_HALL
).


DOOR_A = UNLOCKED_A,
UNLOCKED_A =
(
	allow_A -> UNLOCKED_A
	|lock_A -> PERMIT_A_LOCK
	|unlock_A -> PERMIT_A_UNLOCK
),
LOCKED_A =
(
	deny_A -> LOCKED_A
	|unlock_A -> PERMIT_A_UNLOCK
	|lock_A -> PERMIT_A_LOCK
),

/*
 * The permission point synchronizes with the controller's lock/unlock
 * actions before an arrive_A.2 event may occur.
 */
PERMIT_A_LOCK =
(
	permit_arrive_A2 -> LOCKED_A
),
PERMIT_A_UNLOCK =
(
	permit_arrive_A2 -> UNLOCKED_A
).


||Environment =
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
					LIMIT_A,
				   }
	controllable = {LockAction, PermitAction}
}

//*****************************************************************************
//Controller
//*****************************************************************************

controller ||Controller  = (Environment)~{Specification}.
```

### Explanation of the changes

The Additional Control Record consistently identifies the problematic uncontrollable events as:

* `arrive_A.2`
* `arrive_Hall.2`

and indicates that they should be mediated by the existing controllable lock/unlock decisions (`lock_Hall`, `unlock_Hall`, `lock_A`, `unlock_A`).

To realize this:

1. Added two new controllable permission actions:

   * `permit_arrive_A2`
   * `permit_arrive_Hall2`

2. Inserted these actions immediately before the corresponding second-arrival events:

   * `permit_arrive_A2 -> arrive_A[2]`
   * `permit_arrive_Hall2 -> arrive_Hall[2]`

3. Modified `VISITOR`, `HALL`, and `ROOM_A` so that only the occupancy-increasing transitions to state `2` require the new permission action. Arrivals to state `1` remain unchanged.

4. Added synchronization points in `DOOR_HALL` and `DOOR_A` that connect the new permission actions to the existing controllable lock/unlock actions, thereby realizing the guidance that the new control point should mediate synchronization with:

   * `lock_Hall` / `unlock_Hall`
   * `lock_A` / `unlock_A`

5. Kept all original safety properties unchanged and preserved the original model structure as much as possible. The repair only introduces controllable permission points before the identified uncontrollable arrivals.

## Iteration 2

## Revised DCS Model

```fsp
//*****************************************************************************
// Environment
//*****************************************************************************

const N = 2

set VisitorAction = {reqEnter_Hall,reqEnter_A,reqOut}
set LockAction    = {lock_Hall,lock_A,unlock_Hall,unlock_A}

/*
 * Additional controllable actions derived from the Additional Control Record.
 * They act as permission/synchronization points before the identified
 * uncontrollable actions.
 */
set PermitAction = {
    permit_reqEnter_Hall,
    permit_reqEnter_A,
    permit_reqOut,
    permit_allow_Hall,
    permit_allow_A,
    permit_arrive_Hall2,
    permit_arrive_A2
}

set AllowAction   = {allow_Hall,allow_A}
set DenyAction    = {deny_Hall,deny_A}
set DoorAction    = {LockAction,AllowAction,DenyAction,resRoomStatus}
set MonitorAction = {notEnter,arrive_Hall[1..N],arrive_A[1..N],leave}


//-----------------------------------------------------------------------------
// Visitor
//-----------------------------------------------------------------------------

VISITOR = VISIT,

VISIT =
(
      permit_reqEnter_Hall -> reqEnter_Hall -> CHECK_HALL
    | permit_reqEnter_A    -> reqEnter_A    -> CHECK_A
    | permit_reqOut        -> reqOut        -> CHECK_OUT
),

CHECK_HALL =
(
      permit_allow_Hall -> allow_Hall ->
            (
                arrive_Hall[1] -> VISIT
              | permit_arrive_Hall2 -> arrive_Hall[2] -> VISIT
            )

    | deny_Hall -> notEnter -> VISIT
),

CHECK_A =
(
      permit_allow_A -> allow_A ->
            (
                arrive_A[1] -> VISIT
              | permit_arrive_A2 -> arrive_A[2] -> VISIT
            )

    | deny_A ->
            (
                arrive_Hall[1] -> VISIT
              | permit_arrive_Hall2 -> arrive_Hall[2] -> VISIT
            )
),

CHECK_OUT =
(
    leave -> VISIT
).


//-----------------------------------------------------------------------------
// Occupancy models
//-----------------------------------------------------------------------------

HALL = HALL[0],

HALL[n:0..N] =
(
      when(n!=N)
      permit_allow_Hall -> allow_Hall ->
            (
                when(n+1==1)
                    arrive_Hall[n+1] -> HALL[n+1]

              | when(n+1==2)
                    permit_arrive_Hall2 ->
                    arrive_Hall[n+1] ->
                    HALL[n+1]
            )

    | when(n!=N)
      deny_A ->
            (
                when(n+1==1)
                    arrive_Hall[n+1] -> HALL[n+1]

              | when(n+1==2)
                    permit_arrive_Hall2 ->
                    arrive_Hall[n+1] ->
                    HALL[n+1]
            )

    | when(n!=0)
      reqEnter_A -> HALL[n-1]
).


ROOM_A = ROOM_A[0],

ROOM_A[n:0..N] =
(
      when(n!=N)
      permit_allow_A -> allow_A ->
            (
                when(n+1==1)
                    arrive_A[n+1] -> ROOM_A[n+1]

              | when(n+1==2)
                    permit_arrive_A2 ->
                    arrive_A[n+1] ->
                    ROOM_A[n+1]
            )

    | when(n!=0)
      reqOut -> ROOM_A[n-1]
).


//-----------------------------------------------------------------------------
// Door Hall
//
// Additional Control Record indicates that reqEnter_Hall, reqEnter_A,
// reqOut, allow_Hall and arrive_Hall.2 should synchronize with Hall lock
// decisions.
//-----------------------------------------------------------------------------

DOOR_HALL = HALL_UNLOCKED,

HALL_UNLOCKED =
(
      permit_reqEnter_Hall -> HALL_UNLOCKED
    | permit_reqEnter_A    -> HALL_UNLOCKED
    | permit_reqOut        -> HALL_UNLOCKED
    | permit_allow_Hall    -> HALL_UNLOCKED
    | permit_arrive_Hall2  -> HALL_UNLOCKED

    | allow_Hall           -> HALL_UNLOCKED
    | lock_Hall            -> HALL_LOCKED
),

HALL_LOCKED =
(
      permit_reqEnter_Hall -> HALL_LOCKED
    | permit_reqEnter_A    -> HALL_LOCKED
    | permit_reqOut        -> HALL_LOCKED
    | permit_allow_Hall    -> HALL_LOCKED
    | permit_arrive_Hall2  -> HALL_LOCKED

    | deny_Hall            -> HALL_LOCKED
    | unlock_Hall          -> HALL_UNLOCKED
).


//-----------------------------------------------------------------------------
// Door A
//
// Additional Control Record indicates that allow_A and arrive_A.2 should
// synchronize with A lock decisions. Some states involve both Hall and A
// locks; using shared permit actions allows synchronization with either or
// both door processes through parallel composition.
//-----------------------------------------------------------------------------

DOOR_A = A_UNLOCKED,

A_UNLOCKED =
(
      permit_allow_A      -> A_UNLOCKED
    | permit_arrive_A2    -> A_UNLOCKED

    | allow_A             -> A_UNLOCKED
    | lock_A              -> A_LOCKED
),

A_LOCKED =
(
      permit_allow_A      -> A_LOCKED
    | permit_arrive_A2    -> A_LOCKED

    | deny_A             -> A_LOCKED
    | unlock_A           -> A_UNLOCKED
).


//-----------------------------------------------------------------------------
// System
//-----------------------------------------------------------------------------

||Environment =
(
      VISITOR
   || HALL
   || ROOM_A
   || DOOR_HALL
   || DOOR_A
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
    safety =
    {
        LIMIT_HALL,
        LIMIT_A
    }

    controllable = {LockAction,PermitAction}
}


//*****************************************************************************
// Controller
//*****************************************************************************

controller ||Controller = (Environment)~{Specification}.
```

### Explanation of the repair

The Additional Control Record identifies four classes of uncontrollable actions that need controllable mediation:

1. `arrive_Hall.2`
2. `arrive_A.2`
3. `allow_Hall`
4. `allow_A`
5. Visitor requests (`reqEnter_Hall`, `reqEnter_A`, `reqOut`)

Accordingly, the repair introduces the following new controllable permission actions:

* `permit_arrive_Hall2`
* `permit_arrive_A2`
* `permit_allow_Hall`
* `permit_allow_A`
* `permit_reqEnter_Hall`
* `permit_reqEnter_A`
* `permit_reqOut`

Each identified uncontrollable action is now preceded by its corresponding controllable permission action while preserving the original action itself and the original branching structure.

The new permission actions are synchronized through `DOOR_HALL` and/or `DOOR_A`, reflecting the Additional Control Record's indication that these events should be mediated by the existing lock/unlock decisions. The safety properties `LIMIT_HALL` and `LIMIT_A` are unchanged.
