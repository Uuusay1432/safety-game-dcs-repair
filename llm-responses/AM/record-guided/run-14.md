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
 * Environment Transitions
 *
 * Additional controllable permission points are introduced
 * before request[ID].
 *
 * permit_request[1] mediates synchronization with:
 *   select_data[2][1..5]
 *   ok[2][1..5], ng[2][1..5]
 *   reject[2][1..5]
 *   dataTransfer[2][1..5]
 *   filter[2], unfilter[2]
 *   process[2]
 *
 * permit_request[2] mediates synchronization with:
 *   select_data[1][1..5]
 *   ok[1][1..5], ng[1][1..5]
 *   reject[1][1..5]
 *   dataTransfer[1][1..5]
 *   filter[1], unfilter[1]
 *   process[1]
 */

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
The repair introduces two new controllable actions, permit_request[1] and permit_request[2], immediately before the corresponding uncontrollable request actions.
The original transition:
REQUEST = (request[ID] -> DATA_SELECTION),


is replaced with:
REQUEST = (permit_request[ID] -> request[ID] -> DATA_SELECTION),


This provides a controllable permission point before either system issues a request.
The 40 entries in the Additional Control Record are grouped by their target uncontrollable actions:
- 24 entries for request.1: Addressed by permit_request[1], allowing the controller to coordinate requests from System 1 with the listed controllable actions of System 2.
- 16 entries for request.2: Addressed by permit_request[2], allowing the controller to coordinate requests from System 2 with the listed controllable actions of System 1.
The existing environment processes, data-selection behavior, judgment branches, filtering, data transfer, and processing behavior remain unchanged.
All five original safety requirements are preserved exactly.
Important synthesis qualification: This revision implements controllable permission points before both request actions, but it does not explicitly encode a separate synchronization protocol for each of the 40 recorded environment states. The original FSP model does not expose those expanded states directly. Consequently, although the modification follows the record's request-level guidance, controller synthesizability has not been independently verified and should be checked using the DCS synthesis tool.











ここまでの会話は役に立っていますか？