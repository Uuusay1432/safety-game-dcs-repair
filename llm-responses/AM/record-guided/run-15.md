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

    // Additional controllable request permissions
    permit_select_data[SystemID][DataID],
    permit_ok[SystemID][DataID],
    permit_ng[SystemID][DataID],
    permit_reject[SystemID][DataID],
    permit_dataTransfer[SystemID][DataID],
    permit_filter[SystemID],
    permit_unfilter[SystemID],
    permit_process[SystemID]
}

set FinishActions = {
    process[SystemID],
    reject[SystemID][DataID]
}

/* Environment Transitions */

System(ID = 1) = REQUEST,

REQUEST = (
    // Permission based on data selection
      permit_select_data[ID][data:DataID]
        -> REQUEST_READY

    // Permission based on judgement
    | permit_ok[ID][data:DataID]
        -> REQUEST_READY
    | permit_ng[ID][data:DataID]
        -> REQUEST_READY

    // Permission based on rejection
    | permit_reject[ID][data:DataID]
        -> REQUEST_READY

    // Permission based on data transfer
    | permit_dataTransfer[ID][data:DataID]
        -> REQUEST_READY

    // Permission based on filtering
    | permit_filter[ID]
        -> REQUEST_READY
    | permit_unfilter[ID]
        -> REQUEST_READY

    // Permission based on processing
    | permit_process[ID]
        -> REQUEST_READY
),

REQUEST_READY = (
    request[ID] -> DATA_SELECTION
),

DATA_SELECTION = (
    select_data[ID][data:DataID] -> JUDGEMENT[data]
),

JUDGEMENT[data:DataID] = (
      ok[ID][data] -> OK[data]
    | ng[ID][data] -> NG[data]
),

NG[data:DataID] = (
    reject[ID][data] -> REQUEST
),

OK[data:DataID] = FILTERING[data],

FILTERING[data:DataID] = (
      filter[ID] -> GET_DATA[data]
    | unfilter[ID] -> GET_DATA[data]
),

GET_DATA[data:DataID] = (
    dataTransfer[ID][data] -> USE_DATA
),

USE_DATA = (
    process[ID] -> REQUEST
).

//PrivacyFilter = ().


//*****************************************************************************
// Requirement
//*****************************************************************************

range Public_Data = 1 .. 1

/* FLTL */

ltl_property P_SECURITY_Admin =
    [](!reject[1][DataID])

ltl_property P_SECURITY_OK_02 =
    [](!{dataTransfer[2][DataID]}
       \{dataTransfer[2][Public_Data]})

ltl_property P_SECURITY_NG_02 =
    [](!reject[2][Public_Data])

ltl_property P_NOT_OTHER_REQUEST_01 =
    [](request[1] ->
        X(!request[2] W (
            process[1] ||
            reject[1][1] ||
            reject[1][2] ||
            reject[1][3] ||
            reject[1][4] ||
            reject[1][5]
        ))
    )

ltl_property P_NOT_OTHER_REQUEST_02 =
    [](request[2] ->
        X(!request[1] W (
            process[2] ||
            reject[2][1] ||
            reject[2][2] ||
            reject[2][3] ||
            reject[2][4] ||
            reject[2][5]
        ))
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

// General Synthesis

||Environment = (
    forall [id:SystemID] System(id)
).

controller ||TraditionalController =
    (Environment)~{Specification}.


Explanation of Changes
The revised model introduces controllable permission actions before the uncontrollable request[ID] actions.
The Additional Control Record contains 40 entries, all targeting either request.1 or request.2. These entries are grouped by their associated controllable actions.
Additional permission	Associated controllable actions
permit_select_data[ID][data]	select_data[ID][data]
permit_ok[ID][data]	ok[ID][data]
permit_ng[ID][data]	ng[ID][data]
permit_reject[ID][data]	reject[ID][data]
permit_dataTransfer[ID][data]	dataTransfer[ID][data]
permit_filter[ID]	filter[ID]
permit_unfilter[ID]	unfilter[ID]
permit_process[ID]	process[ID]
The original transition
REQUEST = (request[ID] -> DATA_SELECTION)


is replaced with a permission-controlled transition through REQUEST_READY.
This introduces a controllable decision before either user's request can occur, while preserving the existing uncontrollable request actions.
The remaining environment stages—data selection, judgement, rejection, filtering, data transfer, and processing—retain their original structure.
All five original safety properties are preserved without modification.
Important limitation: The repair implements controllable gates corresponding to the action categories in the Additional Control Record, but it does not encode each recorded global state or enforce synchronization with the specific listed existing actions. Because the original parameterized FSP model does not identify the recorded Q states individually, this is an abstraction of the requested state-specific repair.
Consequently, the model is a candidate repair, not a verified realization of all 40 state-specific control points. Controller synthesizability must be checked using the DCS synthesis tool; it cannot be guaranteed from this transformation alone.