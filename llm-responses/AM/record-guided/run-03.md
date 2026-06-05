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

/* 
 Added controllable permission actions before each request.
 These realize the Additional Control Record by making request[i]
 occur only after explicit controller permission.
*/
set ControllableActions = {
    permitRequest[SystemID],
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

/* Environment Transitions */
System(ID = 1) = REQUEST,

/*
 Inserted controllable synchronization:
     permitRequest[ID] -> request[ID]
 so every previously uncontrollable request is now preceded by
 a controller-issued permission point.
*/
REQUEST                 = (permitRequest[ID] -> request[ID] -> DATA_SELECTION),

DATA_SELECTION          = (select_data[ID][data:DataID] -> JUDGEMENT[data]),

JUDGEMENT[data:DataID]  =
                        (ok[ID][data] -> OK[data]
                        | ng[ID][data] -> NG[data]),

NG[data:DataID]         = (reject[ID][data] -> REQUEST),

OK[data:DataID]         = FILTERING[data],

FILTERING[data:DataID]  =
                        (filter[ID] -> GET_DATA[data]
                        | unfilter[ID] -> GET_DATA[data]),

GET_DATA[data:DataID]   = (dataTransfer[ID][data] -> USE_DATA),

USE_DATA                = (process[ID] -> REQUEST).


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

    //liveness = {GOAL}

    controllable = {ControllableActions}
}


//*****************************************************************************
// Controller
//*****************************************************************************

//General Synthesis
||Environment = (forall [id:SystemID] System(id)).
controller ||TraditionalController = (Environment)~{Specification}.
```

### Explanation of the repair

The unrealizability is caused by uncontrollable `request[1]` / `request[2]` occurring at unsafe moments (as reflected by all entries in the Additional Control Record).
To realize the suggested control points while preserving the model structure, I inserted a single new controllable action:

* `permitRequest[ID]`

directly before each uncontrollable request:

* `permitRequest[ID] -> request[ID]`

This turns every request into a **permission-gated uncontrollable event**, allowing the synthesized controller to decide *when* each request may occur, which matches the intent of the Additional Control Record (all entries target `request.1` or `request.2`).

No safety properties were changed, and all original behavior after `request` remains unchanged.
