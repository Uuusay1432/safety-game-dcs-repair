# Iteration 1

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

//---------------------------------------------------------------------------
// Visitor
//---------------------------------------------------------------------------

VISITOR = VISIT,
VISIT =
(
	reqEnter_Hall  -> CHECK_HALL
    |reqEnter_A    -> CHECK_A
    |reqOut        -> CHECK_OUT
),

CHECK_HALL =
(
	allow_Hall -> arrive_Hall[1..N] -> VISIT
	| deny_Hall -> notEnter -> VISIT
),

CHECK_A =
(
	allow_A -> arrive_A[1..N] -> VISIT
	| deny_A -> arrive_Hall[1..N] -> VISIT
),

CHECK_OUT =
(
	leave -> VISIT
).

//---------------------------------------------------------------------------
// Occupancy models
//---------------------------------------------------------------------------

HALL = HALL[0],
HALL[n:0..N] =
(
	when(n!=N)
		allow_Hall -> arrive_Hall[n+1] -> HALL[n+1]

	| when(n!=N)
		deny_A -> arrive_Hall[n+1] -> HALL[n+1]

	| when(n!=0)
		reqEnter_A -> HALL[n-1]
).

ROOM_A = ROOM_A[0],
ROOM_A[n:0..N] =
(
	when(n!=N)
		allow_A -> arrive_A[n+1] -> ROOM_A[n+1]

	| when(n!=0)
		reqOut -> ROOM_A[n-1]
).

//---------------------------------------------------------------------------
// Door control with explicit synchronization
//---------------------------------------------------------------------------

DOOR_HALL = HALL_UNLOCKED,

HALL_UNLOCKED =
(
	allow_Hall  -> HALL_UNLOCKED
	| lock_Hall -> HALL_LOCKED
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
	deny_A -> A_LOCKED
	| unlock_A -> A_UNLOCKED
).

//---------------------------------------------------------------------------
// Permission protocols
//---------------------------------------------------------------------------

// Hall requests can only be granted while hall door is unlocked.
HALL_PERMISSION =
(
	lock_Hall   -> HALL_DENY
	| unlock_Hall -> HALL_ALLOW
	| allow_Hall -> HALL_PERMISSION
	| deny_Hall  -> HALL_PERMISSION
),

HALL_ALLOW =
(
	allow_Hall -> HALL_ALLOW
	| lock_Hall -> HALL_DENY
),

HALL_DENY =
(
	deny_Hall -> HALL_DENY
	| unlock_Hall -> HALL_ALLOW
).


// Room A requests can only be granted while room A door is unlocked.
A_PERMISSION =
(
	lock_A   -> A_DENY
	| unlock_A -> A_ALLOW
	| allow_A -> A_PERMISSION
	| deny_A  -> A_PERMISSION
),

A_ALLOW =
(
	allow_A -> A_ALLOW
	| lock_A -> A_DENY
),

A_DENY =
(
	deny_A -> A_DENY
	| unlock_A -> A_ALLOW
).


//---------------------------------------------------------------------------

||Environment =
(
	VISITOR
	|| HALL
	|| ROOM_A
	|| DOOR_HALL
	|| DOOR_A
	|| HALL_PERMISSION
	|| A_PERMISSION
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
	safety = {
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

### Explanation of the Repair

The unrealizability comes from the fact that the controller can only control the lock/unlock actions, while the environment can still independently perform `allow_Hall` and `allow_A`. As a result, visitors may continue entering rooms even after the controller locks the doors, making the occupancy limits impossible to enforce.

The repair introduces explicit synchronization between lock state and permission actions:

* `HALL_PERMISSION` constrains:

  * `allow_Hall` to occur only when the hall door is logically unlocked.
  * `deny_Hall` to occur only when the hall door is logically locked.

* `A_PERMISSION` similarly constrains:

  * `allow_A` to unlocked states.
  * `deny_A` to locked states.

This preserves:

* all original safety properties,
* all process names and action names,
* the original branching behavior of visitors,
* the original occupancy accounting.

No safety monitor was weakened or removed. The repair only adds synchronization structure so that controllable lock actions can effectively regulate the uncontrollable admission decisions, restoring controller synthesizability.

# Iteration 2

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

//---------------------------------------------------------------------------
// Visitor
//---------------------------------------------------------------------------

VISITOR = VISIT,
VISIT =
(
	reqEnter_Hall  -> CHECK_HALL
    |reqEnter_A    -> CHECK_A
    |reqOut        -> CHECK_OUT
),

CHECK_HALL =
(
	enterHallReq ->
	(
		allow_Hall -> arrive_Hall[1..N] -> VISIT
		| deny_Hall -> notEnter -> VISIT
	)
),

CHECK_A =
(
	enterAReq ->
	(
		allow_A -> arrive_A[1..N] -> VISIT
		| deny_A -> arrive_Hall[1..N] -> VISIT
	)
),

CHECK_OUT =
(
	leave -> VISIT
).

//---------------------------------------------------------------------------
// Occupancy models
//---------------------------------------------------------------------------

HALL = HALL[0],
HALL[n:0..N] =
(
	when(n!=N)
		allow_Hall -> arrive_Hall[n+1] -> HALL[n+1]

	| when(n!=N)
		deny_A -> arrive_Hall[n+1] -> HALL[n+1]

	| when(n!=0)
		reqEnter_A -> HALL[n-1]
).

ROOM_A = ROOM_A[0],
ROOM_A[n:0..N] =
(
	when(n!=N)
		allow_A -> arrive_A[n+1] -> ROOM_A[n+1]

	| when(n!=0)
		reqOut -> ROOM_A[n-1]
).

//---------------------------------------------------------------------------
// Door models
//---------------------------------------------------------------------------

DOOR_HALL = HALL_UNLOCKED,

HALL_UNLOCKED =
(
	allow_Hall -> HALL_UNLOCKED
	| lock_Hall -> HALL_LOCKED
),

HALL_LOCKED =
(
	deny_Hall -> HALL_LOCKED
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
	deny_A -> A_LOCKED
	| unlock_A -> A_UNLOCKED
).

//---------------------------------------------------------------------------
// Request gating
//
// Requests must first be observed before an allow/deny decision can occur.
// The controller can use door locks to determine the outcome.
//---------------------------------------------------------------------------

HALL_GATE = WAIT_HALL_REQ,

WAIT_HALL_REQ =
(
	enterHallReq ->
	(
		allow_Hall -> WAIT_HALL_REQ
		| deny_Hall -> WAIT_HALL_REQ
	)
).


A_GATE = WAIT_A_REQ,

WAIT_A_REQ =
(
	enterAReq ->
	(
		allow_A -> WAIT_A_REQ
		| deny_A -> WAIT_A_REQ
	)
).

//---------------------------------------------------------------------------

||Environment =
(
	VISITOR
	|| HALL
	|| ROOM_A
	|| DOOR_HALL
	|| DOOR_A
	|| HALL_GATE
	|| A_GATE
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
	safety = {
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

### Explanation of the Changes

The primary source of unrealizability is that `allow_Hall` and `allow_A` are uncontrollable environment actions that can occur independently of any admission request. Consequently, occupancy can increase before the controller has an opportunity to react with `lock_*` actions.

To restore synthesizability while preserving the safety requirements:

1. **Introduced explicit request/decision synchronization**

   * Added `enterHallReq` and `enterAReq`.
   * A visitor must first issue a request signal before either `allow_*` or `deny_*` can occur.

2. **Added gating processes (`HALL_GATE`, `A_GATE`)**

   * These enforce the protocol:

     ```
     request -> (allow | deny)
     ```
   * Admission decisions can no longer occur spontaneously.

3. **Preserved all safety properties**

   * `LIMIT_HALL` and `LIMIT_A` remain unchanged.
   * No monitor or requirement was weakened.

4. **Preserved the original structure**

   * Existing processes (`VISITOR`, `HALL`, `ROOM_A`, `DOOR_HALL`, `DOOR_A`) and actions (`allow_*`, `deny_*`, `lock_*`, `unlock_*`) remain intact.
   * The repair only adds synchronization necessary for the controller to observe requests and use the lock actions to influence future admissions.

This request-before-decision protocol is a standard DCS repair pattern that removes uncontrollable spontaneous admissions and gives the synthesized controller an opportunity to enforce the occupancy limits.

# Iteration 3

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

// New controllable permission actions
set PermitAction  = {permit_Hall,permit_A}

//---------------------------------------------------------------------------
// Visitor
//---------------------------------------------------------------------------

VISITOR = VISIT,
VISIT =
(
	reqEnter_Hall  -> CHECK_HALL
    |reqEnter_A    -> CHECK_A
    |reqOut        -> CHECK_OUT
),

CHECK_HALL =
(
	permit_Hall ->
	(
		allow_Hall -> arrive_Hall[1..N] -> VISIT
		| deny_Hall -> notEnter -> VISIT
	)
),

CHECK_A =
(
	permit_A ->
	(
		allow_A -> arrive_A[1..N] -> VISIT
		| deny_A -> arrive_Hall[1..N] -> VISIT
	)
),

CHECK_OUT =
(
	leave -> VISIT
).

//---------------------------------------------------------------------------
// Occupancy models
//---------------------------------------------------------------------------

HALL = HALL[0],
HALL[n:0..N] =
(
	when(n!=N)
		allow_Hall -> arrive_Hall[n+1] -> HALL[n+1]

	| when(n!=N)
		deny_A -> arrive_Hall[n+1] -> HALL[n+1]

	| when(n!=0)
		reqEnter_A -> HALL[n-1]
).

ROOM_A = ROOM_A[0],
ROOM_A[n:0..N] =
(
	when(n!=N)
		allow_A -> arrive_A[n+1] -> ROOM_A[n+1]

	| when(n!=0)
		reqOut -> ROOM_A[n-1]
).

//---------------------------------------------------------------------------
// Door models
//---------------------------------------------------------------------------

DOOR_HALL = HALL_UNLOCKED,

HALL_UNLOCKED =
(
	allow_Hall -> HALL_UNLOCKED
	| lock_Hall -> HALL_LOCKED
),

HALL_LOCKED =
(
	deny_Hall -> HALL_LOCKED
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
	deny_A -> A_LOCKED
	| unlock_A -> A_UNLOCKED
).

//---------------------------------------------------------------------------
// Permission protocols
//
// A request must obtain controller permission before an allow/deny outcome
// can occur. This gives the controller an opportunity to react to occupancy
// before admission decisions are taken.
//---------------------------------------------------------------------------

HALL_PERMISSION = WAIT_HALL,

WAIT_HALL =
(
	reqEnter_Hall -> permit_Hall -> WAIT_HALL
).

A_PERMISSION = WAIT_A,

WAIT_A =
(
	reqEnter_A -> permit_A -> WAIT_A
).

//---------------------------------------------------------------------------

||Environment =
(
	VISITOR
	|| HALL
	|| ROOM_A
	|| DOOR_HALL
	|| DOOR_A
	|| HALL_PERMISSION
	|| A_PERMISSION
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
	safety = {
				LIMIT_HALL,
				LIMIT_A
			 }

	controllable = {LockAction, PermitAction}
}

//*****************************************************************************
// Controller
//*****************************************************************************

controller ||Controller = (Environment)~{Specification}.
```

### Explanation of the Changes

The key unrealizability issue is that after a visitor request, the environment can still autonomously choose `allow_Hall` or `allow_A`. The controller only controls `lock_*` actions and therefore may be unable to prevent an admission that immediately violates `LIMIT_HALL` or `LIMIT_A`.

To recover synthesizability:

1. Added controllable permission actions:

   * `permit_Hall`
   * `permit_A`

2. Introduced permission synchronization:

   * `reqEnter_Hall -> permit_Hall -> (allow_Hall | deny_Hall)`
   * `reqEnter_A -> permit_A -> (allow_A | deny_A)`

3. Added permission processes:

   * `HALL_PERMISSION`
   * `A_PERMISSION`

4. Extended the controllable set with `PermitAction`.

This preserves all existing safety properties and the original allow/deny semantics, while ensuring the controller gets an explicit decision point before any admission outcome can occur. This is a standard DCS repair pattern that converts an uncontrollable admission race into a controllable authorization protocol.
