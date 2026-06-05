# Iteration 1

The unrealizability comes from a classic **environment over-permissiveness** problem:

* `reqEnter_Hall` and `reqEnter_A` are **uncontrollable** visitor requests.
* The controller can only control `{lock_Hall, lock_A, unlock_Hall, unlock_A}`.
* But in the original model, `allow_*` / `deny_*` are emitted by the environment independently, so the visitor can effectively enter before the controller’s lock decision has any causal effect.
* This creates a race where the environment can force `arrive_Hall[2]` or `arrive_A[2]`, violating `LIMIT_HALL` / `LIMIT_A`.

To recover synthesizability, the environment must be revised so that **door permission is synchronized with lock state**:

* `allow_*` only when the corresponding door is unlocked,
* `deny_*` only when locked.

This preserves all original safety properties and behavior, while making lock actions meaningfully enforceable.

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


//-----------------------------------------------------------------------------
// Visitor
//-----------------------------------------------------------------------------
VISITOR = VISIT,
VISIT =
(
      reqEnter_Hall  -> CHECK_HALL
    | reqEnter_A     -> CHECK_A
    | reqOut         -> CHECK_OUT
),

CHECK_HALL =
(
      allow_Hall -> arrive_Hall[1..N] -> VISIT
    | deny_Hall  -> notEnter          -> VISIT
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


//-----------------------------------------------------------------------------
// Occupancy models
//-----------------------------------------------------------------------------
HALL = HALL[0],
HALL[n:0..N] =
(
      when(n!=N) allow_Hall -> arrive_Hall[n+1] -> HALL[n+1]
    | when(n!=N) deny_A     -> arrive_Hall[n+1] -> HALL[n+1]
    | when(n!=0) reqEnter_A -> HALL[n-1]
).

ROOM_A = ROOM_A[0],
ROOM_A[n:0..N] =
(
      when(n!=N) allow_A -> arrive_A[n+1] -> ROOM_A[n+1]
    | when(n!=0) reqOut  -> ROOM_A[n-1]
).


//-----------------------------------------------------------------------------
// Door models: synchronize permission with lock state
//-----------------------------------------------------------------------------
DOOR_HALL = H_UNLOCKED,

H_UNLOCKED =
(
      allow_Hall  -> H_UNLOCKED
    | lock_Hall   -> H_LOCKED
),

H_LOCKED =
(
      deny_Hall   -> H_LOCKED
    | unlock_Hall -> H_UNLOCKED
).


DOOR_A = A_UNLOCKED,

A_UNLOCKED =
(
      allow_A  -> A_UNLOCKED
    | lock_A   -> A_LOCKED
),

A_LOCKED =
(
      deny_A   -> A_LOCKED
    | unlock_A -> A_UNLOCKED
).


//-----------------------------------------------------------------------------
// Full environment
//-----------------------------------------------------------------------------
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

### Summary of repair

* Added **permission-by-lock synchronization** through the door automata.
* No safety monitor changed.
* No process names or action names changed.
* No new actions were needed.
* Controller now has effective authority over admission, restoring synthesizability.

# Iteration 2

The remaining source of unrealizability is that admission decisions (`allow_Hall`, `deny_Hall`, `allow_A`, `deny_A`) are still **environment events**. Although the door automata constrain when they may occur, the controller cannot force a denial when a room is near capacity. In particular:

* After `reqEnter_Hall`, if the hall currently contains one visitor, the environment can still choose `allow_Hall`, producing `arrive_Hall[2]` and violating `LIMIT_HALL`.
* Similarly, after `reqEnter_A`, if room A contains one visitor, the environment can choose `allow_A`, producing `arrive_A[2]` and violating `LIMIT_A`.

A standard DCS repair is to introduce an explicit **controller authorization step** between a request and the resulting allow/deny outcome. The controller chooses whether admission is granted, while the original allow/deny actions are preserved as confirmation events.

## Revised DCS Model

```fsp
//*****************************************************************************
// Environment
//*****************************************************************************

const N = 2

set VisitorAction = {reqEnter_Hall,reqEnter_A,reqOut}

set LockAction    = {lock_Hall,lock_A,unlock_Hall,unlock_A}

/* controller authorization actions */
set GrantAction   = {grant_Hall,grant_A,reject_Hall,reject_A}

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
      reqEnter_Hall -> CHECK_HALL
    | reqEnter_A    -> CHECK_A
    | reqOut        -> CHECK_OUT
),

CHECK_HALL =
(
      grant_Hall  -> allow_Hall -> arrive_Hall[1..N] -> VISIT
    | reject_Hall -> deny_Hall  -> notEnter          -> VISIT
),

CHECK_A =
(
      grant_A  -> allow_A -> arrive_A[1..N]    -> VISIT
    | reject_A -> deny_A  -> arrive_Hall[1..N] -> VISIT
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
      when(n!=N) allow_Hall -> arrive_Hall[n+1] -> HALL[n+1]
    | when(n!=N) deny_A     -> arrive_Hall[n+1] -> HALL[n+1]
    | when(n!=0) reqEnter_A -> HALL[n-1]
).

ROOM_A = ROOM_A[0],

ROOM_A[n:0..N] =
(
      when(n!=N) allow_A -> arrive_A[n+1] -> ROOM_A[n+1]
    | when(n!=0) reqOut  -> ROOM_A[n-1]
).


//-----------------------------------------------------------------------------
// Door models
//-----------------------------------------------------------------------------
DOOR_HALL = H_UNLOCKED,

H_UNLOCKED =
(
      grant_Hall -> allow_Hall -> H_UNLOCKED
    | reject_Hall -> deny_Hall -> H_UNLOCKED
    | lock_Hall -> H_LOCKED
),

H_LOCKED =
(
      reject_Hall -> deny_Hall -> H_LOCKED
    | unlock_Hall -> H_UNLOCKED
).


DOOR_A = A_UNLOCKED,

A_UNLOCKED =
(
      grant_A -> allow_A -> A_UNLOCKED
    | reject_A -> deny_A -> A_UNLOCKED
    | lock_A -> A_LOCKED
),

A_LOCKED =
(
      reject_A -> deny_A -> A_LOCKED
    | unlock_A -> A_UNLOCKED
).


//-----------------------------------------------------------------------------
// Full environment
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

    controllable =
    {
        LockAction,
        grant_Hall, grant_A,
        reject_Hall, reject_A
    }
}


//*****************************************************************************
// Controller
//*****************************************************************************
controller ||Controller = (Environment)~{Specification}.
```

### Changes

* Added controllable authorization actions:

  * `grant_Hall`, `reject_Hall`
  * `grant_A`, `reject_A`
* Modified the visitor so admission outcomes occur only after controller authorization.
* Modified door processes so `allow_*` and `deny_*` become confirmations of the controller’s grant/reject decision.
* Kept all original safety properties unchanged.
* Preserved original arrival, denial, occupancy, and lock semantics as much as possible while giving the controller actual authority over admissions.
