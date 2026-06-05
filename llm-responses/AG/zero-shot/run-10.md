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

// New controllable permission actions
set PermitAction  = {permit_Hall,permit_A,permit_Out}

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
  | deny_Hall  -> notEnter         -> VISIT
),
CHECK_A =
(
	allow_A -> arrive_A[1..N] -> VISIT
  | deny_A  -> arrive_Hall[1..N] -> VISIT
),
CHECK_OUT =
(
	leave -> VISIT
).

// Requests to move are now synchronized with controller-granted permissions.
HALL = HALL[0],
HALL[n:0..N] =
(
	when(n!=N) allow_Hall -> arrive_Hall[n+1] -> HALL[n+1]
  | when(n!=N) deny_A     -> arrive_Hall[n+1] -> HALL[n+1]
  | when(n!=0) permit_A   -> reqEnter_A       -> HALL[n-1]
).

ROOM_A = ROOM_A[0],
ROOM_A[n:0..N] =
(
	when(n!=N) allow_A    -> arrive_A[n+1] -> ROOM_A[n+1]
  | when(n!=0) permit_Out -> reqOut      -> ROOM_A[n-1]
).

DOOR_HALL = UNLOCKED_HALL,
UNLOCKED_HALL =
(
	allow_Hall -> UNLOCKED_HALL
  | lock_Hall  -> LOCKED_HALL
),
LOCKED_HALL =
(
	deny_Hall   -> LOCKED_HALL
  | unlock_Hall -> UNLOCKED_HALL
).

DOOR_A = UNLOCKED_A,
UNLOCKED_A =
(
	allow_A -> UNLOCKED_A
  | lock_A  -> LOCKED_A
),
LOCKED_A =
(
	deny_A    -> LOCKED_A
  | unlock_A -> UNLOCKED_A
).

// Controller-controlled permissions.
// If a door is unlocked, permission may be issued.
// If locked, permission is suppressed.
GATE_HALL = GATE_HALL_U,
GATE_HALL_U =
(
	permit_Hall -> GATE_HALL_U
  | lock_Hall  -> GATE_HALL_L
),
GATE_HALL_L =
(
	unlock_Hall -> GATE_HALL_U
).

GATE_A = GATE_A_U,
GATE_A_U =
(
	permit_A   -> GATE_A_U
  | permit_Out -> GATE_A_U
  | lock_A     -> GATE_A_L
),
GATE_A_L =
(
	unlock_A -> GATE_A_U
).

||Environment =
(
	VISITOR
 || HALL
 || ROOM_A
 || DOOR_HALL
 || DOOR_A
 || GATE_HALL
 || GATE_A
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
	controllable = {LockAction, PermitAction}
}

//*****************************************************************************
// Controller
//*****************************************************************************

controller ||Controller = (Environment)~{Specification}.
```

### Explanation of the repair

The unrealizability comes from the fact that the safety properties constrain occupancy (`arrive_Hall[2]` and `arrive_A[2]` must never occur), but the controller can only issue lock/unlock commands. The environment can autonomously generate `reqEnter_A` and `reqOut` transitions, causing people to move between rooms regardless of whether the controller is ready to permit such movement. In particular, once a room contains a visitor, uncontrollable movement can force a second arrival event and violate the occupancy limits.

To recover synthesizability:

1. **Added controllable permission actions**

   * `permit_Hall`
   * `permit_A`
   * `permit_Out`

2. **Introduced synchronization before internal movement**

   * `HALL` now requires `permit_A -> reqEnter_A` before a visitor can leave the hall and enter room A.
   * `ROOM_A` now requires `permit_Out -> reqOut` before a visitor can leave room A.

3. **Added gate processes (`GATE_HALL`, `GATE_A`)**

   * These synchronize permissions with the lock state.
   * When a door is locked, the corresponding permission actions are suppressed.
   * When unlocked, the controller may grant permissions.

4. **Preserved the original safety requirements unchanged**

   * `LIMIT_HALL` and `LIMIT_A` are exactly as given.
   * No monitor or LTL property was weakened.

The repair gives the controller direct authority over occupancy-changing movements, allowing it to prevent states that would produce `arrive_Hall[2]` or `arrive_A[2]`, thereby restoring synthesizability.

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

// Added controllable permission actions
set PermitAction  = {permitEnter_Hall,permitEnter_A,permitOut}

VISITOR = VISIT,
VISIT =
(
	reqEnter_Hall  -> WAIT_HALL
  | reqEnter_A     -> WAIT_A
  | reqOut         -> WAIT_OUT
),

// Requests must be explicitly permitted before the corresponding
// allow/deny outcome can occur.
WAIT_HALL =
(
	permitEnter_Hall ->
		(allow_Hall -> arrive_Hall[1..N] -> VISIT
		|deny_Hall  -> notEnter          -> VISIT)
),

WAIT_A =
(
	permitEnter_A ->
		(allow_A -> arrive_A[1..N]    -> VISIT
		|deny_A  -> arrive_Hall[1..N] -> VISIT)
),

WAIT_OUT =
(
	permitOut ->
		(leave -> VISIT)
).

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
  | when(n!=0) reqOut -> ROOM_A[n-1]
).

DOOR_HALL = UNLOCKED_HALL,
UNLOCKED_HALL =
(
	allow_Hall -> UNLOCKED_HALL
  | lock_Hall -> LOCKED_HALL
),
LOCKED_HALL =
(
	deny_Hall   -> LOCKED_HALL
  | unlock_Hall -> UNLOCKED_HALL
).

DOOR_A = UNLOCKED_A,
UNLOCKED_A =
(
	allow_A -> UNLOCKED_A
  | lock_A -> LOCKED_A
),
LOCKED_A =
(
	deny_A    -> LOCKED_A
  | unlock_A -> UNLOCKED_A
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
	safety       = {
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

### Explanation of the changes

The main source of unrealizability is that the environment can autonomously generate `reqEnter_Hall`, `reqEnter_A`, and `reqOut`, and once a request occurs the subsequent admission outcome (`allow_*` / `deny_*`) may force occupancy-changing events (`arrive_Hall[*]`, `arrive_A[*]`) that can violate `LIMIT_HALL` or `LIMIT_A`. The original controller only controls lock actions and therefore may not be able to prevent unsafe request processing.

To restore synthesizability while preserving the safety requirements:

1. Added three new controllable permission actions:

   * `permitEnter_Hall`
   * `permitEnter_A`
   * `permitOut`

2. Inserted synchronization states (`WAIT_HALL`, `WAIT_A`, `WAIT_OUT`) in the visitor process so that every request must first receive controller permission before the corresponding admission or exit outcome can occur.

3. Kept all original process names, action names, occupancy updates, and safety monitors unchanged.

4. Extended the controllable action set to include the new permission actions.

This gives the synthesized controller the ability to delay or suppress processing of requests that would inevitably lead to a violation of `LIMIT_HALL` or `LIMIT_A`, without weakening the requirements.
