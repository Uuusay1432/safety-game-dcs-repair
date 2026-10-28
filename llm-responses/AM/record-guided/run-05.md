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

    permit_select[SystemID][DataID],
    permit_ok[SystemID][DataID],
    permit_ng[SystemID][DataID],
    permit_reject[SystemID][DataID],
    permit_transfer[SystemID][DataID],
    permit_filter[SystemID],
    permit_unfilter[SystemID],
    permit_process[SystemID]
}

set FinishActions = {
    process[SystemID],
    reject[SystemID][DataID]
}

/*
 * Environment Transitions
 *
 * Each request is preceded by a controllable permission action.
 * Permission actions correspond to the controllable actions
 * identified in the Additional Control Record.
 *
 * The original request actions remain uncontrollable.
 * The original processing structure is preserved.
 */

System(ID = 1) = REQUEST,

REQUEST = (
      permit_select[ID][data:DataID] -> request[ID] -> DATA_SELECTION
    | permit_ok[ID][data:DataID] -> request[ID] -> DATA_SELECTION
    | permit_ng[ID][data:DataID] -> request[ID] -> DATA_SELECTION
    | permit_reject[ID][data:DataID] -> request[ID] -> DATA_SELECTION
    | permit_transfer[ID][data:DataID] -> request[ID] -> DATA_SELECTION
    | permit_filter[ID] -> request[ID] -> DATA_SELECTION
    | permit_unfilter[ID] -> request[ID] -> DATA_SELECTION
    | permit_process[ID] -> request[ID] -> DATA_SELECTION
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

||Environment =
    (forall [id:SystemID] System(id)).

controller ||TraditionalController =
    (Environment)~{Specification}.


Explanation of Changes
The revised model introduces controllable permission actions immediately before the original uncontrollable request[ID] actions.
The permission actions are organized according to the controllable actions identified in the Additional Control Record.
New controllable action	Related original actions
permit_select[ID][data]	select_data[ID][data]
permit_ok[ID][data]	ok[ID][data]
permit_ng[ID][data]	ng[ID][data]
permit_reject[ID][data]	reject[ID][data]
permit_transfer[ID][data]	dataTransfer[ID][data]
permit_filter[ID]	filter[ID]
permit_unfilter[ID]	unfilter[ID]
permit_process[ID]	process[ID]
The changes are intended to provide the controller with an opportunity to regulate request execution before the environment performs an uncontrollable request.
Preserved elements:
- All five original safety requirements remain unchanged.
- The original uncontrollable actions request[1] and request[2] are preserved.
- The original processing stages DATA_SELECTION, JUDGEMENT, NG, OK, FILTERING, GET_DATA, and USE_DATA remain unchanged.
- The original system parameters and data ranges are preserved.
Important limitation: The Additional Control Record identifies 40 specific transitions in the composed environment's state graph. The parameterized source model does not expose those individual states directly. Consequently, this implementation adds permission alternatives at the shared REQUEST state rather than implementing 40 state-specific synchronization points.
Although every request is now preceded by a controllable action, the new permission actions are not explicitly synchronized with the corresponding original controllable actions. Therefore, this is a structural repair candidate, not a verified realization of all 40 Additional Control Record entries. Controller synthesizability must be checked using the DCS synthesis tool; recovery cannot be guaranteed from this model alone.