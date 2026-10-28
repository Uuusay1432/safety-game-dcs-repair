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

REQUEST = (
    permit_request[ID] -> request[ID] -> DATA_SELECTION
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

||Environment = (
    forall [id:SystemID] System(id)
).

controller ||TraditionalController =
    (Environment)~{Specification}.


Explanation of Changes
The repair introduces a new controllable action, permit_request[SystemID], immediately before each uncontrollable request[ID] action.
1. Additional controllable synchronization point
The original transition was:
REQUEST = (request[ID] -> DATA_SELECTION),


The revised transition is:
REQUEST = (
    permit_request[ID] -> request[ID] -> DATA_SELECTION
),


This introduces a controllable permission point before each request without changing the meaning of the original uncontrollable action.
2. Coverage of the Additional Control Record
All 40 entries identify either request.1 or request.2 as the uncontrollable action requiring additional control.
The parameterized action permit_request[ID] covers both:
- permit_request[1] precedes request[1].
- permit_request[2] precedes request[2].
The new permission actions allow the controller to coordinate request initiation with the existing controllable actions, including select_data, ok, ng, reject, filter, unfilter, dataTransfer, and process.
3. Preservation of the original model
The revision preserves all original process names, parameters, data-selection branches, judgement branches, filtering behavior, and data-transfer behavior.
All five original safety requirements remain unchanged.
4. Synthesizability consideration
The repair supplies a controllable gate for every request transition identified in the Additional Control Record. Because the record's environment origins are states of the composed environment, the parameterized gate is a compact way to address all 40 entries without explicitly enumerating those states.
However, the record identifies state-specific synchronization opportunities, while this implementation uses a shared permission action for each system. It therefore does not encode separate synchronization relationships for each listed controllable action. Actual recovery of synthesizability must be verified by running the DCS synthesis procedure on the revised model.