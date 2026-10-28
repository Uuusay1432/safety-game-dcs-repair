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
    permit_request[SystemID]
}

set FinishActions = {
    process[SystemID],
    reject[SystemID][DataID]
}

/*
 * Environment Transitions
 *
 * Each uncontrollable request[ID] is preceded by
 * a controllable permit_request[ID] action.
 *
 * The permission action provides a synchronization
 * point between the requesting system and the
 * controllable actions of the other system.
 */

System(ID = 1) = REQUEST,

REQUEST =
    (permit_request[ID] ->
     request[ID] ->
     DATA_SELECTION),

DATA_SELECTION =
    (select_data[ID][data:DataID] ->
     JUDGEMENT[data]),

JUDGEMENT[data:DataID] =
    (ok[ID][data] -> OK[data]
    |ng[ID][data] -> NG[data]),

NG[data:DataID] =
    (reject[ID][data] -> REQUEST),

OK[data:DataID] =
    FILTERING[data],

FILTERING[data:DataID] =
    (filter[ID] -> GET_DATA[data]
    |unfilter[ID] -> GET_DATA[data]),

GET_DATA[data:DataID] =
    (dataTransfer[ID][data] -> USE_DATA),

USE_DATA =
    (process[ID] -> REQUEST).


//*****************************************************************************
// Additional Request Synchronization
//*****************************************************************************

/*
 * The additional permission points mediate requests
 * from the two systems.
 *
 * A new request is enabled only after the corresponding
 * controllable permission action.
 *
 * The controller can coordinate permit_request[1]
 * and permit_request[2] with the existing controllable
 * actions, including:
 *
 * select_data, ok, ng, reject, filter, unfilter,
 * dataTransfer, and process.
 *
 * These permissions cover the request transitions
 * identified by the Additional Control Record.
 */

REQUEST_PERMISSION =
    (permit_request[1] -> REQUEST_PERMISSION
    |permit_request[2] -> REQUEST_PERMISSION).


//*****************************************************************************
// Requirement
//*****************************************************************************

range Public_Data = 1 .. 1

/* FLTL */

ltl_property P_SECURITY_Admin =
    [](!reject[1][DataID])

ltl_property P_SECURITY_OK_02 =
    [](!{dataTransfer[2][DataID]}\
       {dataTransfer[2][Public_Data]})

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

||Environment = (
    forall [id:SystemID] System(id)
    || REQUEST_PERMISSION
).

controller ||TraditionalController =
    (Environment)~{Specification}.


Explanation of Changes
The repair introduces two new controllable actions, permit_request[1] and permit_request[2], immediately before the corresponding uncontrollable request[1] and request[2] actions.
The 40 entries in the Additional Control Record identify environment states in which requests can conflict with the other system's ongoing operations. The new permission actions provide control points at which the synthesized controller can coordinate requests with existing controllable actions such as select_data, ok, ng, reject, filter, unfilter, dataTransfer, and process.
The modification preserves the original processing sequence, branching structure, process names, and all five safety requirements. Neither the behavior nor the controllability classification of the original request actions is changed.
Limitation: This is a compact, parameterized repair rather than an exact state-specific realization of all 40 record entries. The additional permissions apply to every request transition, not only the identified states. The REQUEST_PERMISSION process also does not enforce individual synchronization dependencies with the listed actions; it merely allows the permission events to occur. Consequently, recovery of synthesizability has not been formally verified and must be checked with the DCS synthesis tool.