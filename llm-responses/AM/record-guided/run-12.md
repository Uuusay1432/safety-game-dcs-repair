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

/* Environment Transitions */

System(ID = 1) = REQUEST,

REQUEST = (
      permit_select[ID][data:DataID]
          -> request[ID] -> DATA_SELECTION

    | permit_ok[ID][data:DataID]
          -> request[ID] -> DATA_SELECTION

    | permit_ng[ID][data:DataID]
          -> request[ID] -> DATA_SELECTION

    | permit_reject[ID][data:DataID]
          -> request[ID] -> DATA_SELECTION

    | permit_transfer[ID][data:DataID]
          -> request[ID] -> DATA_SELECTION

    | permit_filter[ID]
          -> request[ID] -> DATA_SELECTION

    | permit_unfilter[ID]
          -> request[ID] -> DATA_SELECTION

    | permit_process[ID]
          -> request[ID] -> DATA_SELECTION
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
The revised model introduces controllable permission actions immediately before the uncontrollable request[ID] action.
The permissions correspond to the controllable actions identified in the Additional Control Record.
Additional permission	Corresponding actions
permit_select[ID][data]	select_data[ID][data]
permit_ok[ID][data]	ok[ID][data]
permit_ng[ID][data]	ng[ID][data]
permit_reject[ID][data]	reject[ID][data]
permit_transfer[ID][data]	dataTransfer[ID][data]
permit_filter[ID]	filter[ID]
permit_unfilter[ID]	unfilter[ID]
permit_process[ID]	process[ID]
The changes follow these principles:
1. Additional controllability: Every request[ID] transition is preceded by a controllable permission action, allowing the synthesized controller to regulate request initiation.
2. Preservation of uncontrollable behavior: The original request[ID] action remains uncontrollable. Its destination remains DATA_SELECTION.
3. Preservation of workflow: The original data selection, judgment, rejection, filtering, data transfer, and processing transitions remain unchanged.
4. Preservation of safety: All five original safety properties are retained without modification.
5. Parameterized implementation: Permission actions use the existing SystemID and DataID ranges, avoiding separate hardcoded transitions for each record entry.
Important limitation
The Additional Control Record identifies 40 distinct game-state-specific control points. However, the original parameterized FSP model does not expose the corresponding Q states as explicit environment states.
Consequently, this revision introduces permission actions at the shared REQUEST state rather than at each individual recorded origin.
Furthermore, the permission actions are currently alternative guards on requests; they do not enforce a persistent correspondence between a particular permission and its associated downstream controllable action.
Therefore, this is a structural repair candidate, not a verified synthesis-recovering implementation of all 40 records. The stated recovery guarantee applies to realizing the recorded control points with their intended synchronization semantics, which this abstraction does not fully establish.
The next step is to run controller synthesis on the revised model and, if necessary, refine the REQUEST transitions using explicit history-dependent synchronization states corresponding to the recorded game states.