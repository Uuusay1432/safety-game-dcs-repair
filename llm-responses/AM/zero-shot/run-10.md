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
 * Added controllable permission actions:
 *   grant_request[SystemID]
 *
 * These actions serialize request admission so that
 * simultaneous/interleaving requests cannot force violation
 * of P_NOT_OTHER_REQUEST_01 / P_NOT_OTHER_REQUEST_02.
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
 * Request execution is now synchronized with an explicit
 * controllable permission step.
 */
REQUEST =
    (grant_request[ID] -> request[ID] -> DATA_SELECTION),

DATA_SELECTION =
    (select_data[ID][data:DataID] -> JUDGEMENT[data]),

/*
 * Added explicit confirmation structure for user 2.
 * Since P_SECURITY_OK_02 forbids transfer of non-public data,
 * the environment must permit the controller to safely deny
 * those cases before uncontrollable progress occurs.
 */
JUDGEMENT[data:DataID] =
    (
        ok[ID][data] -> OK[data]
      | ng[ID][data] -> NG[data]
    ),

NG[data:DataID] =
    (reject[ID][data] -> REQUEST),

OK[data:DataID] =
    FILTERING[data],

FILTERING[data:DataID] =
    (
        filter[ID] -> GET_DATA[data]
      | unfilter[ID] -> GET_DATA[data]
    ),

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

controller ||TraditionalController =
    (Environment)~{Specification}.
```

### Explanation of the repair

The unrealizability is mainly caused by the interaction between:

* the mutual-exclusion request properties
  (`P_NOT_OTHER_REQUEST_01`, `P_NOT_OTHER_REQUEST_02`), and
* uncontrollable request generation in the environment.

Originally, each `System` could autonomously execute `request[ID]` from `REQUEST`, allowing one user to issue a request while another session was still active. Since `request[...]` is uncontrollable, the controller had no way to prevent violations of the “no other request until completion” constraints.

To recover synthesizability, the environment was repaired by introducing a controllable synchronization action:

* `grant_request[SystemID]`

A request can now occur only after the controller explicitly grants permission:

```fsp
REQUEST = (grant_request[ID] -> request[ID] -> DATA_SELECTION)
```

This preserves the original workflow and safety properties while giving the controller the ability to serialize request admission and avoid conflicting concurrent requests.

No safety properties were weakened or modified. The original action structure and processing pipeline were preserved with only a minimal synchronization mechanism added.
