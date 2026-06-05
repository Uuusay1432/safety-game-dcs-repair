The Additional Control Record consistently indicates that the problematic uncontrollable actions are `request.1` and `request.2`, and that each should be *preceded by a new controllable permission point* synchronized with the currently enabled controllable actions.

The cleanest FSP-style repair is therefore to **insert a new controllable action `permitRequest[SystemID]` immediately before every `request[ID]`** in the environment. This preserves all existing behavior, does not alter the meaning of `request`, and gives the synthesized controller an explicit gate to delay/allow each request depending on whether the relevant controllable actions (e.g., `process`, `reject`, `ok/ng`, `filter/unfilter`, `dataTransfer`, `select_data`) have completed.

This directly realizes the intent of the Additional Control Record: add a controllable synchronization point before each uncontrollable request.

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

/* added permitRequest as new controllable gate before request */
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

REQUEST =
        (/*check[ID] -> */
         permitRequest[ID] -> request[ID] -> DATA_SELECTION),

DATA_SELECTION =
        (select_data[ID][data:DataID] -> JUDGEMENT[data]),

JUDGEMENT[data:DataID] =
        (ok[ID][data] -> OK[data]
        | ng[ID][data] -> NG[data]),

NG[data:DataID] =
        (reject[ID][data] -> REQUEST),

OK[data:DataID] =
        FILTERING[data],

FILTERING[data:DataID] =
        (filter[ID]   -> GET_DATA[data]
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

controller ||TraditionalController =
    (Environment)~{Specification}.
```

### Concise explanation of changes

* Added new controllable action: `permitRequest[SystemID]`.
* Inserted it directly before each uncontrollable `request[ID]`:

  * `permitRequest[ID] -> request[ID] -> ...`
* Added `permitRequest` to `ControllableActions`.
* Left all safety properties and all existing behavior unchanged.

This is the minimal structural repair that introduces the required additional control point before `request.1` and `request.2`, enabling DCS to synthesize a controller.
