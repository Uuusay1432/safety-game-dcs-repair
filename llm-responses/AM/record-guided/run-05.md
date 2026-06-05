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

/*
 * request is no longer directly uncontrollable.
 * It is mediated by newly inserted controllable grant_request actions.
 */
set RequestActions = {request[SystemID]}

/*
 * Added controllable action: grant_request[SystemID]
 * This acts as the permission/synchronization point before each request[ID].
 */
set ControllableActions = {
    grant_request[SystemID],
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
 * Inserted controllable gate before request[ID]
 */
REQUEST                = (grant_request[ID] -> request[ID] -> DATA_SELECTION),

DATA_SELECTION         = (select_data[ID][data:DataID] -> JUDGEMENT[data]),

JUDGEMENT[data:DataID] =
                        (ok[ID][data] -> OK[data]
                        |ng[ID][data] -> NG[data]),

NG[data:DataID]        = (reject[ID][data] -> REQUEST),

OK[data:DataID]        = FILTERING[data],

FILTERING[data:DataID] =
                        (filter[ID]   -> GET_DATA[data]
                        |unfilter[ID] -> GET_DATA[data]),

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

* Added a **new controllable action** `grant_request[SystemID]`.

* Modified only the `REQUEST` state:

  ```fsp
  request[ID] -> ...
  ```

  became

  ```fsp
  grant_request[ID] -> request[ID] -> ...
  ```

* This realizes the Additional Control Record by introducing a **permission/synchronization point before every `request[i]`**, allowing the controller to coordinate `request` with the listed controllable actions (`process`, `reject`, `ok/ng`, `filter/unfilter`, `dataTransfer`, `select_data`) without changing the meaning of `request`.

* No safety properties were weakened or altered.

* All original process structure and existing actions were preserved.
