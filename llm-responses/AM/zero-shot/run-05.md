The unrealizability is caused by an **environment race** between the two `System` instances.

The requirements `P_NOT_OTHER_REQUEST_01` and `P_NOT_OTHER_REQUEST_02` require **mutual exclusion of requests**: once one user issues `request[i]`, the other user must not issue `request[j]` until the first user finishes (`process` or `reject`).

However, in the original environment both `System(1)` and `System(2)` independently start in `REQUEST`, so both uncontrollable `request[1]` and `request[2]` may occur immediately and concurrently. Since `request[...]` is uncontrollable, the controller cannot prevent this, making synthesis impossible.

The repair is therefore to add an **environment-side token/permission protocol** that serializes requests before they occur. This preserves all safety properties and only constrains the environment so that a second request cannot happen until the current session finishes.

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
set ControllableActions = {
    grant[SystemID],                 // newly added permission action
    select_data[SystemID][DataID],
    ok[SystemID][DataID],
    ng[SystemID][DataID],
    reject[SystemID][DataID],
    dataTransfer[SystemID][DataID],
    filter[SystemID],
    unfilter[SystemID],
    process[SystemID]
}
set FinishActions = {process[SystemID], reject[SystemID][DataID]}


/* -------------------------------------------------------------------------- */
/* Request arbiter: gives exclusive permission before request can occur       */
/* -------------------------------------------------------------------------- */
ARBITER =
    ( grant[1] -> BUSY[1]
    | grant[2] -> BUSY[2]
    ),

BUSY[id:SystemID] =
    ( process[id] -> ARBITER
    | reject[id][d:DataID] -> ARBITER
    ).


/* -------------------------------------------------------------------------- */
/* Environment Transitions                                                    */
/* -------------------------------------------------------------------------- */
System(ID = 1) = REQUEST,

REQUEST =
    ( grant[ID] -> request[ID] -> DATA_SELECTION ),

DATA_SELECTION =
    ( select_data[ID][data:DataID] -> JUDGEMENT[data] ),

JUDGEMENT[data:DataID] =
    ( ok[ID][data] -> OK[data]
    | ng[ID][data] -> NG[data]
    ),

NG[data:DataID] =
    ( reject[ID][data] -> REQUEST ),

OK[data:DataID] =
    FILTERING[data],

FILTERING[data:DataID] =
    ( filter[ID] -> GET_DATA[data]
    | unfilter[ID] -> GET_DATA[data]
    ),

GET_DATA[data:DataID] =
    ( dataTransfer[ID][data] -> USE_DATA ),

USE_DATA =
    ( process[ID] -> REQUEST ).


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
    ( ARBITER
    || forall [id:SystemID] System(id)
    ).

controller ||TraditionalController =
    (Environment)~{Specification}.
```

### Summary of changes

1. Added controllable action `grant[SystemID]`.
2. Inserted `grant[ID]` before each `request[ID]`, so requests only happen with permission.
3. Added `ARBITER` process to enforce **one active requester at a time**.
4. `ARBITER` releases permission only after `process[id]` or `reject[id][d]`, matching the original completion semantics.

This restores synthesizability while preserving all original safety monitors unchanged.
