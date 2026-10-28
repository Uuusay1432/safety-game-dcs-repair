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
 * Added controllable grant actions.
 * These actions serialize request admission and allow the controller
 * to prevent simultaneous request starts that violate the mutual-exclusion
 * safety properties.
 */
set ControllableActions = {
    grant[SystemID],
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

/*
 * Request issuance is now synchronized with a controllable permission
 * action grant[ID]. The environment cannot autonomously perform request[ID]
 * before the controller explicitly allows it.
 */

System(ID = 1) = IDLE,

IDLE                    =
                            (grant[ID] -> REQUEST),

REQUEST                 =
                            (request[ID] -> DATA_SELECTION),

DATA_SELECTION          =
                            (select_data[ID][data:DataID] -> JUDGEMENT[data]),

JUDGEMENT[data:DataID]  =
                            (
                                ok[ID][data] -> OK[data]
                              | ng[ID][data] -> NG[data]
                            ),

NG[data:DataID]         =
                            (reject[ID][data] -> IDLE),

OK[data:DataID]         =
                            FILTERING[data],

FILTERING[data:DataID]  =
                            (
                                filter[ID]   -> GET_DATA[data]
                              | unfilter[ID] -> GET_DATA[data]
                            ),

GET_DATA[data:DataID]   =
                            (dataTransfer[ID][data] -> USE_DATA),

USE_DATA                =
                            (process[ID] -> IDLE).


//*****************************************************************************
// Requirement
//*****************************************************************************

range Public_Data = 1 .. 1

/*FLTL*/
ltl_property P_SECURITY_Admin   = [](!reject[1][DataID])

ltl_property P_SECURITY_OK_02 =
    [](!{dataTransfer[2][DataID]}\{dataTransfer[2][Public_Data]})

ltl_property P_SECURITY_NG_02 =
    [](!reject[2][Public_Data])

ltl_property P_NOT_OTHER_REQUEST_01 =
    [](
        request[1] ->
        X(
            !request[2]
            W
            (
                process[1]
             || reject[1][1]
             || reject[1][2]
             || reject[1][3]
             || reject[1][4]
             || reject[1][5]
            )
        )
    )

ltl_property P_NOT_OTHER_REQUEST_02 =
    [](
        request[2] ->
        X(
            !request[1]
            W
            (
                process[2]
             || reject[2][1]
             || reject[2][2]
             || reject[2][3]
             || reject[2][4]
             || reject[2][5]
            )
        )
    )

controllerSpec Specification =
{
    safety       = {
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

### Explanation of the Repair

The unrealizability mainly comes from the two mutual-exclusion safety properties:

* `P_NOT_OTHER_REQUEST_01`
* `P_NOT_OTHER_REQUEST_02`

In the original model, `request[ID]` is uncontrollable because it is not included in `ControllableActions`. Therefore, after one user issues a request, the other user may independently issue another request immediately, violating the specification before the controller has any opportunity to intervene.

To recover synthesizability without weakening the requirements:

* A new controllable synchronization action `grant[ID]` was introduced.
* The environment must now execute:

  ```fsp
  grant[ID] -> request[ID]
  ```

  before a request is issued.

This creates an admission-control mechanism:

* the controller can permit only one active requester at a time,
* thereby enforcing the existing request-exclusion properties,
* while preserving all original safety monitors unchanged.

The rest of the protocol structure, actions, and branching behavior were preserved as closely as possible.