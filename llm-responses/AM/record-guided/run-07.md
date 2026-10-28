Revised DCS Model
//*****************************************************************************
// Environment
//*****************************************************************************

/* Set of Object */
const N = 2 /* num of user */
const K = 5 /* num of data (virtual server) */

range SystemID = 1 .. N
range DataID = 1 .. K

/* Set of Actions */
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

// PrivacyFilter = ().


//*****************************************************************************
// Requirement
//*****************************************************************************

range Public_Data = 1 .. 1

/* FLTL */

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

    // liveness = {GOAL}

    controllable = {ControllableActions}
}


//*****************************************************************************
// Controller
//*****************************************************************************

// General Synthesis

||Environment =
    (forall [id:SystemID] System(id)).

controller ||TraditionalController =
    (Environment)~{Specification}.


Explanation of Changes
The repair introduces a controllable permission action, permit_request[ID], immediately before each uncontrollable request[ID].
The original transition:
REQUEST = (request[ID] -> DATA_SELECTION),


becomes:
REQUEST =
    (permit_request[ID] -> request[ID] -> DATA_SELECTION),


This provides a control point for both users without changing the meaning of the original request events.
The Additional Control Record identifies 40 game states requiring additional control over request.1 or request.2. Because these states originate from the same parameterized REQUEST transition, the new permission action covers the corresponding request transitions for both systems.
The controller can use permit_request[1] and permit_request[2] to coordinate requests with the existing controllable actions, including data selection, judgment, filtering, data transfer, rejection, and processing.
All five original safety requirements remain unchanged. The original process names, parameters, data-selection branches, judgment branches, and subsequent processing transitions are also preserved.
Verification note: This is a compact, structurally conservative repair guided by the record, rather than a state-specific implementation of all 40 synchronization relationships. The permission action introduces a controllable gate, but it does not explicitly distinguish the listed controllable-action contexts. Therefore, controller synthesizability is not independently established by this revision.
The next step is to run synthesis on the revised model and, if necessary, refine the permission gate into context-specific synchronization points corresponding to the 40 recorded states.