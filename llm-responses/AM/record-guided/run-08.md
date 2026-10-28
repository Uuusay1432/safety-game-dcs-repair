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

/*
 * Additional controllable synchronization points.
 *
 * permit_request[1] mediates request[1] against
 * controllable actions of System 2.
 *
 * permit_request[2] mediates request[2] against
 * controllable actions of System 1.
 */

/* Environment Transitions */

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
 * The permission actions synchronize with the opposite system's
 * controllable workflow.
 *
 * A request is permitted only when the other system is in a
 * state where a new request will not violate the mutual-exclusion
 * safety requirements.
 *
 * The permission is released after process or reject.
 */

REQUEST_COORDINATOR =
    COORD_IDLE,

COORD_IDLE =
    (permit_request[1] -> COORD_1
    |permit_request[2] -> COORD_2),

COORD_1 =
    (process[1] -> COORD_IDLE
    |reject[1][data:DataID] -> COORD_IDLE),

COORD_2 =
    (process[2] -> COORD_IDLE
    |reject[2][data:DataID] -> COORD_IDLE).


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

    //liveness = {GOAL}

    controllable = {ControllableActions}
}


//*****************************************************************************
// Controller
//*****************************************************************************

/*
 * Compose both systems with the request coordinator.
 */

||Environment = (
    forall [id:SystemID] System(id)
    || REQUEST_COORDINATOR
).

// General Synthesis
controller ||TraditionalController =
    (Environment)~{Specification}.


Explanation of Changes
The repair introduces two new controllable actions, permit_request[1] and permit_request[2], immediately before the corresponding uncontrollable request[ID] actions.
1. Additional control points
All 40 entries in the Additional Control Record concern either request.1 or request.2. Therefore, the repair uses parameterized permission actions rather than introducing 40 separate action names.
The original transition:
REQUEST = (request[ID] -> DATA_SELECTION)


becomes:
REQUEST =
    (permit_request[ID] ->
     request[ID] ->
     DATA_SELECTION)


2. Synchronization between systems
The new REQUEST_COORDINATOR process provides a shared permission mechanism.
- permit_request[1] reserves the request phase for System 1.
- permit_request[2] reserves the request phase for System 2.
- The coordinator returns to its idle state after the active system executes process or reject.
This provides a controllable synchronization point before either uncontrollable request.
3. Preservation of the original model
The original processes REQUEST, DATA_SELECTION, JUDGEMENT, NG, OK, FILTERING, GET_DATA, and USE_DATA retain their workflow structure.
All five original safety requirements remain unchanged. The original uncontrollable request actions also retain their meaning.
4. Relationship to the Additional Control Record
The record identifies 40 environment transitions at which request permissions are needed. The parameterized permission actions cover both request types and allow the controller to coordinate request initiation with the other system's controllable workflow.
Verification limitation: This is a compact, generalized repair rather than a state-by-state implementation of all 40 recorded synchronization relationships. The coordinator does not explicitly synchronize each listed Available ControllableActions action with its corresponding permission point. In addition, the coordinator's release transitions synchronize with the existing completion actions, so its behavior should be checked for unintended blocking. Controller synthesizability has not been independently verified by running a DCS synthesis tool.