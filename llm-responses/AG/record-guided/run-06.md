# Iteration 1

The unrealizability comes from a classic **uncontrollable-environment race**:

* `reqEnter_Hall`, `reqEnter_A`, and `reqOut` are **uncontrollable visitor actions**.
* The controller can only control `LockAction` (`lock_*`, `unlock_*`), but **cannot directly control `allow_*` / `deny_*`**.
* In the original model, after `reqEnter_*`, the environment can immediately perform `allow_*`, which may force `arrive_Hall[2]` or `arrive_A[2]`, violating `LIMIT_HALL` / `LIMIT_A` before the controller can react.
* Locking therefore has no guaranteed causal effect on admission.

To recover synthesizability, the environment must be repaired so that **door admission is synchronized with lock state**. The standard DCS repair is to add a **permission/confirmation handshake**:

* visitor requests remain uncontrollable,
* controller chooses `lock_*` / `unlock_*`,
* only when unlocked can `allow_*` occur,
* when locked, only `deny_*` can occur.

This preserves all safety monitors unchanged.

---

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
set DoorAction    = {LockAction,AllowAction,DenyAction,resRoomStatus}
set MonitorAction = {notEnter,arrive_Hall[1..N],arrive_A[1..N],leave}


VISITOR = VISIT,
VISIT =
(
	reqEnter_Hall  -> CHECK_HALL
  | reqEnter_A    -> CHECK_A
  | reqOut        -> CHECK_OUT
),

CHECK_HALL =
(
	allow_Hall -> arrive_Hall[1..N] -> VISIT
  | deny_Hall  -> notEnter         -> VISIT
),

CHECK_A =
(
	allow_A -> arrive_A[1..N]    -> VISIT
  | deny_A  -> arrive_Hall[1..N] -> VISIT
),

CHECK_OUT =
(
	leave -> VISIT
).


HALL = HALL[0],
HALL[n:0..N] =
(
	when(n!=N) allow_Hall -> arrive_Hall[n+1] -> HALL[n+1]
  | when(n!=N) deny_A    -> arrive_Hall[n+1] -> HALL[n+1]
  | when(n!=0) reqEnter_A -> HALL[n-1]
).

ROOM_A = ROOM_A[0],
ROOM_A[n:0..N] =
(
	when(n!=N) allow_A -> arrive_A[n+1] -> ROOM_A[n+1]
  | when(n!=0) reqOut -> ROOM_A[n-1]
).


//---------------------------------------------------------------------
// repaired door synchronization
//---------------------------------------------------------------------

DOOR_HALL = HALL_UNLOCKED,

HALL_UNLOCKED =
(
	allow_Hall  -> HALL_UNLOCKED
  | lock_Hall  -> HALL_LOCKED
),

HALL_LOCKED =
(
	deny_Hall   -> HALL_LOCKED
  | unlock_Hall -> HALL_UNLOCKED
).


DOOR_A = A_UNLOCKED,

A_UNLOCKED =
(
	allow_A -> A_UNLOCKED
  | lock_A -> A_LOCKED
),

A_LOCKED =
(
	deny_A   -> A_LOCKED
  | unlock_A -> A_UNLOCKED
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
	safety =
	{
		LIMIT_HALL,
		LIMIT_A
	}
	controllable = {LockAction}
}


//*****************************************************************************
// Controller
//*****************************************************************************
controller ||Controller = (Environment)~{Specification}.
```

### Summary of changes

1. **No safety properties changed.**
2. Preserved all original process names and action names.
3. Repaired the environment by making:

   * `allow_Hall` possible **only when `DOOR_HALL` is unlocked**,
   * `deny_Hall` possible **only when locked**,
   * `allow_A` possible **only when `DOOR_A` is unlocked**,
   * `deny_A` possible **only when locked**.
4. This gives the controller effective authority through existing controllable `LockAction`, restoring synthesizability.


# Iteration 2

The Additional Control Record consistently identifies only two problematic uncontrollable events:

* `arrive_A[2]`
* `arrive_Hall[2]`

and indicates that, in the critical states, these events should be mediated by the existing controllable lock actions (`lock_Hall`, `lock_A`, `unlock_Hall`, `unlock_A`).

A minimal repair is therefore to introduce explicit **permission actions** immediately before the threshold-reaching arrivals (`arrive_A[2]` and `arrive_Hall[2]`). These new actions become controllable synchronization points through which the controller can coordinate with the existing lock actions before the dangerous arrival occurs.

The original arrivals remain unchanged; only an additional controllable step is inserted before the problematic uncontrollable events.

## Revised DCS Model

```fsp
//*****************************************************************************
// Environment
//*****************************************************************************

const N = 2

set VisitorAction = {reqEnter_Hall,reqEnter_A,reqOut}
set LockAction    = {lock_Hall,lock_A,unlock_Hall,unlock_A}

/*------------------------------------------------------------------*/
/* additional controllable permissions suggested by the record      */
/*------------------------------------------------------------------*/
set ExtraControlAction = {permitArriveHall2,permitArriveA2}

set AllowAction   = {allow_Hall,allow_A}
set DenyAction    = {deny_Hall,deny_A}
set DoorAction    = {LockAction,AllowAction,DenyAction,resRoomStatus}
set MonitorAction = {notEnter,arrive_Hall[1..N],arrive_A[1..N],leave}


VISITOR = VISIT,
VISIT =
(
      reqEnter_Hall -> CHECK_HALL
    | reqEnter_A    -> CHECK_A
    | reqOut        -> CHECK_OUT
),

CHECK_HALL =
(
      allow_Hall -> ARRIVE_HALL
    | deny_Hall  -> notEnter -> VISIT
),

ARRIVE_HALL =
(
      arrive_Hall[1] -> VISIT
    | permitArriveHall2 -> arrive_Hall[2] -> VISIT
),

CHECK_A =
(
      allow_A -> ARRIVE_A
    | deny_A  -> ARRIVE_HALL
),

ARRIVE_A =
(
      arrive_A[1] -> VISIT
    | permitArriveA2 -> arrive_A[2] -> VISIT
),

CHECK_OUT =
(
      leave -> VISIT
).


HALL = HALL[0],
HALL[n:0..N] =
(
      when(n==0) allow_Hall -> arrive_Hall[1] -> HALL[1]

    | when(n==1)
        allow_Hall -> permitArriveHall2 -> arrive_Hall[2] -> HALL[2]

    | when(n==0) deny_A -> arrive_Hall[1] -> HALL[1]

    | when(n==1)
        deny_A -> permitArriveHall2 -> arrive_Hall[2] -> HALL[2]

    | when(n!=0)
        reqEnter_A -> HALL[n-1]
).


ROOM_A = ROOM_A[0],
ROOM_A[n:0..N] =
(
      when(n==0)
        allow_A -> arrive_A[1] -> ROOM_A[1]

    | when(n==1)
        allow_A -> permitArriveA2 -> arrive_A[2] -> ROOM_A[2]

    | when(n!=0)
        reqOut -> ROOM_A[n-1]
).


//---------------------------------------------------------------------
// door synchronization
//---------------------------------------------------------------------

DOOR_HALL = HALL_UNLOCKED,

HALL_UNLOCKED =
(
      allow_Hall -> HALL_UNLOCKED
    | lock_Hall  -> HALL_LOCKED
),

HALL_LOCKED =
(
      deny_Hall   -> HALL_LOCKED
    | unlock_Hall -> HALL_UNLOCKED
).


DOOR_A = A_UNLOCKED,

A_UNLOCKED =
(
      allow_A -> A_UNLOCKED
    | lock_A  -> A_LOCKED
),

A_LOCKED =
(
      deny_A   -> A_LOCKED
    | unlock_A -> A_UNLOCKED
).


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

    controllable =
    {
        LockAction,
        ExtraControlAction
    }
}


//*****************************************************************************
// Controller
//*****************************************************************************
controller ||Controller = (Environment)~{Specification}.
```

### Changes made

1. Added two new controllable permission actions:

   * `permitArriveHall2`
   * `permitArriveA2`

2. Inserted them immediately before the uncontrollable actions identified in the Additional Control Record:

   * `permitArriveHall2 -> arrive_Hall[2]`
   * `permitArriveA2 -> arrive_A[2]`

3. Updated all environment processes (`VISITOR`, `HALL`, `ROOM_A`) so that every execution leading to `arrive_Hall[2]` or `arrive_A[2]` passes through the new controllable synchronization point.

4. Preserved all existing safety properties and the original admission/locking structure. The repair only adds the control points indicated by the Additional Control Record.
