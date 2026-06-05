# Iteration 1

The unrealizability is caused by an **environment-forced security violation** for user 2 (`ID=2`):

* After `request[2] -> select_data[2][d]`, the environment can force `ok[2][d]` even for **non-public data** (`d ∈ {2..5}`).
* Then `dataTransfer[2][d]` becomes unavoidable, violating `P_SECURITY_OK_02 = [](!{dataTransfer[2][DataID]}\{dataTransfer[2][Public_Data]})`.
* Since `ok/ng` are currently **uncontrollable environment choices**, the controller has no way to prevent this.

The standard repair is to add a **confirmation/permission handshake** before the environment commits to `ok` or `ng`, making that branch controllable by the synthesized controller while preserving all safety properties.

---

## Revised DCS Model

```fsp
//*****************************************************************************
// Environment
//*****************************************************************************

/*Set of Object*/
const N = 2 /* num of user */
const K = 5 /* num of data (virtual server) */

range SystemID = 1 .. N
range DataID = 1 .. K

/*Set of Actions*/
set Authority = {admin, official, general}
set RequestActions = {request[SystemID]}

/* added permit_judge as controllable synchronization */
set ControllableActions = {
    select_data[SystemID][DataID],
    permit_judge[SystemID][DataID],
    ok[SystemID][DataID],
    ng[SystemID][DataID],
    reject[SystemID][DataID],
    dataTransfer[SystemID][DataID],
    filter[SystemID],
    unfilter[SystemID],
    process[SystemID]
}

set FinishActions = {process[SystemID], reject[SystemID][DataID]}


/* Environment Transitions */
System(ID = 1) = REQUEST,

REQUEST =
        (request[ID] -> DATA_SELECTION),

DATA_SELECTION =
        (select_data[ID][data:DataID] -> JUDGE_PERMISSION[data]),

/* controller must explicitly allow judgement */
JUDGE_PERMISSION[data:DataID] =
        (permit_judge[ID][data] -> JUDGEMENT[data]),

/* original branching preserved */
JUDGEMENT[data:DataID] =
        (ok[ID][data] -> OK[data]
        | ng[ID][data] -> NG[data]),

NG[data:DataID] =
        (reject[ID][data] -> REQUEST),

OK[data:DataID] =
        FILTERING[data],

FILTERING[data:DataID] =
        (filter[ID] -> GET_DATA[data]
        | unfilter[ID] -> GET_DATA[data]),

GET_DATA[data:DataID] =
        (dataTransfer[ID][data] -> USE_DATA),

USE_DATA =
        (process[ID] -> REQUEST).


//*****************************************************************************
// Requirement
//*****************************************************************************
range Public_Data = 1 .. 1

/*FLTL*/
ltl_property P_SECURITY_Admin =
    [](!reject[1][DataID])

ltl_property P_SECURITY_OK_02 =
    [](!{dataTransfer[2][DataID]}\{dataTransfer[2][Public_Data]})

ltl_property P_SECURITY_NG_02 =
    [](!reject[2][Public_Data])

ltl_property P_NOT_OTHER_REQUEST_01 =
    [](request[1] ->
        X(!request[2] W
          (process[1]
           || reject[1][1]
           || reject[1][2]
           || reject[1][3]
           || reject[1][4]
           || reject[1][5])))

ltl_property P_NOT_OTHER_REQUEST_02 =
    [](request[2] ->
        X(!request[1] W
          (process[2]
           || reject[2][1]
           || reject[2][2]
           || reject[2][3]
           || reject[2][4]
           || reject[2][5])))

controllerSpec Specification =
{
    safety = {
        P_SECURITY_Admin,
        P_SECURITY_OK_02,
        P_SECURITY_NG_02,
        P_NOT_OTHER_REQUEST_01,
        P_NOT_OTHER_REQUEST_02
    }

    controllable = {ControllableActions}
}


//*****************************************************************************
// Controller
//*****************************************************************************

||Environment = (forall [id:SystemID] System(id)).
controller ||TraditionalController = (Environment)~{Specification}.
```

### Explanation of changes

1. Added new controllable action: `permit_judge[SystemID][DataID]`.
2. Inserted intermediate state `JUDGE_PERMISSION` between `select_data` and `JUDGEMENT`.
3. This forces the environment to **wait for controller authorization** before emitting `ok` or `ng`.
4. No safety property was changed; original branching and action names were preserved.
5. The controller can now block unsafe user-2 judgments on private data, restoring synthesizability.

# Iteration 2

The previous repair is still likely unrealizable because `permit_judge` only delays the problem. Once the controller enables `permit_judge[2][d]`, the environment may still nondeterministically choose either `ok` or `ng`. For user 2:

* If `d = 1` (public data), `ng[2][1]` leads to `reject[2][1]`, violating `P_SECURITY_NG_02`.
* If `d ∈ {2..5}` (private data), `ok[2][d]` leads to `dataTransfer[2][d]`, violating `P_SECURITY_OK_02`.

Thus the controller still cannot safely allow the environment to enter `JUDGEMENT` because the subsequent uncontrollable choice may force a violation.

A more effective repair is to make the **judgement outcome itself subject to controller permission**, while preserving the original `ok/ng` branching structure.

## Revised DCS Model

```fsp
//*****************************************************************************
// Environment
//*****************************************************************************

/*Set of Object*/
const N = 2 /* num of user */
const K = 5 /* num of data (virtual server) */

range SystemID = 1 .. N
range DataID = 1 .. K

/*Set of Actions*/
set Authority = {admin, official, general}
set RequestActions = {request[SystemID]}

set ControllableActions = {
    select_data[SystemID][DataID],

    /* added permission actions */
    permit_ok[SystemID][DataID],
    permit_ng[SystemID][DataID],

    ok[SystemID][DataID],
    ng[SystemID][DataID],

    reject[SystemID][DataID],
    dataTransfer[SystemID][DataID],

    filter[SystemID],
    unfilter[SystemID],
    process[SystemID]
}

set FinishActions = {process[SystemID], reject[SystemID][DataID]}


/* Environment Transitions */
System(ID = 1) = REQUEST,

REQUEST =
        (request[ID] -> DATA_SELECTION),

DATA_SELECTION =
        (select_data[ID][data:DataID] -> JUDGEMENT[data]),

/*
 * Environment still chooses between OK and NG,
 * but each outcome requires controller permission.
 */
JUDGEMENT[data:DataID] =
        (
            permit_ok[ID][data] -> ok[ID][data] -> OK[data]
          | permit_ng[ID][data] -> ng[ID][data] -> NG[data]
        ),

NG[data:DataID] =
        (reject[ID][data] -> REQUEST),

OK[data:DataID] =
        FILTERING[data],

FILTERING[data:DataID] =
        (
            filter[ID] -> GET_DATA[data]
          | unfilter[ID] -> GET_DATA[data]
        ),

GET_DATA[data:DataID] =
        (dataTransfer[ID][data] -> USE_DATA),

USE_DATA =
        (process[ID] -> REQUEST).


//*****************************************************************************
// Requirement
//*****************************************************************************
range Public_Data = 1 .. 1

/*FLTL*/
ltl_property P_SECURITY_Admin =
    [](!reject[1][DataID])

ltl_property P_SECURITY_OK_02 =
    [](!{dataTransfer[2][DataID]}\{dataTransfer[2][Public_Data]})

ltl_property P_SECURITY_NG_02 =
    [](!reject[2][Public_Data])

ltl_property P_NOT_OTHER_REQUEST_01 =
    [](request[1] ->
        X(!request[2] W
          (process[1]
           || reject[1][1]
           || reject[1][2]
           || reject[1][3]
           || reject[1][4]
           || reject[1][5])))

ltl_property P_NOT_OTHER_REQUEST_02 =
    [](request[2] ->
        X(!request[1] W
          (process[2]
           || reject[2][1]
           || reject[2][2]
           || reject[2][3]
           || reject[2][4]
           || reject[2][5])))

controllerSpec Specification =
{
    safety = {
        P_SECURITY_Admin,
        P_SECURITY_OK_02,
        P_SECURITY_NG_02,
        P_NOT_OTHER_REQUEST_01,
        P_NOT_OTHER_REQUEST_02
    }

    controllable = {ControllableActions}
}


//*****************************************************************************
// Controller
//*****************************************************************************

||Environment = (forall [id:SystemID] System(id)).
controller ||TraditionalController = (Environment)~{Specification}.
```

### Explanation of changes

* Removed the ineffective single gate `permit_judge`.
* Added two controllable permission actions:

  * `permit_ok[SystemID][DataID]`
  * `permit_ng[SystemID][DataID]`
* Preserved the original `ok` / `ng` branching structure and all safety properties.
* The controller can now selectively authorize only safe outcomes:

  * For user 2 and public data (`data=1`), authorize `ok` and block `ng`.
  * For user 2 and non-public data (`data=2..5`), authorize `ng` and block `ok`.
  * User 1 remains unrestricted except as required by the existing specifications.

This converts the previously uncontrollable safety-critical choice into a controller-mediated choice without weakening any requirement.

# Iteration 3

The current repair still has a fundamental DCS problem: after `select_data[ID][data]`, the system can deadlock if neither `permit_ok` nor `permit_ng` is enabled. Many DCS tools treat blocking all outgoing controllable actions as losing when an uncontrollable request is waiting. More importantly, the environment no longer actually chooses the judgement outcome; the controller does. This changes the semantics of the original model substantially.

A more conservative repair is to keep the environment's `ok/ng` decision structure intact and introduce a **controller-mediated confirmation stage after the judgement but before the safety-critical effects** (`reject` and `dataTransfer`). This preserves the original branching while allowing the controller to suppress only those consequences that would violate the safety properties.

## Revised DCS Model

```fsp
//*****************************************************************************
// Environment
//*****************************************************************************

/*Set of Object*/
const N = 2 /* num of user */
const K = 5 /* num of data (virtual server) */

range SystemID = 1 .. N
range DataID = 1 .. K

/*Set of Actions*/
set Authority = {admin, official, general}
set RequestActions = {request[SystemID]}

set ControllableActions = {
    select_data[SystemID][DataID],

    /* added confirmation actions */
    permit_transfer[SystemID][DataID],
    permit_reject[SystemID][DataID],

    ok[SystemID][DataID],
    ng[SystemID][DataID],
    reject[SystemID][DataID],
    dataTransfer[SystemID][DataID],

    filter[SystemID],
    unfilter[SystemID],
    process[SystemID]
}

set FinishActions = {process[SystemID], reject[SystemID][DataID]}


/* Environment Transitions */
System(ID = 1) = REQUEST,

REQUEST =
    (request[ID] -> DATA_SELECTION),

DATA_SELECTION =
    (select_data[ID][data:DataID] -> JUDGEMENT[data]),

/* preserve original environment choice */
JUDGEMENT[data:DataID] =
    (
        ok[ID][data] -> OK[data]
      | ng[ID][data] -> NG[data]
    ),

/* rejection requires controller confirmation */
NG[data:DataID] =
    (permit_reject[ID][data] -> reject[ID][data] -> REQUEST),

OK[data:DataID] =
    FILTERING[data],

FILTERING[data:DataID] =
    (
        filter[ID] -> GET_DATA[data]
      | unfilter[ID] -> GET_DATA[data]
    ),

/* transfer requires controller confirmation */
GET_DATA[data:DataID] =
    (permit_transfer[ID][data] ->
        dataTransfer[ID][data] -> USE_DATA),

USE_DATA =
    (process[ID] -> REQUEST).


//*****************************************************************************
// Requirement
//*****************************************************************************
range Public_Data = 1 .. 1

/******************************/
/* FLTL                       */
/******************************/

ltl_property P_SECURITY_Admin =
    [](!reject[1][DataID])

ltl_property P_SECURITY_OK_02 =
    [](!{dataTransfer[2][DataID]}
        \{dataTransfer[2][Public_Data]})

ltl_property P_SECURITY_NG_02 =
    [](!reject[2][Public_Data])

ltl_property P_NOT_OTHER_REQUEST_01 =
    [](request[1] ->
        X(!request[2] W
            (process[1]
             || reject[1][1]
             || reject[1][2]
             || reject[1][3]
             || reject[1][4]
             || reject[1][5])))

ltl_property P_NOT_OTHER_REQUEST_02 =
    [](request[2] ->
        X(!request[1] W
            (process[2]
             || reject[2][1]
             || reject[2][2]
             || reject[2][3]
             || reject[2][4]
             || reject[2][5])))

controllerSpec Specification =
{
    safety = {
        P_SECURITY_Admin,
        P_SECURITY_OK_02,
        P_SECURITY_NG_02,
        P_NOT_OTHER_REQUEST_01,
        P_NOT_OTHER_REQUEST_02
    }

    controllable = {ControllableActions}
}


//*****************************************************************************
// Controller
//*****************************************************************************

||Environment = (forall [id:SystemID] System(id)).
controller ||TraditionalController = (Environment)~{Specification}.
```

### Explanation of changes

1. Preserved the original environment judgement choice:

   * `ok[ID][data] -> OK[data]`
   * `ng[ID][data] -> NG[data]`

2. Added only two new controllable synchronization actions:

   * `permit_transfer[ID][data]`
   * `permit_reject[ID][data]`

3. Moved controller authority to the safety-critical effects:

   * A rejection cannot occur until `permit_reject`.
   * A data transfer cannot occur until `permit_transfer`.

4. All original safety properties remain unchanged.

This repair keeps the original model structure and branching behavior much closer to the source model while giving the controller the minimum additional synchronization needed to prevent unsafe `reject` and `dataTransfer` events.

# Iteration 4

The previous repair is still problematic from a synthesis perspective. After an uncontrollable `ok` or `ng`, the system may enter a state where the only enabled continuation is a controllable permission (`permit_transfer` or `permit_reject`). If the controller disables that permission to avoid a safety violation, the execution deadlocks. Many DCS formulations do not allow the controller to win merely by blocking progress after an uncontrollable choice that has already committed to an unsafe branch.

A more robust repair is to prevent the environment from committing to an unsafe branch in the first place. The smallest structural change is to insert a **controller-controlled authorization step before the environment reveals the judgement outcome**, while preserving the original `ok/ng` and downstream structure.

## Revised DCS Model

```fsp
//*****************************************************************************
// Environment
//*****************************************************************************

/*Set of Object*/
const N = 2 /* num of user */
const K = 5 /* num of data (virtual server) */

range SystemID = 1 .. N
range DataID = 1 .. K

/*Set of Actions*/
set Authority = {admin, official, general}
set RequestActions = {request[SystemID]}

set ControllableActions = {
    select_data[SystemID][DataID],

    /* controller authorizes which judgement branch may occur */
    allow_ok[SystemID][DataID],
    allow_ng[SystemID][DataID],

    ok[SystemID][DataID],
    ng[SystemID][DataID],

    reject[SystemID][DataID],
    dataTransfer[SystemID][DataID],

    filter[SystemID],
    unfilter[SystemID],
    process[SystemID]
}

set FinishActions = {process[SystemID], reject[SystemID][DataID]}


/* Environment Transitions */
System(ID = 1) = REQUEST,

REQUEST =
    (request[ID] -> DATA_SELECTION),

DATA_SELECTION =
    (select_data[ID][data:DataID] -> JUDGEMENT[data]),

/*
 * Controller must authorize the outcome before the
 * environment can perform the corresponding judgement.
 */
JUDGEMENT[data:DataID] =
    (
        allow_ok[ID][data] -> ok[ID][data] -> OK[data]
      | allow_ng[ID][data] -> ng[ID][data] -> NG[data]
    ),

NG[data:DataID] =
    (reject[ID][data] -> REQUEST),

OK[data:DataID] =
    FILTERING[data],

FILTERING[data:DataID] =
    (
        filter[ID] -> GET_DATA[data]
      | unfilter[ID] -> GET_DATA[data]
    ),

GET_DATA[data:DataID] =
    (dataTransfer[ID][data] -> USE_DATA),

USE_DATA =
    (process[ID] -> REQUEST).


//*****************************************************************************
// Requirement
//*****************************************************************************
range Public_Data = 1 .. 1

/******************************/
/* FLTL                       */
/******************************/

ltl_property P_SECURITY_Admin =
    [](!reject[1][DataID])

ltl_property P_SECURITY_OK_02 =
    [](!{dataTransfer[2][DataID]}
        \{dataTransfer[2][Public_Data]})

ltl_property P_SECURITY_NG_02 =
    [](!reject[2][Public_Data])

ltl_property P_NOT_OTHER_REQUEST_01 =
    [](request[1] ->
        X(!request[2] W
            (process[1]
             || reject[1][1]
             || reject[1][2]
             || reject[1][3]
             || reject[1][4]
             || reject[1][5])))

ltl_property P_NOT_OTHER_REQUEST_02 =
    [](request[2] ->
        X(!request[1] W
            (process[2]
             || reject[2][1]
             || reject[2][2]
             || reject[2][3]
             || reject[2][4]
             || reject[2][5])))

controllerSpec Specification =
{
    safety = {
        P_SECURITY_Admin,
        P_SECURITY_OK_02,
        P_SECURITY_NG_02,
        P_NOT_OTHER_REQUEST_01,
        P_NOT_OTHER_REQUEST_02
    }

    controllable = {ControllableActions}
}


//*****************************************************************************
// Controller
//*****************************************************************************

||Environment = (forall [id:SystemID] System(id)).
controller ||TraditionalController = (Environment)~{Specification}.
```

### Explanation of changes

* Added two new controllable synchronization actions:

  * `allow_ok[SystemID][DataID]`
  * `allow_ng[SystemID][DataID]`

* Inserted the authorization step immediately before the environment performs `ok` or `ng`.

* Preserved:

  * all process names,
  * all original actions,
  * the `ok/ng` branching structure,
  * the filtering/data-transfer/reject workflow,
  * every safety property unchanged.

This allows the controller to authorize only judgement outcomes that can subsequently satisfy the security constraints (e.g., for user 2, allow `ok` only on public data and allow `ng` only on non-public data), recovering synthesizability without weakening the specification.

# Iteration 5

The key source of unrealizability is not only the privacy constraints on user 2, but also the mutual-exclusion requirements:

* `P_NOT_OTHER_REQUEST_01` forbids `request[2]` after `request[1]` until user 1 finishes.
* `P_NOT_OTHER_REQUEST_02` symmetrically forbids `request[1]` after `request[2]` until user 2 finishes.

In the original environment, `request[1]` and `request[2]` are uncontrollable and independent. Therefore, after one user issues a request, the other user can immediately issue a request and violate the property before the controller has any opportunity to intervene.

The most conservative repair is therefore:

1. Keep the authorization mechanism before the `ok/ng` judgment (to resolve the user-2 privacy conflict).
2. Add a global request-admission synchronization so that a request must be authorized before it is emitted. This gives the controller the ability to serialize requests and satisfy the two `P_NOT_OTHER_REQUEST_*` properties.

## Revised DCS Model

```fsp
//*****************************************************************************
// Environment
//*****************************************************************************

/*Set of Object*/
const N = 2 /* num of user */
const K = 5 /* num of data (virtual server) */

range SystemID = 1 .. N
range DataID = 1 .. K

/*Set of Actions*/
set Authority = {admin, official, general}
set RequestActions = {request[SystemID]}

set ControllableActions = {

    /* request admission control */
    allow_request[SystemID],

    select_data[SystemID][DataID],

    /* judgement authorization */
    allow_ok[SystemID][DataID],
    allow_ng[SystemID][DataID],

    ok[SystemID][DataID],
    ng[SystemID][DataID],

    reject[SystemID][DataID],
    dataTransfer[SystemID][DataID],

    filter[SystemID],
    unfilter[SystemID],
    process[SystemID]
}

set FinishActions = {process[SystemID], reject[SystemID][DataID]}


/* Environment Transitions */
System(ID = 1) = REQUEST,

REQUEST =
    (
        allow_request[ID] ->
        request[ID] ->
        DATA_SELECTION
    ),

DATA_SELECTION =
    (
        select_data[ID][data:DataID] ->
        JUDGEMENT[data]
    ),

/*
 * Controller authorizes which judgement branch
 * the environment may take.
 */
JUDGEMENT[data:DataID] =
    (
        allow_ok[ID][data] ->
        ok[ID][data] ->
        OK[data]

      |

        allow_ng[ID][data] ->
        ng[ID][data] ->
        NG[data]
    ),

NG[data:DataID] =
    (
        reject[ID][data] ->
        REQUEST
    ),

OK[data:DataID] =
    FILTERING[data],

FILTERING[data:DataID] =
    (
        filter[ID]   -> GET_DATA[data]
      | unfilter[ID] -> GET_DATA[data]
    ),

GET_DATA[data:DataID] =
    (
        dataTransfer[ID][data] ->
        USE_DATA
    ),

USE_DATA =
    (
        process[ID] ->
        REQUEST
    ).


//*****************************************************************************
// Requirement
//*****************************************************************************
range Public_Data = 1 .. 1

/*FLTL*/
ltl_property P_SECURITY_Admin =
    [](!reject[1][DataID])

ltl_property P_SECURITY_OK_02 =
    [](!{dataTransfer[2][DataID]}
        \{dataTransfer[2][Public_Data]})

ltl_property P_SECURITY_NG_02 =
    [](!reject[2][Public_Data])

ltl_property P_NOT_OTHER_REQUEST_01 =
    [](request[1] ->
        X(!request[2] W
          (process[1]
           || reject[1][1]
           || reject[1][2]
           || reject[1][3]
           || reject[1][4]
           || reject[1][5])))

ltl_property P_NOT_OTHER_REQUEST_02 =
    [](request[2] ->
        X(!request[1] W
          (process[2]
           || reject[2][1]
           || reject[2][2]
           || reject[2][3]
           || reject[2][4]
           || reject[2][5])))

controllerSpec Specification =
{
    safety = {
        P_SECURITY_Admin,
        P_SECURITY_OK_02,
        P_SECURITY_NG_02,
        P_NOT_OTHER_REQUEST_01,
        P_NOT_OTHER_REQUEST_02
    }

    controllable = {ControllableActions}
}


//*****************************************************************************
// Controller
//*****************************************************************************

||Environment = (forall [id:SystemID] System(id)).
controller ||TraditionalController = (Environment)~{Specification}.
```

### Explanation of changes

* Added `allow_request[SystemID]` before each `request[ID]`.

  * This lets the controller serialize requests and satisfy `P_NOT_OTHER_REQUEST_01` and `P_NOT_OTHER_REQUEST_02`.

* Kept the judgment authorization mechanism:

  * `allow_ok[ID][data]`
  * `allow_ng[ID][data]`

  This lets the controller prevent:

  * `dataTransfer[2][2..5]` (violates `P_SECURITY_OK_02`)
  * `reject[2][1]` (violates `P_SECURITY_NG_02`)

* Preserved all original safety properties, process names, action names, and the overall request → select → judgement → filter → transfer/reject workflow.

This repair addresses both independent causes of unrealizability: uncontrollable concurrent requests and uncontrollable security-sensitive judgement outcomes.
