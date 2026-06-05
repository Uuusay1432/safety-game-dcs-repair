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
 Added controllable permission actions before each request:
   permit_request[1] synchronizes with user-1-side controllable choices
   permit_request[2] synchronizes with user-2-side controllable choices
*/
set ControllableActions = {
    select_data[SystemID][DataID],
    ok[SystemID][DataID],
    ng[SystemID][DataID],
    reject[SystemID][DataID],
    dataTransfer[SystemID][DataID],
    filter[SystemID],
    unfilter[SystemID],
    process[SystemID],
    permit_request[SystemID]
}

set FinishActions = {process[SystemID], reject[SystemID][DataID]}

/* Environment Transitions */
System(ID = 1) = REQUEST,

/*
 Inserted controllable gate before each uncontrollable request:
   permit_request[ID] -> request[ID]
*/
REQUEST                 = (permit_request[ID] -> request[ID] -> DATA_SELECTION),

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
ltl_property P_SECURITY_Admin   = [](!reject[1][DataID])
ltl_property P_SECURITY_OK_02   = [](!{dataTransfer[2][DataID]}\{dataTransfer[2][Public_Data]})
ltl_property P_SECURITY_NG_02   = [](!reject[2][Public_Data])

ltl_property P_NOT_OTHER_REQUEST_01 =
    [](request[1] ->
        X(!request[2] W
          (process[1] || reject[1][1] || reject[1][2] ||
           reject[1][3] || reject[1][4] || reject[1][5])))

ltl_property P_NOT_OTHER_REQUEST_02 =
    [](request[2] ->
        X(!request[1] W
          (process[2] || reject[2][1] || reject[2][2] ||
           reject[2][3] || reject[2][4] || reject[2][5])))

controllerSpec Specification =
{
    safety = {
        P_SECURITY_Admin,
        P_SECURITY_OK_02,
        P_SECURITY_NG_02,
        P_NOT_OTHER_REQUEST_01,
        P_NOT_OTHER_REQUEST_02
    }

    // liveness = {GOAL}

    controllable = {ControllableActions}
}


//*****************************************************************************
// Controller
//*****************************************************************************

//General Synthesis
||Environment = (forall [id:SystemID] System(id)).
controller ||TraditionalController = (Environment)~{Specification}.
```

### Explanation of changes

* Added a **new controllable action** `permit_request[SystemID]`.
* Inserted it immediately before each uncontrollable `request[ID]` transition:

  * `permit_request[ID] -> request[ID] -> ...`
* This directly realizes the Additional Control Record’s recommendation: introduce a **permission/synchronization point before `request.1` and `request.2`** so the controller can decide when a request may occur, based on existing controllable choices (`ok`, `ng`, `reject`, `filter`, `unfilter`, `dataTransfer`, `process`, `select_data`).
* No safety properties or existing uncontrollable semantics were changed. The repair is minimal and preserves the original workflow structure.
