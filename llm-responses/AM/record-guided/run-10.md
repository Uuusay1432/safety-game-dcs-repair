The Additional Control Record consistently indicates that the problematic uncontrollable actions are `request.1` and `request.2`, and that they should be preceded by a new **controllable permission/synchronization action**.

The cleanest repair is therefore to insert a new controllable action before every `request[ID]`, preserving all existing behavior while giving the controller a new decision point.

I introduce:

* `permit_request[SystemID]` (new controllable action)

and modify:

* `request[ID] -> DATA_SELECTION`

into

* `permit_request[ID] -> request[ID] -> DATA_SELECTION`

This realizes all Additional Control Record entries because the controller can now synchronize `permit_request[ID]` with whichever existing controllable action is relevant in each game state (`ok`, `ng`, `reject`, `dataTransfer`, `filter`, `unfilter`, `select_data`, `process`, etc.), without changing the semantics of `request`.

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

/* added controllable permission before request */
set ControllableActions = {
    permit_request[SystemID],
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

REQUEST
    = (permit_request[ID] -> request[ID] -> DATA_SELECTION),

DATA_SELECTION
    = (select_data[ID][data:DataID] -> JUDGEMENT[data]),

JUDGEMENT[data:DataID]
    = (ok[ID][data] -> OK[data]
      | ng[ID][data] -> NG[data]),

NG[data:DataID]
    = (reject[ID][data] -> REQUEST),

OK[data:DataID]
    = FILTERING[data],

FILTERING[data:DataID]
    = (filter[ID]   -> GET_DATA[data]
      | unfilter[ID] -> GET_DATA[data]),

GET_DATA[data:DataID]
    = (dataTransfer[ID][data] -> USE_DATA),

USE_DATA
    = (process[ID] -> REQUEST).


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
        X(!request[2]
          W (process[1]
           || reject[1][1]
           || reject[1][2]
           || reject[1][3]
           || reject[1][4]
           || reject[1][5])))

ltl_property P_NOT_OTHER_REQUEST_02 =
    [](request[2] ->
        X(!request[1]
          W (process[2]
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

### Summary of changes

* Added new controllable action: `permit_request[SystemID]`
* Inserted it immediately before every uncontrollable `request[ID]`
* Preserved all existing requirements and all original process structure
* No uncontrollable action semantics were changed; only a new control point was added.
