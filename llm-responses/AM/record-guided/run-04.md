Revised DCS Model
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

set FinishActions = {
    process[SystemID],
    reject[SystemID][DataID]
}

/* Environment Transitions */
System(ID = 1) = REQUEST,

REQUEST =
    (permit_request[ID] -> request[ID] -> DATA_SELECTION),

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
    (filter[ID] -> GET_DATA[data]
    | unfilter[ID] -> GET_DATA[data]),

GET_DATA[data:DataID] =
    (dataTransfer[ID][data] -> USE_DATA),

USE_DATA =
    (process[ID] -> REQUEST).

//PrivacyFilter = ().


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

    //liveness = {GOAL}
    controllable = {ControllableActions}
}


//*****************************************************************************
// Controller
//*****************************************************************************

//General Synthesis
||Environment = (forall [id:SystemID] System(id)).
controller ||TraditionalController = (Environment)~{Specification}.


Explanation of Changes
The Additional Control Record identifies 40 transitions involving request.1 or request.2 that require an additional controllable action.
The repair introduces a parameterized controllable action, permit_request[SystemID], immediately before each request:
REQUEST =
    (permit_request[ID] -> request[ID] -> DATA_SELECTION),


This provides the controller with a permission point before the previously uncontrollable request action. The controller can prevent a conflicting request while another system's request is being processed.
The change covers both request.1 and request.2 across the original environment process, including the control situations identified in the Additional Control Record.
All five original safety requirements remain unchanged. The existing process names, data-selection behavior, judgment branches, filtering, transfer, and processing actions are also preserved.
Verification note: This is a structural repair implementing the requested permission points. It does not explicitly encode each record entry's listed Available ControllableActions as a separate synchronization dependency. Therefore, although the added control points may allow a controller to enforce the mutual-exclusion requirements, recovery of synthesizability has not been formally verified. A synthesis run is required to confirm realizability.




