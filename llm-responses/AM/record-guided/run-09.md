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
 Added controllable permission points:
 - permit_request[ID] before request[ID]
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
 Inserted controllable gate before each previously uncontrollable request:
 request[ID] becomes:
     permit_request[ID] -> request[ID]
*/
REQUEST                = (permit_request[ID] -> request[ID] -> DATA_SELECTION),

DATA_SELECTION         = (select_data[ID][data:DataID] -> JUDGEMENT[data]),

JUDGEMENT[data:DataID] =
                        (ok[ID][data] -> OK[data]
                        | ng[ID][data] -> NG[data]),

NG[data:DataID]        = (reject[ID][data] -> REQUEST),

OK[data:DataID]        = FILTERING[data],

FILTERING[data:DataID] =
                        (filter[ID] -> GET_DATA[data]
                        | unfilter[ID] -> GET_DATA[data]),

GET_DATA[data:DataID]  = (dataTransfer[ID][data] -> USE_DATA),

USE_DATA               = (process[ID] -> REQUEST).


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
          (process[1] ||
           reject[1][1] ||
           reject[1][2] ||
           reject[1][3] ||
           reject[1][4] ||
           reject[1][5])))

ltl_property P_NOT_OTHER_REQUEST_02 =
    [](request[2] ->
        X(!request[1] W
          (process[2] ||
           reject[2][1] ||
           reject[2][2] ||
           reject[2][3] ||
           reject[2][4] ||
           reject[2][5])))

controllerSpec Specification =
{
    safety = {
        P_SECURITY_Admin,
        P_SECURITY_OK_02,
        P_SECURITY_NG_02,
        P_NOT_OTHER_REQUEST_01,
        P_NOT_OTHER_REQUEST_02
    }

    //liveness     = {GOAL}
    controllable = {ControllableActions}
}


//*****************************************************************************
// Controller
//*****************************************************************************

//General Synthesis
||Environment = (forall [id:SystemID] System(id)).
controller ||TraditionalController = (Environment)~{Specification}.
```

## Explanation of changes

The Additional Control Record shows that every unrealizable state involves an uncontrollable `request.1` or `request.2`, and that synthesis fails because the controller needs a chance to decide **before** a request occurs.

To realize this, I introduced exactly one new controllable action per user:

* `permit_request[1]`
* `permit_request[2]`

and changed:

* `request[ID] -> DATA_SELECTION`

to:

* `permit_request[ID] -> request[ID] -> DATA_SELECTION`

This preserves all original behavior and safety properties while adding the required control point before each `request`, allowing the controller to synchronize that permission with existing controllable choices (`process`, `reject`, `select_data`, `ok/ng`, `filter/unfilter`, `dataTransfer`) as indicated by the Additional Control Record. No existing uncontrollable action semantics were changed.
