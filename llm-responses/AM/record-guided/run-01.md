The Additional Control Record consistently indicates that the uncontrollable `request[i]` actions need an added **permission/authorization point** before they occur.

A minimal and structurally clean repair is therefore to insert a new controllable action immediately before each `request[ID]` in `REQUEST`, synchronized with the controller. Since the record shows that `request.1` and `request.2` must be mediated from many different predecessor contexts (e.g., `process`, `reject`, `ok/ng`, `filter/unfilter`, `dataTransfer`, `select_data`), placing the permission point directly before `request[ID]` is the least invasive change that covers *all* those contexts without altering existing uncontrollable semantics.

I introduce two new controllable actions:

* `permitRequest[1]`
* `permitRequest[2]`

so that:

`permitRequest[ID] -> request[ID]`

This preserves all existing safety properties and only adds the needed control point.

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
set AddedControlActions = {permitRequest[SystemID]}

set ControllableActions = {
    select_data[SystemID][DataID],
    ok[SystemID][DataID],
    ng[SystemID][DataID],
    reject[SystemID][DataID],
    dataTransfer[SystemID][DataID],
    filter[SystemID],
    unfilter[SystemID],
    process[SystemID],
    permitRequest[SystemID]
}

set FinishActions = {process[SystemID], reject[SystemID][DataID]}

/* Environment Transitions */
System(ID = 1) = REQUEST,

/* repaired: insert controllable permission before uncontrollable request */
REQUEST                = (permitRequest[ID] -> request[ID] -> DATA_SELECTION),

DATA_SELECTION         = (select_data[ID][data:DataID] -> JUDGEMENT[data]),

JUDGEMENT[data:DataID] =
                        (ok[ID][data] -> OK[data]
                        | ng[ID][data] -> NG[data]),

NG[data:DataID]        = (reject[ID][data] -> REQUEST),

OK[data:DataID]        = FILTERING[data],

FILTERING[data:DataID] =
                        (filter[ID]   -> GET_DATA[data]
                        | unfilter[ID] -> GET_DATA[data]),

GET_DATA[data:DataID]  = (dataTransfer[ID][data] -> USE_DATA),

USE_DATA               = (process[ID] -> REQUEST).


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

### Summary of changes

* Added new controllable actions: `permitRequest[SystemID]`.
* Inserted them immediately before every uncontrollable `request[ID]`.
* No safety property or existing branch structure was changed.
* This realizes the Additional Control Record by turning every `request` into a controller-mediated permission point.
