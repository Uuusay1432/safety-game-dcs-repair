# Iteration 1
The unrealizability comes from **uncontrollable environment choices** at the `JUDGEMENT` state:

* `ok[ID][data]` and `ng[ID][data]` are currently **controllable**, but once `ok` occurs for user 2 on private data (`DataID = 2..5`), the environment can immediately force `dataTransfer[2][data]`, violating `P_SECURITY_OK_02`.
* Likewise, if `ng[2][1]` occurs, the controller must avoid `reject[2][1]` because of `P_SECURITY_NG_02`, but the original model has no alternative path after `NG`, causing deadlock/unrealizability.
* The fix is to insert a **permission/confirmation layer before transfer and rejection**, so the controller can safely suppress forbidden actions without weakening the requirements.

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

/* added permit_transfer / permit_reject as controllable permission signals */
set ControllableActions = {
    select_data[SystemID][DataID],
    ok[SystemID][DataID],
    ng[SystemID][DataID],
    reject[SystemID][DataID],
    dataTransfer[SystemID][DataID],
    filter[SystemID],
    unfilter[SystemID],
    process[SystemID],
    permit_transfer[SystemID][DataID],
    permit_reject[SystemID][DataID]
}

set FinishActions = {process[SystemID], reject[SystemID][DataID]}

/* Environment Transitions */
System(ID = 1) = REQUEST,

REQUEST =
    (request[ID] -> DATA_SELECTION),

DATA_SELECTION =
    (select_data[ID][data:DataID] -> JUDGEMENT[data]),

JUDGEMENT[data:DataID] =
    (ok[ID][data] -> OK[data]
    | ng[ID][data] -> NG[data]),

/* added permission before reject */
NG[data:DataID] =
    (permit_reject[ID][data] -> reject[ID][data] -> REQUEST
    | process[ID] -> REQUEST),

OK[data:DataID] =
    FILTERING[data],

FILTERING[data:DataID] =
    (filter[ID] -> GET_DATA[data]
    | unfilter[ID] -> GET_DATA[data]),

/* added permission before data transfer */
GET_DATA[data:DataID] =
    (permit_transfer[ID][data] -> dataTransfer[ID][data] -> USE_DATA
    | process[ID] -> REQUEST),

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

1. **Added `permit_transfer[ID][data]` before `dataTransfer`**

   * lets the controller block illegal transfers (especially user 2 on private data).

2. **Added `permit_reject[ID][data]` before `reject`**

   * lets the controller suppress forbidden rejection (`reject[2][1]`).

3. **Added fallback `process[ID] -> REQUEST` from `NG` and `GET_DATA`**

   * prevents deadlock when transfer/reject is denied.

These are minimal environment-side repairs that preserve the original safety properties while restoring controller synthesizability.

# Iteration2

A remaining source of unrealizability in the provided repaired model is that both `request[1]` and `request[2]` are still uncontrollable and independent. The requirements

* `P_NOT_OTHER_REQUEST_01`
* `P_NOT_OTHER_REQUEST_02`

require mutual exclusion between requests: after one user issues a request, the other user must not issue a request until the first session terminates (`process` or `reject`).

Because `request` is an environment action, the controller cannot prevent the second system from requesting while the first is active. Therefore, the environment itself must enforce request serialization.

A minimal repair is to introduce a shared synchronization process that grants permission for a request and releases it only when the corresponding session finishes. This preserves all safety properties and keeps the original workflow intact.

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
    ok[SystemID][DataID],
    ng[SystemID][DataID],
    reject[SystemID][DataID],
    dataTransfer[SystemID][DataID],
    filter[SystemID],
    unfilter[SystemID],
    process[SystemID],
    permit_transfer[SystemID][DataID],
    permit_reject[SystemID][DataID]
}

set FinishActions = {process[SystemID], reject[SystemID][DataID]}

/***************************************************************************
 * Request serialization monitor
 * Ensures that only one request can be active at a time.
 ***************************************************************************/

REQUEST_MUTEX =
    FREE,

FREE =
    (request[1] -> BUSY1
     | request[2] -> BUSY2),

BUSY1 =
    (process[1] -> FREE
      | reject[1][1] -> FREE
      | reject[1][2] -> FREE
      | reject[1][3] -> FREE
      | reject[1][4] -> FREE
      | reject[1][5] -> FREE),

BUSY2 =
    (process[2] -> FREE
      | reject[2][1] -> FREE
      | reject[2][2] -> FREE
      | reject[2][3] -> FREE
      | reject[2][4] -> FREE
      | reject[2][5] -> FREE).


/* Environment Transitions */

System(ID = 1) = REQUEST,

REQUEST =
    (request[ID] -> DATA_SELECTION),

DATA_SELECTION =
    (select_data[ID][data:DataID] -> JUDGEMENT[data]),

JUDGEMENT[data:DataID] =
    (ok[ID][data] -> OK[data]
     | ng[ID][data] -> NG[data]),

/* permission before rejection */
NG[data:DataID] =
    (permit_reject[ID][data] -> reject[ID][data] -> REQUEST
     | process[ID] -> REQUEST),

OK[data:DataID] =
    FILTERING[data],

FILTERING[data:DataID] =
    (filter[ID] -> GET_DATA[data]
     | unfilter[ID] -> GET_DATA[data]),

/* permission before transfer */
GET_DATA[data:DataID] =
    (permit_transfer[ID][data] -> dataTransfer[ID][data] -> USE_DATA
     | process[ID] -> REQUEST),

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

||Environment =
(
    REQUEST_MUTEX
    || forall [id:SystemID] System(id)
).

controller ||TraditionalController = (Environment)~{Specification}.
```

### Changes made

1. **Retained the permission gates**

   * `permit_transfer[ID][DataID]` before `dataTransfer`
   * `permit_reject[ID][DataID]` before `reject`

   These allow the controller to suppress transfers and rejections that would violate:

   * `P_SECURITY_OK_02`
   * `P_SECURITY_NG_02`
   * `P_SECURITY_Admin`

2. **Added `REQUEST_MUTEX`**

   * Synchronizes on existing actions (`request`, `process`, `reject`).
   * Prevents a second request from occurring while another request is active.
   * Directly enforces the assumptions required by `P_NOT_OTHER_REQUEST_01` and `P_NOT_OTHER_REQUEST_02`.

3. **Preserved all original safety properties**

   * No monitor was weakened or modified.
   * Existing process structure and action names were preserved.
   * Only the minimal synchronization necessary for realizability was added.
