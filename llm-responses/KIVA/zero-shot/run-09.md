You are an expert in Discrete Controller Synthesis (DCS) and FSP-style model repair.

A DCS model is given below. The current model is unrealizable, meaning that no controller satisfying the given safety requirements can be synthesized.

Your task is to revise the environment model so that controller synthesizability can be recovered.

# Repair policy

1. Identify possible causes of unrealizability from the given DCS model.

2. Modify the environment model by introducing synchronization, permission, confirmation, or suppression mechanisms where they appear necessary.

3. Preserve the original safety requirements. Do not weaken, remove, or rewrite the safety monitors to make synthesis easier.

4. Preserve the original model structure, process names, action names, parameters, and branching structure as much as possible.

5. Add new controllable actions only where they are needed to recover controller synthesizability.

6. Output the complete revised DCS model and a concise explanation of the changes.

# Original DCS Model


const PodNum   = 2
const RobotNum = 1

range PodID   = 1..PodNum
range RobotID = 1..RobotNum

//Set
set ShippingRobot  = {robot1}
set BringBoxRobot  = {robot1}
set ReplenishRobot = {robot2}
set BringItemRobot = {robot2}
set Robot = {robot1, robot2}
set Area  = {shippingArea, boxGettingArea, itemGettingArea, packingArea}

//Action Set
set MoveActions = {[Robot].move[PodID], [Robot].move[Area]}
set AreaActions = {ship, supplyBox, supplyItem, getItem, pack}

set PickActions = {[Robot].pick[PodID], [Robot].pick[Area]}
set PutActions = {[Robot].put[PodID], [Robot].put[Area]}
set ArmActions = {PickActions, PutActions}
set Robot1_PickActions = {['robot1].pick[PodID], ['robot1].pick[Area]}
set Robot1_PutActions = {['robot1].put[PodID], ['robot1].put[Area]}
set Robot1_ArmActions = {Robot1_PickActions, Robot1_PutActions}
set Robot2_PickActions = {['robot2].pick[PodID], ['robot2].pick[Area]}
set Robot2_PutActions = {['robot2].put[PodID], ['robot2].put[Area]}
set Robot2_ArmActions = {Robot2_PickActions, Robot2_PutActions}

set RequestActions = {reqReplenishItem, reqBringItem, reqBringBox, reqShip, replenish[PodID], bring[PodID]}
set DoneActions = {doneReplenishItem, doneBringItem, doneBringBox, doneShip}

set ShippingRobotActions = {['robot1].move[PodID], ['robot1].pick[PodID], ['robot1].put[PodID], ['robot1].move[Area], ['robot1].pick[Area], ['robot1].put[Area]}
set ReplenishRobotActions = {['robot2].move[PodID], ['robot2].pick[PodID], ['robot2].put[PodID], ['robot2].move[Area], ['robot2].pick[Area], ['robot2].put[Area]}

set ControllableActions = {/*MoveActions, */AreaActions, ArmActions, DoneActions, reqReplenishItem, reqBringItem, reqBringBox, reqShip}
set MonitorableActions  = {AreaActions}
set AllActions = {MoveActions, AreaActions, ArmActions, RequestActions, DoneActions}

//******************************************************************************************//
// Environment Model
//******************************************************************************************//
INITIAL(N = 1)    = MOVEMENT[N],
MOVEMENT[i:PodID] = (move[ii:PodID] -> MOVEMENT[ii]
					 |move['shippingArea]    -> SHIAREA
					 |move['boxGettingArea]  -> BOX_GET_AREA
					 |move['itemGettingArea] -> ITEM_GET_AREA
					 |move['packingArea]     -> PACK_AREA
		             |pick[i] -> MOVEMENT[i]
		             |put[i]  -> MOVEMENT[i]),
SHIAREA     = (move[ii:PodID] -> MOVEMENT[ii]
				|move['boxGettingArea]  -> BOX_GET_AREA
				|move['itemGettingArea] -> ITEM_GET_AREA
				|move['packingArea]     -> PACK_AREA
				|pick['shippingArea] -> SHIAREA
				|put['shippingArea] -> SHIAREA),
BOX_GET_AREA  = (move[ii:PodID] -> MOVEMENT[ii]
				|move['shippingArea]    -> SHIAREA
				|move['itemGettingArea] -> ITEM_GET_AREA
				|move['packingArea]     -> PACK_AREA
				|pick['boxGettingArea] -> BOX_GET_AREA
				|put['boxGettingArea] -> BOX_GET_AREA),
ITEM_GET_AREA = (move[ii:PodID] -> MOVEMENT[ii]
				|move['shippingArea]    -> SHIAREA
				|move['boxGettingArea]  -> BOX_GET_AREA
				|move['packingArea]     -> PACK_AREA
				|pick['itemGettingArea] -> ITEM_GET_AREA
				|put['itemGettingArea] -> ITEM_GET_AREA),
PACK_AREA     = (move[ii:PodID] -> MOVEMENT[ii]
				|move['shippingArea]    -> SHIAREA
				|move['boxGettingArea]  -> BOX_GET_AREA
				|move['itemGettingArea] -> ITEM_GET_AREA
				|pick['packingArea] -> PACK_AREA
				|put['packingArea] -> PACK_AREA).

SHIAREA     = (ship -> SHIAREA).
BOX_GET_AREA  = (supplyBox -> BOX_GET_AREA).
ITEM_GET_AREA = ({supplyItem, getItem} -> ITEM_GET_AREA).
PACK_AREA     = (pack -> PACK_AREA).

REQUEST_1 = (reqReplenishItem -> replenish[PodID]  -> doneReplenishItem -> REQUEST_1).
REQUEST_2 = (reqBringItem -> bring[PodID]  -> doneBringItem -> REQUEST_2).
REQUEST_3 = (reqBringBox  -> doneBringBox -> REQUEST_3).
REQUEST_4 = (reqShip      -> doneShip     -> REQUEST_4).

||KIVA_System = (['robot1]:INITIAL(1) || ['robot2]:INITIAL(PodNum) || SHIAREA || BOX_GET_AREA || ITEM_GET_AREA || PACK_AREA || REQUEST_1 || REQUEST_2 || REQUEST_3 || REQUEST_4).

//*****************************************************************************
// Requirement
//*****************************************************************************


/*Fluent*/
fluent REPLENISH[i:PodID] = <replenish[i], ['robot2].move[i]>
fluent BRING[i:PodID]     = <bring[i], ['robot2].move[i]>

fluent PROCESS_REQUEST    = <RequestActions, DoneActions>
fluent PROCESS_REPLENISH_ITEM = <reqReplenishItem, doneReplenishItem>
fluent PROCESS_BRING_ITEM = <reqBringItem, doneBringItem>
fluent PROCESS_BRING_BOX  = <reqBringBox, doneBringBox>
fluent PROCESS_SHIP       = <reqShip, doneShip>

fluent REQ_SHIP_TIMEING = <pack, doneShip>

fluent SHIP_TIMING   = <['robot1].put['shippingArea], doneShip>
fluent BOX_TIMING    = <['robot1].move['boxGettingArea], ['robot1].pick['boxGettingArea]>
fluent ITEM_TIMING   = <['robot2].put['itemGettingArea], ['robot2].pick['itemGettingArea]>
fluent GET_TIMING_1  = <['robot2].put['packingArea], ['robot2].pick['packingArea]>
fluent GET_TIMING_2  = <doneBringBox, ['robot2].put['packingArea]>
fluent PACK_TIMING_1 = <doneBringBox, reqShip>
fluent PACK_TIMING_2 = <doneBringItem, reqShip>

fluent RETURN[i:PodID]   = <['robot2].pick[i], ['robot2].move[i]>

/*FLTL*/
// Two Robot
ltl_property SHIPROBOT_RULE  = [](!{['robot1].move['itemGettingArea]})
ltl_property REPLENISH_ROBOT_RULE = [](!{['robot2].move['shippingArea], ['robot2].move['boxGettingArea]})

ltl_property REQUEST_TIMING_1 = [](reqBringBox  -> X(!reqShip      W doneBringBox))
ltl_property REQUEST_TIMING_2 = [](reqShip      -> X(!reqBringBox  W doneShip))
ltl_property REQUEST_TIMING_3 = [](reqReplenishItem -> X(!reqBringItem W doneReplenishItem))
ltl_property REQUEST_TIMING_4 = [](reqBringItem -> X(!reqReplenishItem W doneBringItem))

// Single Robot
ltl_property REQUEST_ONLY_1 = [](!(PROCESS_SHIP || PROCESS_BRING_BOX)        -> !{['robot1].move[PodID], ['robot1].move[Area], ['robot1].pick[PodID], ['robot1].put[PodID], ['robot1].pick[Area], ['robot1].put[Area]})
ltl_property REQUEST_ONLY_2 = [](!(PROCESS_REPLENISH_ITEM || PROCESS_BRING_ITEM) -> !{['robot2].move[PodID], ['robot2].move[Area], ['robot2].pick[PodID], ['robot2].put[PodID], ['robot2].pick[Area], ['robot2].put[Area]})

ltl_property ROBOT1_ARM_RULE_1 = [](!(PROCESS_SHIP || PROCESS_BRING_BOX) -> !{Robot1_ArmActions})
ltl_property ROBOT1_ARM_RULE_2 = []({Robot1_PickActions} -> X(!{Robot1_PickActions} W {Robot1_PutActions}))
ltl_property ROBOT1_ARM_RULE_3 = []({Robot1_PutActions}  -> X(!{Robot1_PutActions} W {Robot1_PickActions}))
ltl_property ROBOT2_ARM_RULE_1 = [](!(PROCESS_REPLENISH_ITEM || PROCESS_BRING_ITEM) -> !{Robot2_ArmActions})
ltl_property ROBOT2_ARM_RULE_2 = []({Robot2_PickActions} -> X(!{Robot2_PickActions} W {Robot2_PutActions}))
ltl_property ROBOT2_ARM_RULE_3 = []({Robot2_PutActions}  -> X(!{Robot2_PutActions} W {Robot2_PickActions}))

//ltl_property REPLENISH_RULE_1 = [](REPLENISH[PodID] -> X(!{replenish[PodID]} W ['robot2].move[PodID]) && !['robot2].move[PodID])
ltl_property REPLENISH_RULE_1 = [](REPLENISH[PodID] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE_2 = [](replenish[PodID] -> X(!{{ReplenishRobotActions}\{['robot2].move[PodID]}} W ['robot2].move[PodID]))
ltl_property REPLENISH_RULE01 = [](REPLENISH[1]  -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE02 = [](REPLENISH[2]  -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE03 = [](REPLENISH[3]  -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE04 = [](REPLENISH[4]  -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE05 = [](REPLENISH[5]  -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE06 = [](REPLENISH[6]  -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE07 = [](REPLENISH[7]  -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE08 = [](REPLENISH[8]  -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE09 = [](REPLENISH[9]  -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE10 = [](REPLENISH[10] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE11 = [](REPLENISH[11] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE12 = [](REPLENISH[12] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE13 = [](REPLENISH[13] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE14 = [](REPLENISH[14] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE15 = [](REPLENISH[15] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE16 = [](REPLENISH[16] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE17 = [](REPLENISH[17] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE18 = [](REPLENISH[18] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE19 = [](REPLENISH[19] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE20 = [](REPLENISH[20] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE21 = [](REPLENISH[21] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE22 = [](REPLENISH[22] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE23 = [](REPLENISH[23] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE24 = [](REPLENISH[24] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE25 = [](REPLENISH[25] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE26 = [](REPLENISH[26] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE27 = []
…
Revised DCS Model
The repair preserves the original safety properties and introduces controllable permission actions to prevent uncontrollable robot movements from violating request sequencing, robot-operation restrictions, and collision avoidance.

The main changes are:

Movement permissions: Each robot movement requires a preceding controllable authorization. The controller can therefore prevent unsafe movements without making the original move actions controllable.

Request synchronization: Robot movements are enabled only during the corresponding active request, and incompatible requests cannot overlap.

Collision prevention: The permission mechanism prevents both robots from occupying the same pod or area.

Movement confirmation: Each authorized movement is followed by its corresponding original movement action before another movement can be authorized.

Original safety requirements: All original fluent definitions, LTL properties, and entries in Specification.safety are retained.

The revised model below uses the original requirements unchanged. Synthesis success still needs to be checked using the target DCS tool; it is not claimed as experimentally verified.

const PodNum   = 2
const RobotNum = 1

range PodID   = 1..PodNum
range RobotID = 1..RobotNum

//Set
set ShippingRobot  = {robot1}
set BringBoxRobot  = {robot1}
set ReplenishRobot = {robot2}
set BringItemRobot = {robot2}
set Robot = {robot1, robot2}
set Area  = {shippingArea, boxGettingArea, itemGettingArea, packingArea}

//Action Set
set MoveActions = {[Robot].move[PodID], [Robot].move[Area]}
set AreaActions = {ship, supplyBox, supplyItem, getItem, pack}

set PickActions = {[Robot].pick[PodID], [Robot].pick[Area]}
set PutActions = {[Robot].put[PodID], [Robot].put[Area]}
set ArmActions = {PickActions, PutActions}
set Robot1_PickActions = {['robot1].pick[PodID], ['robot1].pick[Area]}
set Robot1_PutActions = {['robot1].put[PodID], ['robot1].put[Area]}
set Robot1_ArmActions = {Robot1_PickActions, Robot1_PutActions}
set Robot2_PickActions = {['robot2].pick[PodID], ['robot2].pick[Area]}
set Robot2_PutActions = {['robot2].put[PodID], ['robot2].put[Area]}
set Robot2_ArmActions = {Robot2_PickActions, Robot2_PutActions}

set RequestActions = {reqReplenishItem, reqBringItem, reqBringBox, reqShip, replenish[PodID], bring[PodID]}
set DoneActions = {doneReplenishItem, doneBringItem, doneBringBox, doneShip}

set ShippingRobotActions = {['robot1].move[PodID], ['robot1].pick[PodID], ['robot1].put[PodID], ['robot1].move[Area], ['robot1].pick[Area], ['robot1].put[Area]}
set ReplenishRobotActions = {['robot2].move[PodID], ['robot2].pick[PodID], ['robot2].put[PodID], ['robot2].move[Area], ['robot2].pick[Area], ['robot2].put[Area]}

// Repair: controllable movement authorization.
set PermissionActions = {
    [Robot].permitMove[PodID],
    [Robot].permitMove[Area]
}

set ControllableActions = {
    AreaActions,
    ArmActions,
    DoneActions,
    reqReplenishItem,
    reqBringItem,
    reqBringBox,
    reqShip,
    PermissionActions
}

set MonitorableActions = {AreaActions}
set AllActions = {MoveActions, AreaActions, ArmActions,
                  RequestActions, DoneActions, PermissionActions}

//*****************************************************************************
// Environment Model
//*****************************************************************************

// Repair: each uncontrollable movement requires authorization.
// The original movement, pick, and put actions are preserved.

INITIAL(N = 1) = MOVEMENT[N],

MOVEMENT[i:PodID] =
    (permitMove[ii:PodID] -> move[ii] -> MOVEMENT[ii]
    |permitMove['shippingArea] ->
        move['shippingArea] -> SHIAREA
    |permitMove['boxGettingArea] ->
        move['boxGettingArea] -> BOX_GET_AREA
    |permitMove['itemGettingArea] ->
        move['itemGettingArea] -> ITEM_GET_AREA
    |permitMove['packingArea] ->
        move['packingArea] -> PACK_AREA
    |pick[i] -> MOVEMENT[i]
    |put[i] -> MOVEMENT[i]),

SHIAREA =
    (permitMove[ii:PodID] -> move[ii] -> MOVEMENT[ii]
    |permitMove['boxGettingArea] ->
        move['boxGettingArea] -> BOX_GET_AREA
    |permitMove['itemGettingArea] ->
        move['itemGettingArea] -> ITEM_GET_AREA
    |permitMove['packingArea] ->
        move['packingArea] -> PACK_AREA
    |pick['shippingArea] -> SHIAREA
    |put['shippingArea] -> SHIAREA),

BOX_GET_AREA =
    (permitMove[ii:PodID] -> move[ii] -> MOVEMENT[ii]
    |permitMove['shippingArea] ->
        move['shippingArea] -> SHIAREA
    |permitMove['itemGettingArea] ->
        move['itemGettingArea] -> ITEM_GET_AREA
    |permitMove['packingArea] ->
        move['packingArea] -> PACK_AREA
    |pick['boxGettingArea] -> BOX_GET_AREA
    |put['boxGettingArea] -> BOX_GET_AREA),

ITEM_GET_AREA =
    (permitMove[ii:PodID] -> move[ii] -> MOVEMENT[ii]
    |permitMove['shippingArea] ->
        move['shippingArea] -> SHIAREA
    |permitMove['boxGettingArea] ->
        move['boxGettingArea] -> BOX_GET_AREA
    |permitMove['packingArea] ->
        move['packingArea] -> PACK_AREA
    |pick['itemGettingArea] -> ITEM_GET_AREA
    |put['itemGettingArea] -> ITEM_GET_AREA),

PACK_AREA =
    (permitMove[ii:PodID] -> move[ii] -> MOVEMENT[ii]
    |permitMove['shippingArea] ->
        move['shippingArea] -> SHIAREA
    |permitMove['boxGettingArea] ->
        move['boxGettingArea] -> BOX_GET_AREA
    |permitMove['itemGettingArea] ->
        move['itemGettingArea] -> ITEM_GET_AREA
    |pick['packingArea] -> PACK_AREA
    |put['packingArea] -> PACK_AREA).

// Original area processes.
SHIAREA = (ship -> SHIAREA).
BOX_GET_AREA = (supplyBox -> BOX_GET_AREA).
ITEM_GET_AREA = ({supplyItem, getItem} -> ITEM_GET_AREA).
PACK_AREA = (pack -> PACK_AREA).

// Original request processes.
REQUEST_1 =
    (reqReplenishItem ->
     replenish[PodID] ->
     doneReplenishItem -> REQUEST_1).

REQUEST_2 =
    (reqBringItem ->
     bring[PodID] ->
     doneBringItem -> REQUEST_2).

REQUEST_3 =
    (reqBringBox ->
     doneBringBox -> REQUEST_3).

REQUEST_4 =
    (reqShip ->
     doneShip -> REQUEST_4).

// Repair: serialize requests sharing robot1.
R1_REQUEST_PERMISSION =
    (reqBringBox -> R1_BOX_BUSY
    |reqShip -> R1_SHIP_BUSY),

R1_BOX_BUSY =
    (doneBringBox -> R1_REQUEST_PERMISSION),

R1_SHIP_BUSY =
    (doneShip -> R1_REQUEST_PERMISSION).

// Repair: serialize requests sharing robot2.
R2_REQUEST_PERMISSION =
    (reqReplenishItem -> R2_REPLENISH_BUSY
    |reqBringItem -> R2_BRING_BUSY),

R2_REPLENISH_BUSY =
    (doneReplenishItem -> R2_REQUEST_PERMISSION),

R2_BRING_BUSY =
    (doneBringItem -> R2_REQUEST_PERMISSION).

// Repair: movement permissions are only available while
// the corresponding robot is processing a request.
R1_MOVEMENT_PERMISSION =
    (reqBringBox -> R1_MOVEMENT_ACTIVE_BOX
    |reqShip -> R1_MOVEMENT_ACTIVE_SHIP),

R1_MOVEMENT_ACTIVE_BOX =
    (['robot1].permitMove[PodID] ->
        R1_MOVEMENT_ACTIVE_BOX
    |['robot1].permitMove[Area] ->
        R1_MOVEMENT_ACTIVE_BOX
    |doneBringBox -> R1_MOVEMENT_PERMISSION),

R1_MOVEMENT_ACTIVE_SHIP =
    (['robot1].permitMove[PodID] ->
        R1_MOVEMENT_ACTIVE_SHIP
    |['robot1].permitMove[Area] ->
        R1_MOVEMENT_ACTIVE_SHIP
    |doneShip -> R1_MOVEMENT_PERMISSION).

R2_MOVEMENT_PERMISSION =
    (reqReplenishItem -> R2_MOVEMENT_ACTIVE_REPLENISH
    |reqBringItem -> R2_MOVEMENT_ACTIVE_BRING),

R2_MOVEMENT_ACTIVE_REPLENISH =
    (['robot2].permitMove[PodID] ->
        R2_MOVEMENT_ACTIVE_REPLENISH
    |['robot2].permitMove[Area] ->
        R2_MOVEMENT_ACTIVE_REPLENISH
    |doneReplenishItem -> R2_MOVEMENT_PERMISSION),

R2_MOVEMENT_ACTIVE_BRING =
    (['robot2].permitMove[PodID] ->
        R2_MOVEMENT_ACTIVE_BRING
    |['robot2].permitMove[Area] ->
        R2_MOVEMENT_ACTIVE_BRING
    |doneBringItem -> R2_MOVEMENT_PERMISSION).

// Repair: restrict uncontrollable pod-selection events
// to the current active request.
R2_POD_REQUEST_PERMISSION =
    (reqReplenishItem -> R2_WAIT_REPLENISH
    |reqBringItem -> R2_WAIT_BRING),

R2_WAIT_REPLENISH =
    (replenish[p:PodID] -> R2_SELECTED_REPLENISH[p]),

R2_SELECTED_REPLENISH[p:PodID] =
    (doneReplenishItem -> R2_POD_REQUEST_PERMISSION),

R2_WAIT_BRING =
    (bring[p:PodID] -> R2_SELECTED_BRING[p]),

R2_SELECTED_BRING[p:PodID] =
    (doneBringItem -> R2_POD_REQUEST_PERMISSION).

// Repair: collision-aware location authorization.
// The initial locations are pod1 and pod2.
// Authorization of a destination is allowed only when
// the other robot is not occupying that destination.

// Location indexing for collision arbitration.
const LocationNum = PodNum + 4
range LocationID = 1..LocationNum

// 1..PodNum: pods
// PodNum+1: shippingArea
// PodNum+2: boxGettingArea
// PodNum+3: itemGettingArea
// PodNum+4: packingArea

COLLISION_PERMISSION =
    COLLISION_STATE[1][PodNum],

COLLISION_STATE[r1:LocationID][r2:LocationID] =
    (when (1 != r2)
        ['robot1].permitMove[1] ->
            ['robot1].move[1] ->
                COLLISION_STATE[1][r2]
    |when (2 != r2)
        ['robot1].permitMove[2] ->
            ['robot1].move[2] ->
                COLLISION_STATE[2][r2]

    |when (1 != r1)
        ['robot2].permitMove[1] ->
            ['robot2].move[1] ->
                COLLISION_STATE[r1][1]
    |when (2 != r1)
        ['robot2].permitMove[2] ->
            ['robot2].move[2] ->
                COLLISION_STATE[r1][2]

    |when (PodNum+1 != r2)
        ['robot1].permitMove['shippingArea] ->
            ['robot1].move['shippingArea] ->
                COLLISION_STATE[PodNum+1][r2]

    |when (PodNum+2 != r2)
        ['robot1].permitMove['boxGettingArea] ->
            ['robot1].move['boxGettingArea] ->
                COLLISION_STATE[PodNum+2][r2]

    |when (PodNum+3 != r2)
        ['robot1].permitMove['itemGettingArea] ->
            ['robot1].move['itemGettingArea] ->
                COLLISION_STATE[PodNum+3][r2]

    |when (PodNum+4 != r2)
        ['robot1].permitMove['packingArea] ->
            ['robot1].move['packingArea] ->
                COLLISION_STATE[PodNum+4][r2]

    |when (PodNum+1 != r1)
        ['robot2].permitMove['shippingArea] ->
            ['robot2].move['shippingArea] ->
                COLLISION_STATE[r1][PodNum+1]

    |when (PodNum+2 != r1)
        ['robot2].permitMove['boxGettingArea] ->
            ['robot2].move['boxGettingArea] ->
                COLLISION_STATE[r1][PodNum+2]

    |when (PodNum+3 != r1)
        ['robot2].permitMove['itemGettingArea] ->
            ['robot2].move['itemGettingArea] ->
                COLLISION_STATE[r1][PodNum+3]

    |when (PodNum+4 != r1)
        ['robot2].permitMove['packingArea] ->
            ['robot2].move['packingArea] ->
                COLLISION_STATE[r1][PodNum+4]).

// Revised system composition.
||KIVA_System =
    (['robot1]:INITIAL(1)
    || ['robot2]:INITIAL(PodNum)
    || SHIAREA
    || BOX_GET_AREA
    || ITEM_GET_AREA
    || PACK_AREA
    || REQUEST_1
    || REQUEST_2
    || REQUEST_3
    || REQUEST_4
    || R1_REQUEST_PERMISSION
    || R2_REQUEST_PERMISSION
    || R1_MOVEMENT_PERMISSION
    || R2_MOVEMENT_PERMISSION
    || R2_POD_REQUEST_PERMISSION
    || COLLISION_PERMISSION).

//*****************************************************************************
// Requirement
//*****************************************************************************

/*Fluent*/
fluent REPLENISH[i:PodID] = <replenish[i], ['robot2].move[i]>
fluent BRING[i:PodID] = <bring[i], ['robot2].move[i]>

fluent PROCESS_REQUEST = <RequestActions, DoneActions>
fluent PROCESS_REPLENISH_ITEM = <reqReplenishItem, doneReplenishItem>
fluent PROCESS_BRING_ITEM = <reqBringItem, doneBringItem>
fluent PROCESS_BRING_BOX = <reqBringBox, doneBringBox>
fluent PROCESS_SHIP = <reqShip, doneShip>

fluent REQ_SHIP_TIMEING = <pack, doneShip>

fluent SHIP_TIMING = <['robot1].put['shippingArea], doneShip>
fluent BOX_TIMING = <['robot1].move['boxGettingArea], ['robot1].pick['boxGettingArea]>
fluent ITEM_TIMING = <['robot2].put['itemGettingArea], ['robot2].pick['itemGettingArea]>
fluent GET_TIMING_1 = <['robot2].put['packingArea], ['robot2].pick['packingArea]>
fluent GET_TIMING_2 = <doneBringBox, ['robot2].put['packingArea]>
fluent PACK_TIMING_1 = <doneBringBox, reqShip>
fluent PACK_TIMING_2 = <doneBringItem, reqShip>

fluent RETURN[i:PodID] = <['robot2].pick[i], ['robot2].move[i]>

/*FLTL*/
// Two Robot
ltl_property SHIPROBOT_RULE = [](!{['robot1].move['itemGettingArea]})
ltl_property REPLENISH_ROBOT_RULE = [](!{['robot2].move['shippingArea], ['robot2].move['boxGettingArea]})

ltl_property REQUEST_TIMING_1 = [](reqBringBox -> X(!reqShip W doneBringBox))
ltl_property REQUEST_TIMING_2 = [](reqShip -> X(!reqBringBox W doneShip))
ltl_property REQUEST_TIMING_3 = [](reqReplenishItem -> X(!reqBringItem W doneReplenishItem))
ltl_property REQUEST_TIMING_4 = [](reqBringItem -> X(!reqReplenishItem W doneBringItem))

// Single Robot
ltl_property REQUEST_ONLY_1 = [](!(PROCESS_SHIP || PROCESS_BRING_BOX) -> !{['robot1].move[PodID], ['robot1].move[Area], ['robot1].pick[PodID], ['robot1].put[PodID], ['robot1].pick[Area], ['robot1].put[Area]})
ltl_property REQUEST_ONLY_2 = [](!(PROCESS_REPLENISH_ITEM || PROCESS_BRING_ITEM) -> !{['robot2].move[PodID], ['robot2].move[Area], ['robot2].pick[PodID], ['robot2].put[PodID], ['robot2].pick[Area], ['robot2].put[Area]})

ltl_property ROBOT1_ARM_RULE_1 = [](!(PROCESS_SHIP || PROCESS_BRING_BOX) -> !{Robot1_ArmActions})
ltl_property ROBOT1_ARM_RULE_2 = []({Robot1_PickActions} -> X(!{Robot1_PickActions} W {Robot1_PutActions}))
ltl_property ROBOT1_ARM_RULE_3 = []({Robot1_PutActions} -> X(!{Robot1_PutActions} W {Robot1_PickActions}))
ltl_property ROBOT2_ARM_RULE_1 = [](!(PROCESS_REPLENISH_ITEM || PROCESS_BRING_ITEM) -> !{Robot2_ArmActions})
ltl_property ROBOT2_ARM_RULE_2 = []({Robot2_PickActions} -> X(!{Robot2_PickActions} W {Robot2_PutActions}))
ltl_property ROBOT2_ARM_RULE_3 = []({Robot2_PutActions} -> X(!{Robot2_PutActions} W {Robot2_PickActions}))

//ltl_property REPLENISH_RULE_1 = [](REPLENISH[PodID] -> X(!{replenish[PodID]} W ['robot2].move[PodID]) && !['robot2].move[PodID])
ltl_property REPLENISH_RULE_1 = [](REPLENISH[PodID] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE_2 = [](replenish[PodID] -> X(!{{ReplenishRobotActions}\{['robot2].move[PodID]}} W ['robot2].move[PodID]))
ltl_property REPLENISH_RULE01 = [](REPLENISH[1] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE02 = [](REPLENISH[2] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE03 = [](REPLENISH[3] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE04 = [](REPLENISH[4] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE05 = [](REPLENISH[5] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE06 = [](REPLENISH[6] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE07 = [](REPLENISH[7] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE08 = [](REPLENISH[8] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE09 = [](REPLENISH[9] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE10 = [](REPLENISH[10] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE11 = [](REPLENISH[11] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE12 = [](REPLENISH[12] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE13 = [](REPLENISH[13] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE14 = [](REPLENISH[14] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE15 = [](REPLENISH[15] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE16 = [](REPLENISH[16] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE17 = [](REPLENISH[17] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE18 = [](REPLENISH[18] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE19 = [](REPLENISH[19] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE20 = [](REPLENISH[20] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE21 = [](REPLENISH[21] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE22 = [](REPLENISH[22] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE23 = [](REPLENISH[23] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE24 = [](REPLENISH[24] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE25 = [](REPLENISH[25] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE26 = [](REPLENISH[26] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE27 = [](REPLENISH[27] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE28 = [](REPLENISH[28] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE29 = [](REPLENISH[29] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE30 = [](REPLENISH[30] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE31 = [](REPLENISH[31] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE32 = [](REPLENISH[32] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE33 = [](REPLENISH[33] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE34 = [](REPLENISH[34] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE35 = [](REPLENISH[35] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE36 = [](REPLENISH[36] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE37 = [](REPLENISH[37] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE38 = [](REPLENISH[38] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE39 = [](REPLENISH[39] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE40 = [](REPLENISH[40] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE41 = [](REPLENISH[41] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE42 = [](REPLENISH[42] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE43 = [](REPLENISH[43] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE44 = [](REPLENISH[44] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE45 = [](REPLENISH[45] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE46 = [](REPLENISH[46] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE47 = [](REPLENISH[47] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE48 = [](REPLENISH[48] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE49 = [](REPLENISH[49] -> !['robot2].move[PodID])
ltl_property REPLENISH_RULE50 = [](REPLENISH[50] -> !['robot2].move[PodID])

//ltl_property BRING_RULE_1 = [](BRING[PodID] -> X(!{bring[PodID]} W ['robot2].move[PodID]) && !['robot2].move[PodID])
ltl_property BRING_RULE_1 = [](BRING[PodID] -> !['robot2].move[PodID])
ltl_property BRING_RULE_2 = [](bring[PodID] -> X(!{{ReplenishRobotActions}\{['robot2].move[PodID]}} W ['robot2].move[PodID]))
ltl_property BRING_RULE01 = [](BRING[1] -> !['robot2].move[PodID])
ltl_property BRING_RULE02 = [](BRING[2] -> !['robot2].move[PodID])
ltl_property BRING_RULE03 = [](BRING[3] -> !['robot2].move[PodID])
ltl_property BRING_RULE04 = [](BRING[4] -> !['robot2].move[PodID])
ltl_property BRING_RULE05 = [](BRING[5] -> !['robot2].move[PodID])
ltl_property BRING_RULE06 = [](BRING[6] -> !['robot2].move[PodID])
ltl_property BRING_RULE07 = [](BRING[7] -> !['robot2].move[PodID])
ltl_property BRING_RULE08 = [](BRING[8] -> !['robot2].move[PodID])
ltl_property BRING_RULE09 = [](BRING[9] -> !['robot2].move[PodID])
ltl_property BRING_RULE10 = [](BRING[10] -> !['robot2].move[PodID])
ltl_property BRING_RULE11 = [](BRING[11] -> !['robot2].move[PodID])
ltl_property BRING_RULE12 = [](BRING[12] -> !['robot2].move[PodID])
ltl_property BRING_RULE13 = [](BRING[13] -> !['robot2].move[PodID])
ltl_property BRING_RULE14 = [](BRING[14] -> !['robot2].move[PodID])
ltl_property BRING_RULE15 = [](BRING[15] -> !['robot2].move[PodID])
ltl_property BRING_RULE16 = [](BRING[16] -> !['robot2].move[PodID])
ltl_property BRING_RULE17 = [](BRING[17] -> !['robot2].move[PodID])
ltl_property BRING_RULE18 = [](BRING[18] -> !['robot2].move[PodID])
ltl_property BRING_RULE19 = [](BRING[19] -> !['robot2].move[PodID])
ltl_property BRING_RULE20 = [](BRING[20] -> !['robot2].move[PodID])
ltl_property BRING_RULE21 = [](BRING[21] -> !['robot2].move[PodID])
ltl_property BRING_RULE22 = [](BRING[22] -> !['robot2].move[PodID])
ltl_property BRING_RULE23 = [](BRING[23] -> !['robot2].move[PodID])
ltl_property BRING_RULE24 = [](BRING[24] -> !['robot2].move[PodID])
ltl_property BRING_RULE25 = [](BRING[25] -> !['robot2].move[PodID])
ltl_property BRING_RULE26 = [](BRING[26] -> !['robot2].move[PodID])
ltl_property BRING_RULE27 = [](BRING[27] -> !['robot2].move[PodID])
ltl_property BRING_RULE28 = [](BRING[28] -> !['robot2].move[PodID])
ltl_property BRING_RULE29 = [](BRING[29] -> !['robot2].move[PodID])
ltl_property BRING_RULE30 = [](BRING[30] -> !['robot2].move[PodID])
ltl_property BRING_RULE31 = [](BRING[31] -> !['robot2].move[PodID])
ltl_property BRING_RULE32 = [](BRING[32] -> !['robot2].move[PodID])
ltl_property BRING_RULE33 = [](BRING[33] -> !['robot2].move[PodID])
ltl_property BRING_RULE34 = [](BRING[34] -> !['robot2].move[PodID])
ltl_property BRING_RULE35 = [](BRING[35] -> !['robot2].move[PodID])
ltl_property BRING_RULE36 = [](BRING[36] -> !['robot2].move[PodID])
ltl_property BRING_RULE37 = [](BRING[37] -> !['robot2].move[PodID])
ltl_property BRING_RULE38 = [](BRING[38] -> !['robot2].move[PodID])
ltl_property BRING_RULE39 = [](BRING[39] -> !['robot2].move[PodID])
ltl_property BRING_RULE40 = [](BRING[40] -> !['robot2].move[PodID])
ltl_property BRING_RULE41 = [](BRING[41] -> !['robot2].move[PodID])
ltl_property BRING_RULE42 = [](BRING[42] -> !['robot2].move[PodID])
ltl_property BRING_RULE43 = [](BRING[43] -> !['robot2].move[PodID])
ltl_property BRING_RULE44 = [](BRING[44] -> !['robot2].move[PodID])
ltl_property BRING_RULE45 = [](BRING[45] -> !['robot2].move[PodID])
ltl_property BRING_RULE46 = [](BRING[46] -> !['robot2].move[PodID])
ltl_property BRING_RULE47 = [](BRING[47] -> !['robot2].move[PodID])
ltl_property BRING_RULE48 = [](BRING[48] -> !['robot2].move[PodID])
ltl_property BRING_RULE49 = [](BRING[49] -> !['robot2].move[PodID])
ltl_property BRING_RULE50 = [](BRING[50] -> !['robot2].move[PodID])

ltl_property AREA_RULE_1 = [](!SHIP_TIMING -> !ship)
ltl_property AREA_RULE_2 = [](!BOX_TIMING -> !supplyBox)
ltl_property AREA_RULE_3 = [](!ITEM_TIMING -> !supplyItem)
ltl_property AREA_RULE_4 = [](!(GET_TIMING_1 && !GET_TIMING_2 && PROCESS_BRING_ITEM) -> !getItem)
ltl_property AREA_RULE_5 = [](!PACK_TIMING_1 -> !pack)
ltl_property AREA_RULE_6 = [](!PACK_TIMING_2 -> !pack)
ltl_property AREA_RULE_7 = [](pack -> X(!pack W doneBringBox))
ltl_property AREA_RULE_8 = [](pack -> X(!pack W doneBringItem))

set ReplenishItemActions = {reqReplenishItem, replenish[PodID], ['robot2].move[PodID], ['robot2].pick[PodID], ['robot2].move['itemGettingArea], ['robot2].put['itemGettingArea],
                           supplyItem, ['robot2].pick['itemGettingArea], /*['robot2].move[PodID],*/ ['robot2].put[PodID], doneReplenishItem}
ltl_property REPLENISH_ITEM_RULE_1 = [](PROCESS_REPLENISH_ITEM -> !{{ReplenishRobotActions}\{ReplenishItemActions}})
ltl_property REPLENISH_ITEM_RULE_2 = [](reqReplenishItem -> X(!{{ReplenishRobotActions}\{replenish[PodID]}} W replenish[PodID]))
ltl_property REPLENISH_ITEM_RULE_3 = [](reqReplenishItem -> X(!{['robot2].pick[PodID], ['robot2].pick[Area]} W ['robot2].move[PodID]))
ltl_property REPLENISH_ITEM_RULE_4 = []((PROCESS_REPLENISH_ITEM && ['robot2].move[PodID]) -> X(!{['robot2].move[PodID], ['robot2].move[Area]} W {['robot2].pick[PodID], ['robot2].put[PodID]}))
ltl_property REPLENISH_ITEM_RULE_5 = [](reqReplenishItem -> X(!['robot2].put[PodID] W ['robot2].pick[PodID]))
ltl_property REPLENISH_ITEM_RULE_6 = []((PROCESS_REPLENISH_ITEM && ['robot2].pick[PodID]) -> X(!{{ReplenishRobotActions}\{['robot2].move['itemGettingArea]}} W ['robot2].move['itemGettingArea]))
ltl_property REPLENISH_ITEM_RULE_7 = [](['robot2].move['itemGettingArea] -> X(!['robot2].move[PodID] W ['robot2].pick['itemGettingArea]))
ltl_property REPLENISH_ITEM_RULE_8 = [](['robot2].move['itemGettingArea] -> X(!['robot2].pick['itemGettingArea] W supplyItem))
ltl_property REPLENISH_ITEM_RULE_9 = [](supplyItem -> X(!supplyItem W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE_10 = [](['robot2].pick['itemGettingArea] -> X(!['robot2].put['itemGettingArea] W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE_11 = []((PROCESS_REPLENISH_ITEM && RETURN[PodID]) -> !['robot2].move[PodID])
ltl_property REPLENISH_ITEM_RULE_12 = []((PROCESS_REPLENISH_ITEM && ['robot2].put[PodID]) -> X(!{{ReplenishRobotActions}\{doneReplenishItem}} W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE_13 = []((PROCESS_REPLENISH_ITEM && reqReplenishItem) -> X(!doneReplenishItem W ['robot2].put[PodID]))

ltl_property REPLENISH_ITEM_RULE01_11 = [](replenish[1] -> X(!{['robot2].move[PodID]}\{['robot2].move[1]} W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE02_11 = [](replenish[2] -> X(!{['robot2].move[PodID]}\{['robot2].move[2]} W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE03_11 = [](replenish[3] -> X(!{['robot2].move[PodID]}\{['robot2].move[3]} W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE04_11 = [](replenish[4] -> X(!{['robot2].move[PodID]}\{['robot2].move[4]} W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE05_11 = [](replenish[5] -> X(!{['robot2].move[PodID]}\{['robot2].move[5]} W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE06_11 = [](replenish[6] -> X(!{['robot2].move[PodID]}\{['robot2].move[6]} W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE07_11 = [](replenish[7] -> X(!{['robot2].move[PodID]}\{['robot2].move[7]} W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE08_11 = [](replenish[8] -> X(!{['robot2].move[PodID]}\{['robot2].move[8]} W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE09_11 = [](replenish[9] -> X(!{['robot2].move[PodID]}\{['robot2].move[9]} W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE10_11 = [](replenish[10] -> X(!{['robot2].move[PodID]}\{['robot2].move[10]} W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE11_11 = [](replenish[11] -> X(!{['robot2].move[PodID]}\{['robot2].move[11]} W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE12_11 = [](replenish[12] -> X(!{['robot2].move[PodID]}\{['robot2].move[12]} W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE13_11 = [](replenish[13] -> X(!{['robot2].move[PodID]}\{['robot2].move[13]} W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE14_11 = [](replenish[14] -> X(!{['robot2].move[PodID]}\{['robot2].move[14]} W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE15_11 = [](replenish[15] -> X(!{['robot2].move[PodID]}\{['robot2].move[15]} W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE16_11 = [](replenish[16] -> X(!{['robot2].move[PodID]}\{['robot2].move[16]} W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE17_11 = [](replenish[17] -> X(!{['robot2].move[PodID]}\{['robot2].move[17]} W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE18_11 = [](replenish[18] -> X(!{['robot2].move[PodID]}\{['robot2].move[18]} W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE19_11 = [](replenish[19] -> X(!{['robot2].move[PodID]}\{['robot2].move[19]} W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE20_11 = [](replenish[20] -> X(!{['robot2].move[PodID]}\{['robot2].move[20]} W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE21_11 = [](replenish[21] -> X(!{['robot2].move[PodID]}\{['robot2].move[21]} W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE22_11 = [](replenish[22] -> X(!{['robot2].move[PodID]}\{['robot2].move[22]} W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE23_11 = [](replenish[23] -> X(!{['robot2].move[PodID]}\{['robot2].move[23]} W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE24_11 = [](replenish[24] -> X(!{['robot2].move[PodID]}\{['robot2].move[24]} W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE25_11 = [](replenish[25] -> X(!{['robot2].move[PodID]}\{['robot2].move[25]} W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE26_11 = [](replenish[26] -> X(!{['robot2].move[PodID]}\{['robot2].move[26]} W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE27_11 = [](replenish[27] -> X(!{['robot2].move[PodID]}\{['robot2].move[27]} W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE28_11 = [](replenish[28] -> X(!{['robot2].move[PodID]}\{['robot2].move[28]} W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE29_11 = [](replenish[29] -> X(!{['robot2].move[PodID]}\{['robot2].move[29]} W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE30_11 = [](replenish[30] -> X(!{['robot2].move[PodID]}\{['robot2].move[30]} W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE31_11 = [](replenish[31] -> X(!{['robot2].move[PodID]}\{['robot2].move[31]} W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE32_11 = [](replenish[32] -> X(!{['robot2].move[PodID]}\{['robot2].move[32]} W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE33_11 = [](replenish[33] -> X(!{['robot2].move[PodID]}\{['robot2].move[33]} W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE34_11 = [](replenish[34] -> X(!{['robot2].move[PodID]}\{['robot2].move[34]} W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE35_11 = [](replenish[35] -> X(!{['robot2].move[PodID]}\{['robot2].move[35]} W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE36_11 = [](replenish[36] -> X(!{['robot2].move[PodID]}\{['robot2].move[36]} W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE37_11 = [](replenish[37] -> X(!{['robot2].move[PodID]}\{['robot2].move[37]} W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE38_11 = [](replenish[38] -> X(!{['robot2].move[PodID]}\{['robot2].move[38]} W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE39_11 = [](replenish[39] -> X(!{['robot2].move[PodID]}\{['robot2].move[39]} W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE40_11 = [](replenish[40] -> X(!{['robot2].move[PodID]}\{['robot2].move[40]} W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE41_11 = [](replenish[41] -> X(!{['robot2].move[PodID]}\{['robot2].move[41]} W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE42_11 = [](replenish[42] -> X(!{['robot2].move[PodID]}\{['robot2].move[42]} W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE43_11 = [](replenish[43] -> X(!{['robot2].move[PodID]}\{['robot2].move[43]} W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE44_11 = [](replenish[44] -> X(!{['robot2].move[PodID]}\{['robot2].move[44]} W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE45_11 = [](replenish[45] -> X(!{['robot2].move[PodID]}\{['robot2].move[45]} W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE46_11 = [](replenish[46] -> X(!{['robot2].move[PodID]}\{['robot2].move[46]} W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE47_11 = [](replenish[47] -> X(!{['robot2].move[PodID]}\{['robot2].move[47]} W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE48_11 = [](replenish[48] -> X(!{['robot2].move[PodID]}\{['robot2].move[48]} W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE49_11 = [](replenish[49] -> X(!{['robot2].move[PodID]}\{['robot2].move[49]} W doneReplenishItem))
ltl_property REPLENISH_ITEM_RULE50_11 = [](replenish[50] -> X(!{['robot2].move[PodID]}\{['robot2].move[50]} W doneReplenishItem))

set BringItemActions = {reqBringItem, bring[PodID], ['robot2].move[PodID], ['robot2].pick[PodID], ['robot2].move['packingArea], ['robot2].put['packingArea],
                        getItem, ['robot2].pick['packingArea], /*['robot2].move[PodID],*/ ['robot2].put[PodID], doneBringItem}
ltl_property BRING_ITEM_RULE_1 = [](PROCESS_BRING_ITEM -> !{{ReplenishRobotActions}\{BringItemActions}})
ltl_property BRING_ITEM_RULE_2 = [](reqBringItem -> X(!{{ReplenishRobotActions}\{bring[PodID]}} W bring[PodID]))
ltl_property BRING_ITEM_RULE_3 = [](reqBringItem -> X(!{['robot2].pick[PodID], ['robot2].pick[Area]} W ['robot2].move[PodID]))
ltl_property BRING_ITEM_RULE_4 = []((PROCESS_BRING_ITEM && ['robot2].move[PodID]) -> X(!{['robot2].move[PodID], ['robot2].move[Area]} W {['robot2].pick[PodID], ['robot2].put[PodID]}))
ltl_property BRING_ITEM_RULE_5 = [](reqBringItem -> X(!['robot2].put[PodID] W ['robot2].pick[PodID]))
ltl_property BRING_ITEM_RULE_6 = []((PROCESS_BRING_ITEM && ['robot2].pick[PodID]) -> X(!{{ReplenishRobotActions}\{['robot2].move['packingArea]}} W ['robot2].move['packingArea]))
ltl_property BRING_ITEM_RULE_7 = []((PROCESS_BRING_ITEM && ['robot2].move['packingArea]) -> X(!['robot2].move[PodID] W ['robot2].pick['packingArea]))
ltl_property BRING_ITEM_RULE_8 = []((PROCESS_BRING_ITEM && ['robot2].move['packingArea]) -> X(!['robot2].pick['packingArea] W getItem))
ltl_property BRING_ITEM_RULE_9 = [](getItem -> X(!getItem W doneBringItem))
ltl_property BRING_ITEM_RULE_10 = []((PROCESS_BRING_ITEM && ['robot2].pick['packingArea]) -> X(!['robot2].put['packingArea] W doneBringItem))
ltl_property BRING_ITEM_RULE_11 = []((PROCESS_BRING_ITEM && RETURN[PodID]) -> !['robot2].move[PodID])
ltl_property BRING_ITEM_RULE_12 = []((PROCESS_BRING_ITEM && ['robot2].put[PodID]) -> X(!{{ReplenishRobotActions}\{doneBringItem}} W doneBringItem))
ltl_property BRING_ITEM_RULE_13 = []((PROCESS_BRING_ITEM && reqBringItem) -> X(!doneBringItem W ['robot2].put[PodID]))

ltl_property BRING_ITEM_RULE01_11 = [](bring[1] -> X(!{['robot2].move[PodID]}\{['robot2].move[1]} W doneBringItem))
ltl_property BRING_ITEM_RULE02_11 = [](bring[2] -> X(!{['robot2].move[PodID]}\{['robot2].move[2]} W doneBringItem))
ltl_property BRING_ITEM_RULE03_11 = [](bring[3] -> X(!{['robot2].move[PodID]}\{['robot2].move[3]} W doneBringItem))
ltl_property BRING_ITEM_RULE04_11 = [](bring[4] -> X(!{['robot2].move[PodID]}\{['robot2].move[4]} W doneBringItem))
ltl_property BRING_ITEM_RULE05_11 = [](bring[5] -> X(!{['robot2].move[PodID]}\{['robot2].move[5]} W doneBringItem))
ltl_property BRING_ITEM_RULE06_11 = [](bring[6] -> X(!{['robot2].move[PodID]}\{['robot2].move[6]} W doneBringItem))
ltl_property BRING_ITEM_RULE07_11 = [](bring[7] -> X(!{['robot2].move[PodID]}\{['robot2].move[7]} W doneBringItem))
ltl_property BRING_ITEM_RULE08_11 = [](bring[8] -> X(!{['robot2].move[PodID]}\{['robot2].move[8]} W doneBringItem))
ltl_property BRING_ITEM_RULE09_11 = [](bring[9] -> X(!{['robot2].move[PodID]}\{['robot2].move[9]} W doneBringItem))
ltl_property BRING_ITEM_RULE10_11 = [](bring[10] -> X(!{['robot2].move[PodID]}\{['robot2].move[10]} W doneBringItem))
ltl_property BRING_ITEM_RULE11_11 = [](bring[11] -> X(!{['robot2].move[PodID]}\{['robot2].move[11]} W doneBringItem))
ltl_property BRING_ITEM_RULE12_11 = [](bring[12] -> X(!{['robot2].move[PodID]}\{['robot2].move[12]} W doneBringItem))
ltl_property BRING_ITEM_RULE13_11 = [](bring[13] -> X(!{['robot2].move[PodID]}\{['robot2].move[13]} W doneBringItem))
ltl_property BRING_ITEM_RULE14_11 = [](bring[14] -> X(!{['robot2].move[PodID]}\{['robot2].move[14]} W doneBringItem))
ltl_property BRING_ITEM_RULE15_11 = [](bring[15] -> X(!{['robot2].move[PodID]}\{['robot2].move[15]} W doneBringItem))
ltl_property BRING_ITEM_RULE16_11 = [](bring[16] -> X(!{['robot2].move[PodID]}\{['robot2].move[16]} W doneBringItem))
ltl_property BRING_ITEM_RULE17_11 = [](bring[17] -> X(!{['robot2].move[PodID]}\{['robot2].move[17]} W doneBringItem))
ltl_property BRING_ITEM_RULE18_11 = [](bring[18] -> X(!{['robot2].move[PodID]}\{['robot2].move[18]} W doneBringItem))
ltl_property BRING_ITEM_RULE19_11 = [](bring[19] -> X(!{['robot2].move[PodID]}\{['robot2].move[19]} W doneBringItem))
ltl_property BRING_ITEM_RULE20_11 = [](bring[20] -> X(!{['robot2].move[PodID]}\{['robot2].move[20]} W doneBringItem))
ltl_property BRING_ITEM_RULE21_11 = [](bring[21] -> X(!{['robot2].move[PodID]}\{['robot2].move[21]} W doneBringItem))
ltl_property BRING_ITEM_RULE22_11 = [](bring[22] -> X(!{['robot2].move[PodID]}\{['robot2].move[22]} W doneBringItem))
ltl_property BRING_ITEM_RULE23_11 = [](bring[23] -> X(!{['robot2].move[PodID]}\{['robot2].move[23]} W doneBringItem))
ltl_property BRING_ITEM_RULE24_11 = [](bring[24] -> X(!{['robot2].move[PodID]}\{['robot2].move[24]} W doneBringItem))
ltl_property BRING_ITEM_RULE25_11 = [](bring[25] -> X(!{['robot2].move[PodID]}\{['robot2].move[25]} W doneBringItem))
ltl_property BRING_ITEM_RULE26_11 = [](bring[26] -> X(!{['robot2].move[PodID]}\{['robot2].move[26]} W doneBringItem))
ltl_property BRING_ITEM_RULE27_11 = [](bring[27] -> X(!{['robot2].move[PodID]}\{['robot2].move[27]} W doneBringItem))
ltl_property BRING_ITEM_RULE28_11 = [](bring[28] -> X(!{['robot2].move[PodID]}\{['robot2].move[28]} W doneBringItem))
ltl_property BRING_ITEM_RULE29_11 = [](bring[29] -> X(!{['robot2].move[PodID]}\{['robot2].move[29]} W doneBringItem))
ltl_property BRING_ITEM_RULE30_11 = [](bring[30] -> X(!{['robot2].move[PodID]}\{['robot2].move[30]} W doneBringItem))
ltl_property BRING_ITEM_RULE31_11 = [](bring[31] -> X(!{['robot2].move[PodID]}\{['robot2].move[31]} W doneBringItem))
ltl_property BRING_ITEM_RULE32_11 = [](bring[32] -> X(!{['robot2].move[PodID]}\{['robot2].move[32]} W doneBringItem))
ltl_property BRING_ITEM_RULE33_11 = [](bring[33] -> X(!{['robot2].move[PodID]}\{['robot2].move[33]} W doneBringItem))
ltl_property BRING_ITEM_RULE34_11 = [](bring[34] -> X(!{['robot2].move[PodID]}\{['robot2].move[34]} W doneBringItem))
ltl_property BRING_ITEM_RULE35_11 = [](bring[35] -> X(!{['robot2].move[PodID]}\{['robot2].move[35]} W doneBringItem))
ltl_property BRING_ITEM_RULE36_11 = [](bring[36] -> X(!{['robot2].move[PodID]}\{['robot2].move[36]} W doneBringItem))
ltl_property BRING_ITEM_RULE37_11 = [](bring[37] -> X(!{['robot2].move[PodID]}\{['robot2].move[37]} W doneBringItem))
ltl_property BRING_ITEM_RULE38_11 = [](bring[38] -> X(!{['robot2].move[PodID]}\{['robot2].move[38]} W doneBringItem))
ltl_property BRING_ITEM_RULE39_11 = [](bring[39] -> X(!{['robot2].move[PodID]}\{['robot2].move[39]} W doneBringItem))
ltl_property BRING_ITEM_RULE40_11 = [](bring[40] -> X(!{['robot2].move[PodID]}\{['robot2].move[40]} W doneBringItem))
ltl_property BRING_ITEM_RULE41_11 = [](bring[41] -> X(!{['robot2].move[PodID]}\{['robot2].move[41]} W doneBringItem))
ltl_property BRING_ITEM_RULE42_11 = [](bring[42] -> X(!{['robot2].move[PodID]}\{['robot2].move[42]} W doneBringItem))
ltl_property BRING_ITEM_RULE43_11 = [](bring[43] -> X(!{['robot2].move[PodID]}\{['robot2].move[43]} W doneBringItem))
ltl_property BRING_ITEM_RULE44_11 = [](bring[44] -> X(!{['robot2].move[PodID]}\{['robot2].move[44]} W doneBringItem))
ltl_property BRING_ITEM_RULE45_11 = [](bring[45] -> X(!{['robot2].move[PodID]}\{['robot2].move[45]} W doneBringItem))
ltl_property BRING_ITEM_RULE46_11 = [](bring[46] -> X(!{['robot2].move[PodID]}\{['robot2].move[46]} W doneBringItem))
ltl_property BRING_ITEM_RULE47_11 = [](bring[47] -> X(!{['robot2].move[PodID]}\{['robot2].move[47]} W doneBringItem))
ltl_property BRING_ITEM_RULE48_11 = [](bring[48] -> X(!{['robot2].move[PodID]}\{['robot2].move[48]} W doneBringItem))
ltl_property BRING_ITEM_RULE49_11 = [](bring[49] -> X(!{['robot2].move[PodID]}\{['robot2].move[49]} W doneBringItem))
ltl_property BRING_ITEM_RULE50_11 = [](bring[50] -> X(!{['robot2].move[PodID]}\{['robot2].move[50]} W doneBringItem))

set BringBoxActions = {reqBringBox, ['robot1].move['boxGettingArea], supplyBox, ['robot1].pick['boxGettingArea], ['robot1].move['packingArea], ['robot1].put['packingArea], doneBringBox}
ltl_property BRING_BOX_RULE_1 = [](PROCESS_BRING_BOX -> !{{ShippingRobotActions}\{BringBoxActions}})
ltl_property BRING_BOX_RULE_2 = [](reqBringBox -> X(!['robot1].move['packingArea] W ['robot1].pick['boxGettingArea]))
ltl_property BRING_BOX_RULE_3 = [](reqBringBox -> X(!['robot1].pick['boxGettingArea] W supplyBox))
ltl_property BRING_BOX_RULE_4 = [](supplyBox -> X(!supplyBox W ['robot1].pick['boxGettingArea]))
ltl_property BRING_BOX_RULE_5 = [](['robot1].pick['boxGettingArea] -> X(!['robot1].move['boxGettingArea] W doneBringBox))
ltl_property BRING_BOX_RULE_6 = [](reqBringBox -> X(!doneBringBox W ['robot1].put['packingArea]))

set ShipActions = {reqShip, ['robot1].move['packingArea], ['robot1].pick['packingArea], ['robot1].move['shippingArea], ['robot1].put['shippingArea], ship, doneShip}
ltl_property SHIP_RULE_0 = [](!REQ_SHIP_TIMEING -> !reqShip)
ltl_property SHIP_RULE_1 = [](PROCESS_SHIP -> !{{ShippingRobotActions}\{ShipActions}})
ltl_property SHIP_RULE_2 = [](reqShip -> X(!['robot1].move['shippingArea] W ['robot1].pick['packingArea]))
ltl_property SHIP_RULE_3 = [](['robot1].move['shippingArea] -> X(!['robot1].move['packingArea] W doneShip))
ltl_property SHIP_RULE_4 = [](reqShip -> X(!ship W ['robot1].put['shippingArea]))
ltl_property SHIP_RULE_5 = [](ship -> X(!ship W doneShip))
ltl_property SHIP_RULE_6 = [](reqShip -> X(!doneShip W ship))

set R1_Moves = {['robot1].move[PodID], ['robot1].move[Area]}
set R2_Moves = {['robot2].move[PodID], ['robot2].move[Area]}

fluent AT_R1_POD[p:PodID] = <['robot1].move[p], R1_Moves\{['robot1].move[p]}>
fluent AT_R1_AREA[a:Area] = <['robot1].move[a], R1_Moves\{['robot1].move[a]}>

fluent AT_R2_POD[p:PodID] = <['robot2].move[p], R2_Moves\{['robot2].move[p]}>
fluent AT_R2_AREA[a:Area] = <['robot2].move[a], R2_Moves\{['robot2].move[a]}>

ltl_property NO_COLLISION = [](
    forall[p:PodID] []!(AT_R1_POD[p] && AT_R2_POD[p]) &&
    forall[a:Area] []!(AT_R1_AREA[a] && AT_R2_AREA[a])
)

controllerSpec Specification =
{
    safety = {
        REPLENISH_ROBOT_RULE,
        SHIPROBOT_RULE,

        REQUEST_TIMING_1,
        REQUEST_TIMING_2,
        REQUEST_TIMING_3,
        REQUEST_TIMING_4,

        REQUEST_ONLY_1,
        REQUEST_ONLY_2,
        ROBOT1_ARM_RULE_1,
        ROBOT1_ARM_RULE_2,
        ROBOT1_ARM_RULE_3,
        ROBOT2_ARM_RULE_1,
        ROBOT2_ARM_RULE_2,
        ROBOT2_ARM_RULE_3,

        //REPLENISH_RULE_1, //軽量化の際にはコメントアウト
        REPLENISH_RULE_2,
        //BRING_RULE_1, //軽量化の際にはコメントアウト
        BRING_RULE_2,

        AREA_RULE_1,
        AREA_RULE_2,
        AREA_RULE_3,
        AREA_RULE_4,
        AREA_RULE_5,
        AREA_RULE_6,
        AREA_RULE_7,
        AREA_RULE_8,

        REPLENISH_ITEM_RULE_1,
        REPLENISH_ITEM_RULE_2,
        REPLENISH_ITEM_RULE_3,
        REPLENISH_ITEM_RULE_4,
        REPLENISH_ITEM_RULE_5,
        REPLENISH_ITEM_RULE_6,
        REPLENISH_ITEM_RULE_7,
        REPLENISH_ITEM_RULE_8,
        REPLENISH_ITEM_RULE_9,
        REPLENISH_ITEM_RULE_10,
        //REPLENISH_ITEM_RULE_11, //軽量化の際にはコメントアウト
        REPLENISH_ITEM_RULE_12,
        REPLENISH_ITEM_RULE_13,

        BRING_ITEM_RULE_1,
        BRING_ITEM_RULE_2,
        BRING_ITEM_RULE_3,
        BRING_ITEM_RULE_4,
        BRING_ITEM_RULE_5,
        BRING_ITEM_RULE_6,
        BRING_ITEM_RULE_7,
        BRING_ITEM_RULE_8,
        BRING_ITEM_RULE_9,
        BRING_ITEM_RULE_10,
        //BRING_ITEM_RULE_11, //軽量化の際にはコメントアウト
        BRING_ITEM_RULE_12,
        BRING_ITEM_RULE_13,

        BRING_BOX_RULE_1,
        BRING_BOX_RULE_2,
        BRING_BOX_RULE_3,
        BRING_BOX_RULE_4,
        BRING_BOX_RULE_5,
        BRING_BOX_RULE_6,

        SHIP_RULE_0,
        SHIP_RULE_1,
        SHIP_RULE_2,
        SHIP_RULE_3,
        SHIP_RULE_4,
        SHIP_RULE_5,
        SHIP_RULE_6,

        REPLENISH_RULE01,
        BRING_RULE01,
        REPLENISH_ITEM_RULE01_11,
        BRING_ITEM_RULE01_11,

        REPLENISH_RULE02,
        BRING_RULE02,
        REPLENISH_ITEM_RULE02_11,
        BRING_ITEM_RULE02_11,

        NO_COLLISION
    }
    controllable = {/*AllActions*/ControllableActions}
}

//*****************************************************************************
// Controller
//*****************************************************************************

// Traditional Synthesis
||Environment = (KIVA_System).
controller ||TraditionalController = (Environment)~{Specification}.

// Proposal Synthesis
// controller ||Controller = (['robot1]:INITIAL(1) ||
//     ['robot2]:INITIAL(PodNum) || SHIAREA || BOX_GET_AREA ||
//     ITEM_GET_AREA || PACK_AREA || REQUEST_1 || REQUEST_2 ||
//     REQUEST_3 || REQUEST_4)~{Specification}.
Explanation of the repair
The primary unrealizability risk is that move actions are uncontrollable, while several safety requirements prohibit movements depending on request state, destination, and the other robot's position.

The revised environment adds permitMove actions as controllable synchronization points. This gives the synthesized controller an opportunity to authorize movements before the corresponding uncontrollable event becomes enabled.

The new request-permission processes prevent conflicting requests from overlapping, while COLLISION_PERMISSION tracks both robot locations and suppresses authorizations to occupied destinations. All original safety monitors remain unchanged.

Important verification limitations: The revised model has not been compiled or synthesized. In particular, the collision-arbitration process must be checked for compatibility with the target FSP parser and synchronization semantics. The original model also contains two definitions of names such as SHIAREA, and its indexed safety expressions may require parser-specific handling. These issues cannot be resolved by claiming synthesizability without running the actual synthesis tool.

The repair is therefore a proposed environment-level synchronization strategy, not a verified realizable controller.