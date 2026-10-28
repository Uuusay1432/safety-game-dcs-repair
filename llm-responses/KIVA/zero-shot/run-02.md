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

// 修正: MoveActions を controller の制御対象にする
set ControllableActions = {MoveActions, AreaActions, ArmActions, DoneActions, reqReplenishItem, reqBringItem, reqBringBox, reqShip}
set MonitorableActions  = {AreaActions}
set AllActions = {MoveActions, AreaActions, ArmActions, RequestActions, DoneActions}

//******************************************************************************************//
// Environment Model
//******************************************************************************************//
INITIAL(N = 1)    = MOVEMENT[N],
MOVEMENT[i:PodID] = (move[ii:PodID] -> MOVEMENT[ii]
					 |move['shippingArea]    -> SHIP_AREA
					 |move['boxGettingArea]  -> BOX_GET_AREA
					 |move['itemGettingArea] -> ITEM_GET_AREA
					 |move['packingArea]     -> PACK_AREA
		             |pick[i] -> MOVEMENT[i]
		             |put[i]  -> MOVEMENT[i]),
SHIP_AREA     = (move[ii:PodID] -> MOVEMENT[ii]
				|move['boxGettingArea]  -> BOX_GET_AREA
				|move['itemGettingArea] -> ITEM_GET_AREA
				|move['packingArea]     -> PACK_AREA
				|pick['shippingArea] -> SHIP_AREA
				|put['shippingArea] -> SHIP_AREA),
BOX_GET_AREA  = (move[ii:PodID] -> MOVEMENT[ii]
				|move['shippingArea]    -> SHIP_AREA
				|move['itemGettingArea] -> ITEM_GET_AREA
				|move['packingArea]     -> PACK_AREA
				|pick['boxGettingArea] -> BOX_GET_AREA
				|put['boxGettingArea] -> BOX_GET_AREA),
ITEM_GET_AREA = (move[ii:PodID] -> MOVEMENT[ii]
				|move['shippingArea]    -> SHIP_AREA
				|move['boxGettingArea]  -> BOX_GET_AREA
				|move['packingArea]     -> PACK_AREA
				|pick['itemGettingArea] -> ITEM_GET_AREA
				|put['itemGettingArea] -> ITEM_GET_AREA),
PACK_AREA     = (move[ii:PodID] -> MOVEMENT[ii]
				|move['shippingArea]    -> SHIP_AREA
				|move['boxGettingArea]  -> BOX_GET_AREA
				|move['itemGettingArea] -> ITEM_GET_AREA
				|pick['packingArea] -> PACK_AREA
				|put['packingArea] -> PACK_AREA).

SHIP_AREA     = (ship -> SHIP_AREA).
BOX_GET_AREA  = (supplyBox -> BOX_GET_AREA).
ITEM_GET_AREA = ({supplyItem, getItem} -> ITEM_GET_AREA).
PACK_AREA     = (pack -> PACK_AREA).

REQUEST_1 = (reqReplenishItem -> replenish[PodID]  -> doneReplenishItem -> REQUEST_1).
REQUEST_2 = (reqBringItem -> bring[PodID]  -> doneBringItem -> REQUEST_2).
REQUEST_3 = (reqBringBox  -> doneBringBox -> REQUEST_3).
REQUEST_4 = (reqShip      -> doneShip     -> REQUEST_4).

||KIVA_System = (['robot1]:INITIAL(1) || ['robot2]:INITIAL(PodNum) || SHIP_AREA || BOX_GET_AREA || ITEM_GET_AREA || PACK_AREA || REQUEST_1 || REQUEST_2 || REQUEST_3 || REQUEST_4).

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
assert SHIP_ROBOT_RULE  = !{['robot1].move['itemGettingArea]}
assert REPLENISH_ROBOT_RULE = !{['robot2].move['shippingArea], ['robot2].move['boxGettingArea]}

assert REQUEST_TIMING_1 = (reqBringBox  -> X(!reqShip      W doneBringBox))
assert REQUEST_TIMING_2 = (reqShip      -> X(!reqBringBox  W doneShip))
assert REQUEST_TIMING_3 = (reqReplenishItem -> X(!reqBringItem W doneReplenishItem))
assert REQUEST_TIMING_4 = (reqBringItem -> X(!reqReplenishItem W doneBringItem))

// Single Robot
assert REQUEST_ONLY_1 = (!(PROCESS_SHIP || PROCESS_BRING_BOX)        -> !{['robot1].move[PodID], ['robot1].move[Area], ['robot1].pick[PodID], ['robot1].put[PodID], ['robot1].pick[Area], ['robot1].put[Area]})
assert REQUEST_ONLY_2 = (!(PROCESS_REPLENISH_ITEM || PROCESS_BRING_ITEM) -> !{['robot2].move[PodID], ['robot2].move[Area], ['robot2].pick[PodID], ['robot2].put[PodID], ['robot2].pick[Area], ['robot2].put[Area]})

assert ROBOT1_ARM_RULE_1 = (!(PROCESS_SHIP || PROCESS_BRING_BOX) -> !{Robot1_ArmActions})
assert ROBOT1_ARM_RULE_2 = ({Robot1_PickActions} -> X(!{Robot1_PickActions} W {Robot1_PutActions}))
assert ROBOT1_ARM_RULE_3 = ({Robot1_PutActions}  -> X(!{Robot1_PutActions} W {Robot1_PickActions}))
assert ROBOT2_ARM_RULE_1 = (!(PROCESS_REPLENISH_ITEM || PROCESS_BRING_ITEM) -> !{Robot2_ArmActions})
assert ROBOT2_ARM_RULE_2 = ({Robot2_PickActions} -> X(!{Robot2_PickActions} W {Robot2_PutActions}))
assert ROBOT2_ARM_RULE_3 = ({Robot2_PutActions}  -> X(!{Robot2_PutActions} W {Robot2_PickActions}))

//assert REPLENISH_RULE_1 = (REPLENISH[PodID] -> X(!{replenish[PodID]} W ['robot2].move[PodID]) && !['robot2].move[PodID])
assert REPLENISH_RULE_1 = (REPLENISH[PodID] -> !['robot2].move[PodID])
assert REPLENISH_RULE_2 = (replenish[PodID] -> X(!{{ReplenishRobotActions}\{['robot2].move[PodID]}} W ['robot2].move[PodID]))
assert REPLENISH_RULE01 = (REPLENISH[1]  -> !['robot2].move[PodID])
assert REPLENISH_RULE02 = (REPLENISH[2]  -> !['robot2].move[PodID])

// Bring
assert BRING_RULE_1 = (BRING[PodID] -> !['robot2].move[PodID])
assert BRING_RULE_2 = (bring[PodID] -> X(!{{ReplenishRobotActions}\{['robot2].move[PodID]}} W ['robot2].move[PodID]))
assert BRING_RULE01 = (BRING[1]  -> !['robot2].move[PodID])
assert BRING_RULE02 = (BRING[2]  -> !['robot2].move[PodID])

assert AREA_RULE_1 = (!SHIP_TIMING   -> !ship)
assert AREA_RULE_2 = (!BOX_TIMING    -> !supplyBox)
assert AREA_RULE_3 = (!ITEM_TIMING   -> !supplyItem)
assert AREA_RULE_4 = (!(GET_TIMING_1 && !GET_TIMING_2 && PROCESS_BRING_ITEM) -> !getItem)
assert AREA_RULE_5 = (!PACK_TIMING_1 -> !pack)
assert AREA_RULE_6 = (!PACK_TIMING_2 -> !pack)
assert AREA_RULE_7 = (pack -> X(!pack W doneBringBox))
assert AREA_RULE_8 = (pack -> X(!pack W doneBringItem))

set ReplenishItemActions = {reqReplenishItem, replenish[PodID], ['robot2].move[PodID], ['robot2].pick[PodID], ['robot2].move['itemGettingArea], ['robot2].put['itemGettingArea], 
						supplyItem, ['robot2].pick['itemGettingArea], /*['robot2].move[PodID],*/ ['robot2].put[PodID], doneReplenishItem}
assert REPLENISH_ITEM_RULE_1 = (PROCESS_REPLENISH_ITEM -> !{{ReplenishRobotActions}\{ReplenishItemActions}})
assert REPLENISH_ITEM_RULE_2 = (reqReplenishItem -> X(!{{ReplenishRobotActions}\{replenish[PodID]}} W replenish[PodID]))
assert REPLENISH_ITEM_RULE_3 = (reqReplenishItem -> X(!{['robot2].pick[PodID], ['robot2].pick[Area]} W ['robot2].move[PodID]))
assert REPLENISH_ITEM_RULE_4 = ((PROCESS_REPLENISH_ITEM && ['robot2].move[PodID]) -> X(!{['robot2].move[PodID], ['robot2].move[Area]} W {['robot2].pick[PodID], ['robot2].put[PodID]}))
assert REPLENISH_ITEM_RULE_5 = (reqReplenishItem -> X(!['robot2].put[PodID] W ['robot2].pick[PodID]))
assert REPLENISH_ITEM_RULE_6 = ((PROCESS_REPLENISH_ITEM && ['robot2].pick[PodID]) -> X(!{{ReplenishRobotActions}\{['robot2].move['itemGettingArea]}} W ['robot2].move['itemGettingArea]))
assert REPLENISH_ITEM_RULE_7 = (['robot2].move['itemGettingArea] -> X(!
…

38s考えました
The main repair is to move control one step earlier, into the environment: a robot may perform a move[...] only after a new controllable permitMove[...] action. Thus an unsafe movement is no longer an uncontrollable environmental event that the controller is forced to accept. The original move, request, arm, timing, and collision requirements are otherwise preserved.

I also merged the duplicated area-process definitions (SHIP_AREA, BOX_GET_AREA, etc.) into single environment states so that their movement/pick/put behavior and area-operation behavior coexist rather than redefining the same FSP process name. The two missing *_RULE01_11/*_RULE02_11 assertions referenced by Specification are restored from the evident pattern.

Revised DCS Model
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

// Environment-level permission/confirmation actions.
// A move can occur only after its corresponding permitMove action.
set MovePermissionActions = {
    [Robot].permitMove[PodID],
    [Robot].permitMove[Area]
}

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

set RequestActions = {
    reqReplenishItem,
    reqBringItem,
    reqBringBox,
    reqShip,
    replenish[PodID],
    bring[PodID]
}

set DoneActions = {
    doneReplenishItem,
    doneBringItem,
    doneBringBox,
    doneShip
}

set ShippingRobotActions = {
    ['robot1].move[PodID],
    ['robot1].pick[PodID],
    ['robot1].put[PodID],
    ['robot1].move[Area],
    ['robot1].pick[Area],
    ['robot1].put[Area]
}

set ReplenishRobotActions = {
    ['robot2].move[PodID],
    ['robot2].pick[PodID],
    ['robot2].put[PodID],
    ['robot2].move[Area],
    ['robot2].pick[Area],
    ['robot2].put[Area]
}

// Move itself is now an environmental/uncontrollable action.
// The controller controls the preceding permission action.
set ControllableActions = {
    MovePermissionActions,
    AreaActions,
    ArmActions,
    DoneActions,
    reqReplenishItem,
    reqBringItem,
    reqBringBox,
    reqShip
}

set MonitorableActions = {AreaActions}

set AllActions = {
    MoveActions,
    MovePermissionActions,
    AreaActions,
    ArmActions,
    RequestActions,
    DoneActions
}

//******************************************************************************************//
// Environment Model
//******************************************************************************************//

INITIAL(N = 1) = MOVEMENT[N]

//------------------------------------------------------------------------------------------
// Pod movement
//------------------------------------------------------------------------------------------

MOVEMENT[i:PodID] = (
      permitMove[ii:PodID]       -> move[ii] -> MOVEMENT[ii]
    | permitMove['shippingArea]  -> move['shippingArea]    -> SHIP_AREA
    | permitMove['boxGettingArea] -> move['boxGettingArea] -> BOX_GET_AREA
    | permitMove['itemGettingArea] -> move['itemGettingArea] -> ITEM_GET_AREA
    | permitMove['packingArea]   -> move['packingArea]     -> PACK_AREA
    | pick[i]                    -> MOVEMENT[i]
    | put[i]                     -> MOVEMENT[i]
).

//------------------------------------------------------------------------------------------
// Area states
//------------------------------------------------------------------------------------------

SHIP_AREA = (
      permitMove[ii:PodID]        -> move[ii] -> MOVEMENT[ii]
    | permitMove['boxGettingArea] -> move['boxGettingArea] -> BOX_GET_AREA
    | permitMove['itemGettingArea] -> move['itemGettingArea] -> ITEM_GET_AREA
    | permitMove['packingArea]    -> move['packingArea] -> PACK_AREA
    | pick['shippingArea]         -> SHIP_AREA
    | put['shippingArea]          -> SHIP_AREA
    | ship                        -> SHIP_AREA
).

BOX_GET_AREA = (
      permitMove[ii:PodID]         -> move[ii] -> MOVEMENT[ii]
    | permitMove['shippingArea]    -> move['shippingArea] -> SHIP_AREA
    | permitMove['itemGettingArea] -> move['itemGettingArea] -> ITEM_GET_AREA
    | permitMove['packingArea]     -> move['packingArea] -> PACK_AREA
    | pick['boxGettingArea]        -> BOX_GET_AREA
    | put['boxGettingArea]         -> BOX_GET_AREA
    | supplyBox                    -> BOX_GET_AREA
).

ITEM_GET_AREA = (
      permitMove[ii:PodID]        -> move[ii] -> MOVEMENT[ii]
    | permitMove['shippingArea]   -> move['shippingArea] -> SHIP_AREA
    | permitMove['boxGettingArea] -> move['boxGettingArea] -> BOX_GET_AREA
    | permitMove['packingArea]    -> move['packingArea] -> PACK_AREA
    | pick['itemGettingArea]     -> ITEM_GET_AREA
    | put['itemGettingArea]      -> ITEM_GET_AREA
    | supplyItem                  -> ITEM_GET_AREA
    | getItem                    -> ITEM_GET_AREA
).

PACK_AREA = (
      permitMove[ii:PodID]        -> move[ii] -> MOVEMENT[ii]
    | permitMove['shippingArea]   -> move['shippingArea] -> SHIP_AREA
    | permitMove['boxGettingArea] -> move['boxGettingArea] -> BOX_GET_AREA
    | permitMove['itemGettingArea] -> move['itemGettingArea] -> ITEM_GET_AREA
    | pick['packingArea]          -> PACK_AREA
    | put['packingArea]           -> PACK_AREA
    | pack                        -> PACK_AREA
).

//------------------------------------------------------------------------------------------
// Requests
//------------------------------------------------------------------------------------------

REQUEST_1 = (
    reqReplenishItem
    -> replenish[PodID]
    -> doneReplenishItem
    -> REQUEST_1
).

REQUEST_2 = (
    reqBringItem
    -> bring[PodID]
    -> doneBringItem
    -> REQUEST_2
).

REQUEST_3 = (
    reqBringBox
    -> doneBringBox
    -> REQUEST_3
).

REQUEST_4 = (
    reqShip
    -> doneShip
    -> REQUEST_4
).

||KIVA_System = (
      ['robot1]:INITIAL(1)
    || ['robot2]:INITIAL(PodNum)
    || SHIP_AREA
    || BOX_GET_AREA
    || ITEM_GET_AREA
    || PACK_AREA
    || REQUEST_1
    || REQUEST_2
    || REQUEST_3
    || REQUEST_4
).

//*****************************************************************************
// Requirement
//*****************************************************************************

/*Fluent*/
fluent REPLENISH[i:PodID] = <replenish[i], ['robot2].move[i]>
fluent BRING[i:PodID]     = <bring[i], ['robot2].move[i]>

fluent PROCESS_REQUEST =
    <RequestActions, DoneActions>

fluent PROCESS_REPLENISH_ITEM =
    <reqReplenishItem, doneReplenishItem>

fluent PROCESS_BRING_ITEM =
    <reqBringItem, doneBringItem>

fluent PROCESS_BRING_BOX =
    <reqBringBox, doneBringBox>

fluent PROCESS_SHIP =
    <reqShip, doneShip>

fluent REQ_SHIP_TIMEING =
    <pack, doneShip>

fluent SHIP_TIMING =
    <['robot1].put['shippingArea], doneShip>

fluent BOX_TIMING =
    <['robot1].move['boxGettingArea], ['robot1].pick['boxGettingArea]>

fluent ITEM_TIMING =
    <['robot2].put['itemGettingArea], ['robot2].pick['itemGettingArea]>

fluent GET_TIMING_1 =
    <['robot2].put['packingArea], ['robot2].pick['packingArea]>

fluent GET_TIMING_2 =
    <doneBringBox, ['robot2].put['packingArea]>

fluent PACK_TIMING_1 =
    <doneBringBox, reqShip>

fluent PACK_TIMING_2 =
    <doneBringItem, reqShip>

fluent RETURN[i:PodID] =
    <['robot2].pick[i], ['robot2].move[i]>

/*FLTL*/

// Two Robot
assert SHIP_ROBOT_RULE =
    !{['robot1].move['itemGettingArea]}

assert REPLENISH_ROBOT_RULE =
    !{['robot2].move['shippingArea], ['robot2].move['boxGettingArea]}

assert REQUEST_TIMING_1 =
    (reqBringBox -> X(!reqShip W doneBringBox))

assert REQUEST_TIMING_2 =
    (reqShip -> X(!reqBringBox W doneShip))

assert REQUEST_TIMING_3 =
    (reqReplenishItem -> X(!reqBringItem W doneReplenishItem))

assert REQUEST_TIMING_4 =
    (reqBringItem -> X(!reqReplenishItem W doneBringItem))

// Single Robot
assert REQUEST_ONLY_1 =
    (!(PROCESS_SHIP || PROCESS_BRING_BOX)
        ->
        !{
            ['robot1].move[PodID],
            ['robot1].move[Area],
            ['robot1].pick[PodID],
            ['robot1].put[PodID],
            ['robot1].pick[Area],
            ['robot1].put[Area]
        })

assert REQUEST_ONLY_2 =
    (!(PROCESS_REPLENISH_ITEM || PROCESS_BRING_ITEM)
        ->
        !{
            ['robot2].move[PodID],
            ['robot2].move[Area],
            ['robot2].pick[PodID],
            ['robot2].put[PodID],
            ['robot2].pick[Area],
            ['robot2].put[Area]
        })

assert ROBOT1_ARM_RULE_1 =
    (!(PROCESS_SHIP || PROCESS_BRING_BOX) -> !{Robot1_ArmActions})

assert ROBOT1_ARM_RULE_2 =
    ({Robot1_PickActions}
        -> X(!{Robot1_PickActions} W {Robot1_PutActions}))

assert ROBOT1_ARM_RULE_3 =
    ({Robot1_PutActions}
        -> X(!{Robot1_PutActions} W {Robot1_PickActions}))

assert ROBOT2_ARM_RULE_1 =
    (!(PROCESS_REPLENISH_ITEM || PROCESS_BRING_ITEM) -> !{Robot2_ArmActions})

assert ROBOT2_ARM_RULE_2 =
    ({Robot2_PickActions}
        -> X(!{Robot2_PickActions} W {Robot2_PutActions}))

assert ROBOT2_ARM_RULE_3 =
    ({Robot2_PutActions}
        -> X(!{Robot2_PutActions} W {Robot2_PickActions}))

// Replenish
assert REPLENISH_RULE_1 =
    (REPLENISH[PodID] -> !['robot2].move[PodID])

assert REPLENISH_RULE_2 =
    (replenish[PodID]
        -> X(!{{ReplenishRobotActions}\{['robot2].move[PodID]}}
             W ['robot2].move[PodID]))

assert REPLENISH_RULE01 =
    (REPLENISH[1] -> !['robot2].move[PodID])

assert REPLENISH_RULE02 =
    (REPLENISH[2] -> !['robot2].move[PodID])

// Bring
assert BRING_RULE_1 =
    (BRING[PodID] -> !['robot2].move[PodID])

assert BRING_RULE_2 =
    (bring[PodID]
        -> X(!{{ReplenishRobotActions}\{['robot2].move[PodID]}}
             W ['robot2].move[PodID]))

assert BRING_RULE01 =
    (BRING[1] -> !['robot2].move[PodID])

assert BRING_RULE02 =
    (BRING[2] -> !['robot2].move[PodID])

// Area
assert AREA_RULE_1 =
    (!SHIP_TIMING -> !ship)

assert AREA_RULE_2 =
    (!BOX_TIMING -> !supplyBox)

assert AREA_RULE_3 =
    (!ITEM_TIMING -> !supplyItem)

assert AREA_RULE_4 =
    (!(GET_TIMING_1 && !GET_TIMING_2 && PROCESS_BRING_ITEM)
        -> !getItem)

assert AREA_RULE_5 =
    (!PACK_TIMING_1 -> !pack)

assert AREA_RULE_6 =
    (!PACK_TIMING_2 -> !pack)

assert AREA_RULE_7 =
    (pack -> X(!pack W doneBringBox))

assert AREA_RULE_8 =
    (pack -> X(!pack W doneBringItem))

// Replenish item
set ReplenishItemActions = {
    reqReplenishItem,
    replenish[PodID],
    ['robot2].move[PodID],
    ['robot2].pick[PodID],
    ['robot2].move['itemGettingArea],
    ['robot2].put['itemGettingArea],
    supplyItem,
    ['robot2].pick['itemGettingArea],
    ['robot2].put[PodID],
    doneReplenishItem
}

assert REPLENISH_ITEM_RULE_1 =
    (PROCESS_REPLENISH_ITEM
        -> !{{ReplenishRobotActions}\{ReplenishItemActions}})

assert REPLENISH_ITEM_RULE_2 =
    (reqReplenishItem
        -> X(!{{ReplenishRobotActions}\{replenish[PodID]}}
             W replenish[PodID]))

assert REPLENISH_ITEM_RULE_3 =
    (reqReplenishItem
        -> X(!{['robot2].pick[PodID], ['robot2].pick[Area]}
             W ['robot2].move[PodID]))

assert REPLENISH_ITEM_RULE_4 =
    ((PROCESS_REPLENISH_ITEM && ['robot2].move[PodID])
        ->
        X(!{['robot2].move[PodID], ['robot2].move[Area]}
          W {['robot2].pick[PodID], ['robot2].put[PodID]}))

assert REPLENISH_ITEM_RULE_5 =
    (reqReplenishItem
        -> X(!['robot2].put[PodID] W ['robot2].pick[PodID]))

assert REPLENISH_ITEM_RULE_6 =
    ((PROCESS_REPLENISH_ITEM && ['robot2].pick[PodID])
        ->
        X(!{{ReplenishRobotActions}\{['robot2].move['itemGettingArea]}}
          W ['robot2].move['itemGettingArea]))

assert REPLENISH_ITEM_RULE_7 =
    (['robot2].move['itemGettingArea]
        -> X(!['robot2].move[PodID]
             W ['robot2].pick['itemGettingArea]))

assert REPLENISH_ITEM_RULE_8 =
    (['robot2].move['itemGettingArea]
        -> X(!['robot2].pick['itemGettingArea]
             W supplyItem))

assert REPLENISH_ITEM_RULE_9 =
    (supplyItem -> X(!supplyItem W doneReplenishItem))

assert REPLENISH_ITEM_RULE_10 =
    (['robot2].pick['itemGettingArea]
        -> X(!['robot2].put['itemGettingArea]
             W doneReplenishItem))

assert REPLENISH_ITEM_RULE_11 =
    ((PROCESS_REPLENISH_ITEM && RETURN[PodID])
        -> !['robot2].move[PodID])

assert REPLENISH_ITEM_RULE_12 =
    ((PROCESS_REPLENISH_ITEM && ['robot2].put[PodID])
        ->
        X(!{{ReplenishRobotActions}\{doneReplenishItem}}
          W doneReplenishItem))

assert REPLENISH_ITEM_RULE_13 =
    ((PROCESS_REPLENISH_ITEM && reqReplenishItem)
        -> X(!doneReplenishItem W ['robot2].put[PodID]))

// Explicit pod-specific return restrictions
assert REPLENISH_ITEM_RULE01_11 =
    (replenish[1]
        -> X(!{['robot2].move[PodID]}\{['robot2].move[1]}
             W doneReplenishItem))

assert REPLENISH_ITEM_RULE02_11 =
    (replenish[2]
        -> X(!{['robot2].move[PodID]}\{['robot2].move[2]}
             W doneReplenishItem))

// Bring item
set BringItemActions = {
    reqBringItem,
    bring[PodID],
    ['robot2].move[PodID],
    ['robot2].pick[PodID],
    ['robot2].move['packingArea],
    ['robot2].put['packingArea],
    getItem,
    ['robot2].pick['packingArea],
    ['robot2].put[PodID],
    doneBringItem
}

assert BRING_ITEM_RULE_1 =
    (PROCESS_BRING_ITEM
        -> !{{ReplenishRobotActions}\{BringItemActions}})

assert BRING_ITEM_RULE_2 =
    (reqBringItem
        -> X(!{{ReplenishRobotActions}\{bring[PodID]}}
             W bring[PodID]))

assert BRING_ITEM_RULE_3 =
    (reqBringItem
        -> X(!{['robot2].pick[PodID], ['robot2].pick[Area]}
             W ['robot2].move[PodID]))

assert BRING_ITEM_RULE_4 =
    ((PROCESS_BRING_ITEM && ['robot2].move[PodID])
        ->
        X(!{['robot2].move[PodID], ['robot2].move[Area]}
          W {['robot2].pick[PodID], ['robot2].put[PodID]}))

assert BRING_ITEM_RULE_5 =
    (reqBringItem
        -> X(!['robot2].put[PodID] W ['robot2].pick[PodID]))

assert BRING_ITEM_RULE_6 =
    ((PROCESS_BRING_ITEM && ['robot2].pick[PodID])
        ->
        X(!{{ReplenishRobotActions}\{['robot2].move['packingArea]}}
          W ['robot2].move['packingArea]))

assert BRING_ITEM_RULE_7 =
    ((PROCESS_BRING_ITEM && ['robot2].move['packingArea])
        -> X(!['robot2].move[PodID]
             W ['robot2].pick['packingArea]))

assert BRING_ITEM_RULE_8 =
    ((PROCESS_BRING_ITEM && ['robot2].move['packingArea])
        -> X(!['robot2].pick['packingArea]
             W getItem))

assert BRING_ITEM_RULE_9 =
    (getItem -> X(!getItem W doneBringItem))

assert BRING_ITEM_RULE_10 =
    ((PROCESS_BRING_ITEM && ['robot2].pick['packingArea])
        -> X(!['robot2].put['packingArea]
             W doneBringItem))

assert BRING_ITEM_RULE_11 =
    ((PROCESS_BRING_ITEM && RETURN[PodID])
        -> !['robot2].move[PodID])

assert BRING_ITEM_RULE_12 =
    ((PROCESS_BRING_ITEM && ['robot2].put[PodID])
        ->
        X(!{{ReplenishRobotActions}\{doneBringItem}}
          W doneBringItem))

assert BRING_ITEM_RULE_13 =
    ((PROCESS_BRING_ITEM && reqBringItem)
        -> X(!doneBringItem W ['robot2].put[PodID]))

assert BRING_ITEM_RULE01_11 =
    (bring[1]
        -> X(!{['robot2].move[PodID]}\{['robot2].move[1]}
             W doneBringItem))

assert BRING_ITEM_RULE02_11 =
    (bring[2]
        -> X(!{['robot2].move[PodID]}\{['robot2].move[2]}
             W doneBringItem))

// Bring box
set BringBoxActions = {
    reqBringBox,
    ['robot1].move['boxGettingArea],
    supplyBox,
    ['robot1].pick['boxGettingArea],
    ['robot1].move['packingArea],
    ['robot1].put['packingArea],
    doneBringBox
}

assert BRING_BOX_RULE_1 =
    (PROCESS_BRING_BOX
        -> !{{ShippingRobotActions}\{BringBoxActions}})

assert BRING_BOX_RULE_2 =
    (reqBringBox
        -> X(!['robot1].move['packingArea]
             W ['robot1].pick['boxGettingArea]))

assert BRING_BOX_RULE_3 =
    (reqBringBox
        -> X(!['robot1].pick['boxGettingArea]
             W supplyBox))

assert BRING_BOX_RULE_4 =
    (supplyBox
        -> X(!supplyBox W ['robot1].pick['boxGettingArea]))

assert BRING_BOX_RULE_5 =
    (['robot1].pick['boxGettingArea]
        -> X(!['robot1].move['boxGettingArea]
             W doneBringBox))

assert BRING_BOX_RULE_6 =
    (reqBringBox
        -> X(!doneBringBox W ['robot1].put['packingArea]))

// Ship
set ShipActions = {
    reqShip,
    ['robot1].move['packingArea],
    ['robot1].pick['packingArea],
    ['robot1].move['shippingArea],
    ['robot1].put['shippingArea],
    ship,
    doneShip
}

assert SHIP_RULE_0 =
    (!REQ_SHIP_TIMEING -> !reqShip)

assert SHIP_RULE_1 =
    (PROCESS_SHIP
        -> !{{ShippingRobotActions}\{ShipActions}})

assert SHIP_RULE_2 =
    (reqShip
        -> X(!['robot1].move['shippingArea]
             W ['robot1].pick['packingArea]))

assert SHIP_RULE_3 =
    (['robot1].move['shippingArea]
        -> X(!['robot1].move['packingArea]
             W doneShip))

assert SHIP_RULE_4 =
    (reqShip
        -> X(!ship W ['robot1].put['shippingArea]))

assert SHIP_RULE_5 =
    (ship -> X(!ship W doneShip))

assert SHIP_RULE_6 =
    (reqShip -> X(!doneShip W ship))

/*LTL Property*/

ltl_property P_SHIP_ROBOT_RULE =
    [](SHIP_ROBOT_RULE)

ltl_property P_REPLENISH_ROBOT_RULE =
    [](REPLENISH_ROBOT_RULE)

ltl_property P_REQUEST_TIMING_1 =
    [](REQUEST_TIMING_1)

ltl_property P_REQUEST_TIMING_2 =
    [](REQUEST_TIMING_2)

ltl_property P_REQUEST_TIMING_3 =
    [](REQUEST_TIMING_3)

ltl_property P_REQUEST_TIMING_4 =
    [](REQUEST_TIMING_4)

ltl_property P_REQUEST_ONLY_1 =
    [](REQUEST_ONLY_1)

ltl_property P_REQUEST_ONLY_2 =
    [](REQUEST_ONLY_2)

ltl_property P_ROBOT1_ARM_RULE_1 =
    [](ROBOT1_ARM_RULE_1)

ltl_property P_ROBOT1_ARM_RULE_2 =
    [](ROBOT1_ARM_RULE_2)

ltl_property P_ROBOT1_ARM_RULE_3 =
    [](ROBOT1_ARM_RULE_3)

ltl_property P_ROBOT2_ARM_RULE_1 =
    [](ROBOT2_ARM_RULE_1)

ltl_property P_ROBOT2_ARM_RULE_2 =
    [](ROBOT2_ARM_RULE_2)

ltl_property P_ROBOT2_ARM_RULE_3 =
    [](ROBOT2_ARM_RULE_3)

ltl_property P_REPLENISH_RULE_1 =
    [](REPLENISH_RULE_1)

ltl_property P_REPLENISH_RULE_2 =
    [](REPLENISH_RULE_2)

ltl_property P_REPLENISH_RULE01 =
    [](REPLENISH_RULE01)

ltl_property P_REPLENISH_RULE02 =
    [](REPLENISH_RULE02)

ltl_property P_BRING_RULE_1 =
    [](BRING_RULE_1)

ltl_property P_BRING_RULE_2 =
    [](BRING_RULE_2)

ltl_property P_BRING_RULE01 =
    [](BRING_RULE01)

ltl_property P_BRING_RULE02 =
    [](BRING_RULE02)

ltl_property P_AREA_RULE_1 =
    [](AREA_RULE_1)

ltl_property P_AREA_RULE_2 =
    [](AREA_RULE_2)

ltl_property P_AREA_RULE_3 =
    [](AREA_RULE_3)

ltl_property P_AREA_RULE_4 =
    [](AREA_RULE_4)

ltl_property P_AREA_RULE_5 =
    [](AREA_RULE_5)

ltl_property P_AREA_RULE_6 =
    [](AREA_RULE_6)

ltl_property P_AREA_RULE_7 =
    [](AREA_RULE_7)

ltl_property P_AREA_RULE_8 =
    [](AREA_RULE_8)

ltl_property P_REPLENISH_ITEM_RULE_1 =
    [](REPLENISH_ITEM_RULE_1)

ltl_property P_REPLENISH_ITEM_RULE_2 =
    [](REPLENISH_ITEM_RULE_2)

ltl_property P_REPLENISH_ITEM_RULE_3 =
    [](REPLENISH_ITEM_RULE_3)

ltl_property P_REPLENISH_ITEM_RULE_4 =
    [](REPLENISH_ITEM_RULE_4)

ltl_property P_REPLENISH_ITEM_RULE_5 =
    [](REPLENISH_ITEM_RULE_5)

ltl_property P_REPLENISH_ITEM_RULE_6 =
    [](REPLENISH_ITEM_RULE_6)

ltl_property P_REPLENISH_ITEM_RULE_7 =
    [](REPLENISH_ITEM_RULE_7)

ltl_property P_REPLENISH_ITEM_RULE_8 =
    [](REPLENISH_ITEM_RULE_8)

ltl_property P_REPLENISH_ITEM_RULE_9 =
    [](REPLENISH_ITEM_RULE_9)

ltl_property P_REPLENISH_ITEM_RULE_10 =
    [](REPLENISH_ITEM_RULE_10)

ltl_property P_REPLENISH_ITEM_RULE_11 =
    [](REPLENISH_ITEM_RULE_11)

ltl_property P_REPLENISH_ITEM_RULE_12 =
    [](REPLENISH_ITEM_RULE_12)

ltl_property P_REPLENISH_ITEM_RULE_13 =
    [](REPLENISH_ITEM_RULE_13)

ltl_property P_BRING_ITEM_RULE_1 =
    [](BRING_ITEM_RULE_1)

ltl_property P_BRING_ITEM_RULE_2 =
    [](BRING_ITEM_RULE_2)

ltl_property P_BRING_ITEM_RULE_3 =
    [](BRING_ITEM_RULE_3)

ltl_property P_BRING_ITEM_RULE_4 =
    [](BRING_ITEM_RULE_4)

ltl_property P_BRING_ITEM_RULE_5 =
    [](BRING_ITEM_RULE_5)

ltl_property P_BRING_ITEM_RULE_6 =
    [](BRING_ITEM_RULE_6)

ltl_property P_BRING_ITEM_RULE_7 =
    [](BRING_ITEM_RULE_7)

ltl_property P_BRING_ITEM_RULE_8 =
    [](BRING_ITEM_RULE_8)

ltl_property P_BRING_ITEM_RULE_9 =
    [](BRING_ITEM_RULE_9)

ltl_property P_BRING_ITEM_RULE_10 =
    [](BRING_ITEM_RULE_10)

ltl_property P_BRING_ITEM_RULE_11 =
    [](BRING_ITEM_RULE_11)

ltl_property P_BRING_ITEM_RULE_12 =
    [](BRING_ITEM_RULE_12)

ltl_property P_BRING_ITEM_RULE_13 =
    [](BRING_ITEM_RULE_13)

ltl_property P_BRING_BOX_RULE_1 =
    [](BRING_BOX_RULE_1)

ltl_property P_BRING_BOX_RULE_2 =
    [](BRING_BOX_RULE_2)

ltl_property P_BRING_BOX_RULE_3 =
    [](BRING_BOX_RULE_3)

ltl_property P_BRING_BOX_RULE_4 =
    [](BRING_BOX_RULE_4)

ltl_property P_BRING_BOX_RULE_5 =
    [](BRING_BOX_RULE_5)

ltl_property P_BRING_BOX_RULE_6 =
    [](BRING_BOX_RULE_6)

ltl_property P_SHIP_RULE_0 =
    [](SHIP_RULE_0)

ltl_property P_SHIP_RULE_1 =
    [](SHIP_RULE_1)

ltl_property P_SHIP_RULE_2 =
    [](SHIP_RULE_2)

ltl_property P_SHIP_RULE_3 =
    [](SHIP_RULE_3)

ltl_property P_SHIP_RULE_4 =
    [](SHIP_RULE_4)

ltl_property P_SHIP_RULE_5 =
    [](SHIP_RULE_5)

ltl_property P_SHIP_RULE_6 =
    [](SHIP_RULE_6)

//*****************************************************************************
// Collision monitor
//*****************************************************************************

set R1_Moves = {
    ['robot1].move[PodID],
    ['robot1].move[Area]
}

set R2_Moves = {
    ['robot2].move[PodID],
    ['robot2].move[Area]
}

fluent AT_R1_POD[p:PodID] =
    <['robot1].move[p],
     R1_Moves\{['robot1].move[p]}>

fluent AT_R1_AREA[a:Area] =
    <['robot1].move[a],
     R1_Moves\{['robot1].move[a]}>

fluent AT_R2_POD[p:PodID] =
    <['robot2].move[p],
     R2_Moves\{['robot2].move[p]}>

fluent AT_R2_AREA[a:Area] =
    <['robot2].move[a],
     R2_Moves\{['robot2].move[a]}>

assert NO_COLLISION = (
    forall[p:PodID]
        []!(AT_R1_POD[p] && AT_R2_POD[p])
    &&
    forall[a:Area]
        []!(AT_R1_AREA[a] && AT_R2_AREA[a])
)

ltl_property P_NO_COLLISION =
    NO_COLLISION

//*****************************************************************************
// Controller Specification
//*****************************************************************************

controllerSpec Specification =
{
    safety = {
        P_REPLENISH_ROBOT_RULE,
        P_SHIP_ROBOT_RULE,

        P_REQUEST_TIMING_1,
        P_REQUEST_TIMING_2,
        P_REQUEST_TIMING_3,
        P_REQUEST_TIMING_4,

        P_REQUEST_ONLY_1,
        P_REQUEST_ONLY_2,

        P_ROBOT1_ARM_RULE_1,
        P_ROBOT1_ARM_RULE_2,
        P_ROBOT1_ARM_RULE_3,

        P_ROBOT2_ARM_RULE_1,
        P_ROBOT2_ARM_RULE_2,
        P_ROBOT2_ARM_RULE_3,

        P_REPLENISH_RULE_2,
        P_BRING_RULE_2,

        P_AREA_RULE_1,
        P_AREA_RULE_2,
        P_AREA_RULE_3,
        P_AREA_RULE_4,
        P_AREA_RULE_5,
        P_AREA_RULE_6,
        P_AREA_RULE_7,
        P_AREA_RULE_8,

        P_REPLENISH_ITEM_RULE_1,
        P_REPLENISH_ITEM_RULE_2,
        P_REPLENISH_ITEM_RULE_3,
        P_REPLENISH_ITEM_RULE_4,
        P_REPLENISH_ITEM_RULE_5,
        P_REPLENISH_ITEM_RULE_6,
        P_REPLENISH_ITEM_RULE_7,
        P_REPLENISH_ITEM_RULE_8,
        P_REPLENISH_ITEM_RULE_9,
        P_REPLENISH_ITEM_RULE_10,
        P_REPLENISH_ITEM_RULE_12,
        P_REPLENISH_ITEM_RULE_13,

        P_BRING_ITEM_RULE_1,
        P_BRING_ITEM_RULE_2,
        P_BRING_ITEM_RULE_3,
        P_BRING_ITEM_RULE_4,
        P_BRING_ITEM_RULE_5,
        P_BRING_ITEM_RULE_6,
        P_BRING_ITEM_RULE_7,
        P_BRING_ITEM_RULE_8,
        P_BRING_ITEM_RULE_9,
        P_BRING_ITEM_RULE_10,
        P_BRING_ITEM_RULE_12,
        P_BRING_ITEM_RULE_13,

        P_BRING_BOX_RULE_1,
        P_BRING_BOX_RULE_2,
        P_BRING_BOX_RULE_3,
        P_BRING_BOX_RULE_4,
        P_BRING_BOX_RULE_5,
        P_BRING_BOX_RULE_6,

        P_SHIP_RULE_0,
        P_SHIP_RULE_1,
        P_SHIP_RULE_2,
        P_SHIP_RULE_3,
        P_SHIP_RULE_4,
        P_SHIP_RULE_5,
        P_SHIP_RULE_6,

        P_REPLENISH_RULE01,
        P_BRING_RULE01,
        P_REPLENISH_ITEM_RULE01_11,
        P_BRING_ITEM_RULE01_11,

        P_REPLENISH_RULE02,
        P_BRING_RULE02,
        P_REPLENISH_ITEM_RULE02_11,
        P_BRING_ITEM_RULE02_11,

        P_NO_COLLISION
    }

    controllable = {ControllableActions}
}

//*****************************************************************************
// Controller
//*****************************************************************************

//Traditional Synthesis
||Environment = (KIVA_System).

controller ||TraditionalController =
    (Environment)~{Specification}.

// Proposal Synthesis
// controller ||Controller =
//     (['robot1]:INITIAL(1)
//      || ['robot2]:INITIAL(PodNum)
//      || SHIP_AREA
//      || BOX_GET_AREA
//      || ITEM_GET_AREA
//      || PACK_AREA
//      || REQUEST_1
//      || REQUEST_2
//      || REQUEST_3
//      || REQUEST_4)~{Specification}.
Why this repair addresses the unrealizability
The critical problem is the timing between the environment and the safety monitors. The original environment exposed move[...] directly from every movement state. Even when MoveActions were placed in ControllableActions, the environment still had unrestricted movement branches at every state, while many monitors impose very precise windows in which a particular movement is permitted.

The repair introduces:

permitMove[...] -> move[...]
for every movement branch. permitMove[...] is controllable, while the actual move[...] remains the physical/environmental action.

Therefore:

The controller can suppress an unsafe movement before it occurs.
Once it grants permission, the environment confirms that movement by executing the corresponding move[...].
All original safety properties continue to observe the same move[...], pick[...], put[...], request, and completion actions.
No safety monitor has been weakened or removed.
Collision detection still operates on the actual move[...] events.
The original robot movement graph and action parameters are retained.
The area-operation branches (ship, supplyBox, supplyItem, getItem, pack) are retained.
permitMove is the only new controllable mechanism.
One additional issue in the supplied model is that SHIP_AREA, BOX_GET_AREA, ITEM_GET_AREA, and PACK_AREA were each defined twice. The revised version combines their movement/pick/put branches with their area-operation self-loops into one definition per process; otherwise an FSP compiler may reject the model because of duplicate process definitions.

Important: this is a structural DCS repair and makes the previously uncontrollable environmental movements suppressible. It is not a claim that a particular LTSA/FSP-DCS implementation has been executed here and formally returned a nonempty controller. The actual synthesis result still needs to be checked in the target DCS tool.

I reviewed the supplied model. The key environment-side repair is the permission/synchronization layer around robot movement: move[...] remains an environmental action, but it is exposed only after the controller grants permitMove[...]. This preserves the original safety monitors while giving the controller a way to prevent uncontrollable movement from violating them. The supplied model already contains this mechanism; I cleaned it into a standalone .fsp model.    貼り付けたテキスト（1）    貼り付けたテキスト（1）
Main repair
- Added controllable permitMove[PodID] / permitMove[Area].
- Changed every environment movement branch to:
  permitMove[...] -> move[...] -> next state.
- Kept the actual move[...] actions uncontrollable, so the repair does not simply change the plant's action ownership.    貼り付けたテキスト（1）
- Preserved the collision monitor and the original safety-property set.    貼り付けたテキスト（1）    貼り付けたテキスト（1）
- Kept the existing process/action names and branching structure as far as possible.
Important qualification: this is an environment repair intended to restore realizability; I cannot honestly claim that an LTSA/DCS synthesis run succeeded because no DCS synthesis engine is available in this session. Also, there is a deeper logical tension in the supplied requirements: REPLENISH_RULE_2/BRING_RULE_2 require the corresponding move eventually, while the associated fluent-based rules prohibit robot 2's pod movement while that fluent is active. For example, REPLENISH is activated by replenish[i] and terminated by move[i], while REPLENISH_RULE_1 forbids that movement while the fluent holds.    貼り付けたテキスト（1）    貼り付けたテキスト（1） This should be checked separately if synthesis still reports unrealizability after the permission repair.

Complete revised model:

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

// Environment-level permission/confirmation actions.
// A move can occur only after its corresponding permitMove action.
set MovePermissionActions = {
    [Robot].permitMove[PodID],
    [Robot].permitMove[Area]
}

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

set RequestActions = {
    reqReplenishItem,
    reqBringItem,
    reqBringBox,
    reqShip,
    replenish[PodID],
    bring[PodID]
}

set DoneActions = {
    doneReplenishItem,
    doneBringItem,
    doneBringBox,
    doneShip
}

set ShippingRobotActions = {
    ['robot1].move[PodID],
    ['robot1].pick[PodID],
    ['robot1].put[PodID],
    ['robot1].move[Area],
    ['robot1].pick[Area],
    ['robot1].put[Area]
}

set ReplenishRobotActions = {
    ['robot2].move[PodID],
    ['robot2].pick[PodID],
    ['robot2].put[PodID],
    ['robot2].move[Area],
    ['robot2].pick[Area],
    ['robot2].put[Area]
}

// Move itself is now an environmental/uncontrollable action.
// The controller controls the preceding permission action.
set ControllableActions = {
    MovePermissionActions,
    AreaActions,
    ArmActions,
    DoneActions,
    reqReplenishItem,
    reqBringItem,
    reqBringBox,
    reqShip
}

set MonitorableActions = {AreaActions}

set AllActions = {
    MoveActions,
    MovePermissionActions,
    AreaActions,
    ArmActions,
    RequestActions,
    DoneActions
}

//******************************************************************************************//
// Environment Model
//******************************************************************************************//

INITIAL(N = 1) = MOVEMENT[N]

//------------------------------------------------------------------------------------------
// Pod movement
//------------------------------------------------------------------------------------------

MOVEMENT[i:PodID] = (
      permitMove[ii:PodID]       -> move[ii] -> MOVEMENT[ii]
    | permitMove['shippingArea]  -> move['shippingArea]    -> SHIP_AREA
    | permitMove['boxGettingArea] -> move['boxGettingArea] -> BOX_GET_AREA
    | permitMove['itemGettingArea] -> move['itemGettingArea] -> ITEM_GET_AREA
    | permitMove['packingArea]   -> move['packingArea]     -> PACK_AREA
    | pick[i]                    -> MOVEMENT[i]
    | put[i]                     -> MOVEMENT[i]
).

//------------------------------------------------------------------------------------------
// Area states
//------------------------------------------------------------------------------------------

SHIP_AREA = (
      permitMove[ii:PodID]        -> move[ii] -> MOVEMENT[ii]
    | permitMove['boxGettingArea] -> move['boxGettingArea] -> BOX_GET_AREA
    | permitMove['itemGettingArea] -> move['itemGettingArea] -> ITEM_GET_AREA
    | permitMove['packingArea]    -> move['packingArea] -> PACK_AREA
    | pick['shippingArea]         -> SHIP_AREA
    | put['shippingArea]          -> SHIP_AREA
    | ship                        -> SHIP_AREA
).

BOX_GET_AREA = (
      permitMove[ii:PodID]         -> move[ii] -> MOVEMENT[ii]
    | permitMove['shippingArea]    -> move['shippingArea] -> SHIP_AREA
    | permitMove['itemGettingArea] -> move['itemGettingArea] -> ITEM_GET_AREA
    | permitMove['packingArea]     -> move['packingArea] -> PACK_AREA
    | pick['boxGettingArea]        -> BOX_GET_AREA
    | put['boxGettingArea]         -> BOX_GET_AREA
    | supplyBox                    -> BOX_GET_AREA
).

ITEM_GET_AREA = (
      permitMove[ii:PodID]        -> move[ii] -> MOVEMENT[ii]
    | permitMove['shippingArea]   -> move['shippingArea] -> SHIP_AREA
    | permitMove['boxGettingArea] -> move['boxGettingArea] -> BOX_GET_AREA
    | permitMove['packingArea]    -> move['packingArea] -> PACK_AREA
    | pick['itemGettingArea]     -> ITEM_GET_AREA
    | put['itemGettingArea]      -> ITEM_GET_AREA
    | supplyItem                  -> ITEM_GET_AREA
    | getItem                    -> ITEM_GET_AREA
).

PACK_AREA = (
      permitMove[ii:PodID]        -> move[ii] -> MOVEMENT[ii]
    | permitMove['shippingArea]   -> move['shippingArea] -> SHIP_AREA
    | permitMove['boxGettingArea] -> move['boxGettingArea] -> BOX_GET_AREA
    | permitMove['itemGettingArea] -> move['itemGettingArea] -> ITEM_GET_AREA
    | pick['packingArea]          -> PACK_AREA
    | put['packingArea]           -> PACK_AREA
    | pack                        -> PACK_AREA
).

//------------------------------------------------------------------------------------------
// Requests
//------------------------------------------------------------------------------------------

REQUEST_1 = (
    reqReplenishItem
    -> replenish[PodID]
    -> doneReplenishItem
    -> REQUEST_1
).

REQUEST_2 = (
    reqBringItem
    -> bring[PodID]
    -> doneBringItem
    -> REQUEST_2
).

REQUEST_3 = (
    reqBringBox
    -> doneBringBox
    -> REQUEST_3
).

REQUEST_4 = (
    reqShip
    -> doneShip
    -> REQUEST_4
).

||KIVA_System = (
      ['robot1]:INITIAL(1)
    || ['robot2]:INITIAL(PodNum)
    || SHIP_AREA
    || BOX_GET_AREA
    || ITEM_GET_AREA
    || PACK_AREA
    || REQUEST_1
    || REQUEST_2
    || REQUEST_3
    || REQUEST_4
).

//*****************************************************************************
// Requirement
//*****************************************************************************

/*Fluent*/
fluent REPLENISH[i:PodID] = <replenish[i], ['robot2].move[i]>
fluent BRING[i:PodID]     = <bring[i], ['robot2].move[i]>

fluent PROCESS_REQUEST =
    <RequestActions, DoneActions>

fluent PROCESS_REPLENISH_ITEM =
    <reqReplenishItem, doneReplenishItem>

fluent PROCESS_BRING_ITEM =
    <reqBringItem, doneBringItem>

fluent PROCESS_BRING_BOX =
    <reqBringBox, doneBringBox>

fluent PROCESS_SHIP =
    <reqShip, doneShip>

fluent REQ_SHIP_TIMEING =
    <pack, doneShip>

fluent SHIP_TIMING =
    <['robot1].put['shippingArea], doneShip>

fluent BOX_TIMING =
    <['robot1].move['boxGettingArea], ['robot1].pick['boxGettingArea]>

fluent ITEM_TIMING =
    <['robot2].put['itemGettingArea], ['robot2].pick['itemGettingArea]>

fluent GET_TIMING_1 =
    <['robot2].put['packingArea], ['robot2].pick['packingArea]>

fluent GET_TIMING_2 =
    <doneBringBox, ['robot2].put['packingArea]>

fluent PACK_TIMING_1 =
    <doneBringBox, reqShip>

fluent PACK_TIMING_2 =
    <doneBringItem, reqShip>

fluent RETURN[i:PodID] =
    <['robot2].pick[i], ['robot2].move[i]>

/*FLTL*/

// Two Robot
assert SHIP_ROBOT_RULE =
    !{['robot1].move['itemGettingArea]}

assert REPLENISH_ROBOT_RULE =
    !{['robot2].move['shippingArea], ['robot2].move['boxGettingArea]}

assert REQUEST_TIMING_1 =
    (reqBringBox -> X(!reqShip W doneBringBox))

assert REQUEST_TIMING_2 =
    (reqShip -> X(!reqBringBox W doneShip))

assert REQUEST_TIMING_3 =
    (reqReplenishItem -> X(!reqBringItem W doneReplenishItem))

assert REQUEST_TIMING_4 =
    (reqBringItem -> X(!reqReplenishItem W doneBringItem))

// Single Robot
assert REQUEST_ONLY_1 =
    (!(PROCESS_SHIP || PROCESS_BRING_BOX)
        ->
        !{
            ['robot1].move[PodID],
            ['robot1].move[Area],
            ['robot1].pick[PodID],
            ['robot1].put[PodID],
            ['robot1].pick[Area],
            ['robot1].put[Area]
        })

assert REQUEST_ONLY_2 =
    (!(PROCESS_REPLENISH_ITEM || PROCESS_BRING_ITEM)
        ->
        !{
            ['robot2].move[PodID],
            ['robot2].move[Area],
            ['robot2].pick[PodID],
            ['robot2].put[PodID],
            ['robot2].pick[Area],
            ['robot2].put[Area]
        })

assert ROBOT1_ARM_RULE_1 =
    (!(PROCESS_SHIP || PROCESS_BRING_BOX) -> !{Robot1_ArmActions})

assert ROBOT1_ARM_RULE_2 =
    ({Robot1_PickActions}
        -> X(!{Robot1_PickActions} W {Robot1_PutActions}))

assert ROBOT1_ARM_RULE_3 =
    ({Robot1_PutActions}
        -> X(!{Robot1_PutActions} W {Robot1_PickActions}))

assert ROBOT2_ARM_RULE_1 =
    (!(PROCESS_REPLENISH_ITEM || PROCESS_BRING_ITEM) -> !{Robot2_ArmActions})

assert ROBOT2_ARM_RULE_2 =
    ({Robot2_PickActions}
        -> X(!{Robot2_PickActions} W {Robot2_PutActions}))

assert ROBOT2_ARM_RULE_3 =
    ({Robot2_PutActions}
        -> X(!{Robot2_PutActions} W {Robot2_PickActions}))

// Replenish
assert REPLENISH_RULE_1 =
    (REPLENISH[PodID] -> !['robot2].move[PodID])

assert REPLENISH_RULE_2 =
    (replenish[PodID]
        -> X(!{{ReplenishRobotActions}\{['robot2].move[PodID]}}
             W ['robot2].move[PodID]))

assert REPLENISH_RULE01 =
    (REPLENISH[1] -> !['robot2].move[PodID])

assert REPLENISH_RULE02 =
    (REPLENISH[2] -> !['robot2].move[PodID])

// Bring
assert BRING_RULE_1 =
    (BRING[PodID] -> !['robot2].move[PodID])

assert BRING_RULE_2 =
    (bring[PodID]
        -> X(!{{ReplenishRobotActions}\{['robot2].move[PodID]}}
             W ['robot2].move[PodID]))

assert BRING_RULE01 =
    (BRING[1] -> !['robot2].move[PodID])

assert BRING_RULE02 =
    (BRING[2] -> !['robot2].move[PodID])

// Area
assert AREA_RULE_1 =
    (!SHIP_TIMING -> !ship)

assert AREA_RULE_2 =
    (!BOX_TIMING -> !supplyBox)

assert AREA_RULE_3 =
    (!ITEM_TIMING -> !supplyItem)

assert AREA_RULE_4 =
    (!(GET_TIMING_1 && !GET_TIMING_2 && PROCESS_BRING_ITEM)
        -> !getItem)

assert AREA_RULE_5 =
    (!PACK_TIMING_1 -> !pack)

assert AREA_RULE_6 =
    (!PACK_TIMING_2 -> !pack)

assert AREA_RULE_7 =
    (pack -> X(!pack W doneBringBox))

assert AREA_RULE_8 =
    (pack -> X(!pack W doneBringItem))

// Replenish item
set ReplenishItemActions = {
    reqReplenishItem,
    replenish[PodID],
    ['robot2].move[PodID],
    ['robot2].pick[PodID],
    ['robot2].move['itemGettingArea],
    ['robot2].put['itemGettingArea],
    supplyItem,
    ['robot2].pick['itemGettingArea],
    ['robot2].put[PodID],
    doneReplenishItem
}

assert REPLENISH_ITEM_RULE_1 =
    (PROCESS_REPLENISH_ITEM
        -> !{{ReplenishRobotActions}\{ReplenishItemActions}})

assert REPLENISH_ITEM_RULE_2 =
    (reqReplenishItem
        -> X(!{{ReplenishRobotActions}\{replenish[PodID]}}
             W replenish[PodID]))

assert REPLENISH_ITEM_RULE_3 =
    (reqReplenishItem
        -> X(!{['robot2].pick[PodID], ['robot2].pick[Area]}
             W ['robot2].move[PodID]))

assert REPLENISH_ITEM_RULE_4 =
    ((PROCESS_REPLENISH_ITEM && ['robot2].move[PodID])
        ->
        X(!{['robot2].move[PodID], ['robot2].move[Area]}
          W {['robot2].pick[PodID], ['robot2].put[PodID]}))

assert REPLENISH_ITEM_RULE_5 =
    (reqReplenishItem
        -> X(!['robot2].put[PodID] W ['robot2].pick[PodID]))

assert REPLENISH_ITEM_RULE_6 =
    ((PROCESS_REPLENISH_ITEM && ['robot2].pick[PodID])
        ->
        X(!{{ReplenishRobotActions}\{['robot2].move['itemGettingArea]}}
          W ['robot2].move['itemGettingArea]))

assert REPLENISH_ITEM_RULE_7 =
    (['robot2].move['itemGettingArea]
        -> X(!['robot2].move[PodID]
             W ['robot2].pick['itemGettingArea]))

assert REPLENISH_ITEM_RULE_8 =
    (['robot2].move['itemGettingArea]
        -> X(!['robot2].pick['itemGettingArea]
             W supplyItem))

assert REPLENISH_ITEM_RULE_9 =
    (supplyItem -> X(!supplyItem W doneReplenishItem))

assert REPLENISH_ITEM_RULE_10 =
    (['robot2].pick['itemGettingArea]
        -> X(!['robot2].put['itemGettingArea]
             W doneReplenishItem))

assert REPLENISH_ITEM_RULE_11 =
    ((PROCESS_REPLENISH_ITEM && RETURN[PodID])
        -> !['robot2].move[PodID])

assert REPLENISH_ITEM_RULE_12 =
    ((PROCESS_REPLENISH_ITEM && ['robot2].put[PodID])
        ->
        X(!{{ReplenishRobotActions}\{doneReplenishItem}}
          W doneReplenishItem))

assert REPLENISH_ITEM_RULE_13 =
    ((PROCESS_REPLENISH_ITEM && reqReplenishItem)
        -> X(!doneReplenishItem W ['robot2].put[PodID]))

// Explicit pod-specific return restrictions
assert REPLENISH_ITEM_RULE01_11 =
    (replenish[1]
        -> X(!{['robot2].move[PodID]}\{['robot2].move[1]}
             W doneReplenishItem))

assert REPLENISH_ITEM_RULE02_11 =
    (replenish[2]
        -> X(!{['robot2].move[PodID]}\{['robot2].move[2]}
             W doneReplenishItem))

// Bring item
set BringItemActions = {
    reqBringItem,
    bring[PodID],
    ['robot2].move[PodID],
    ['robot2].pick[PodID],
    ['robot2].move['packingArea],
    ['robot2].put['packingArea],
    getItem,
    ['robot2].pick['packingArea],
    ['robot2].put[PodID],
    doneBringItem
}

assert BRING_ITEM_RULE_1 =
    (PROCESS_BRING_ITEM
        -> !{{ReplenishRobotActions}\{BringItemActions}})

assert BRING_ITEM_RULE_2 =
    (reqBringItem
        -> X(!{{ReplenishRobotActions}\{bring[PodID]}}
             W bring[PodID]))

assert BRING_ITEM_RULE_3 =
    (reqBringItem
        -> X(!{['robot2].pick[PodID], ['robot2].pick[Area]}
             W ['robot2].move[PodID]))

assert BRING_ITEM_RULE_4 =
    ((PROCESS_BRING_ITEM && ['robot2].move[PodID])
        ->
        X(!{['robot2].move[PodID], ['robot2].move[Area]}
          W {['robot2].pick[PodID], ['robot2].put[PodID]}))

assert BRING_ITEM_RULE_5 =
    (reqBringItem
        -> X(!['robot2].put[PodID] W ['robot2].pick[PodID]))

assert BRING_ITEM_RULE_6 =
    ((PROCESS_BRING_ITEM && ['robot2].pick[PodID])
        ->
        X(!{{ReplenishRobotActions}\{['robot2].move['packingArea]}}
          W ['robot2].move['packingArea]))

assert BRING_ITEM_RULE_7 =
    ((PROCESS_BRING_ITEM && ['robot2].move['packingArea])
        -> X(!['robot2].move[PodID]
             W ['robot2].pick['packingArea]))

assert BRING_ITEM_RULE_8 =
    ((PROCESS_BRING_ITEM && ['robot2].move['packingArea])
        -> X(!['robot2].pick['packingArea]
             W getItem))

assert BRING_ITEM_RULE_9 =
    (getItem -> X(!getItem W doneBringItem))

assert BRING_ITEM_RULE_10 =
    ((PROCESS_BRING_ITEM && ['robot2].pick['packingArea])
        -> X(!['robot2].put['packingArea]
             W doneBringItem))

assert BRING_ITEM_RULE_11 =
    ((PROCESS_BRING_ITEM && RETURN[PodID])
        -> !['robot2].move[PodID])

assert BRING_ITEM_RULE_12 =
    ((PROCESS_BRING_ITEM && ['robot2].put[PodID])
        ->
        X(!{{ReplenishRobotActions}\{doneBringItem}}
          W doneBringItem))

assert BRING_ITEM_RULE_13 =
    ((PROCESS_BRING_ITEM && reqBringItem)
        -> X(!doneBringItem W ['robot2].put[PodID]))

assert BRING_ITEM_RULE01_11 =
    (bring[1]
        -> X(!{['robot2].move[PodID]}\{['robot2].move[1]}
             W doneBringItem))

assert BRING_ITEM_RULE02_11 =
    (bring[2]
        -> X(!{['robot2].move[PodID]}\{['robot2].move[2]}
             W doneBringItem))

// Bring box
set BringBoxActions = {
    reqBringBox,
    ['robot1].move['boxGettingArea],
    supplyBox,
    ['robot1].pick['boxGettingArea],
    ['robot1].move['packingArea],
    ['robot1].put['packingArea],
    doneBringBox
}

assert BRING_BOX_RULE_1 =
    (PROCESS_BRING_BOX
        -> !{{ShippingRobotActions}\{BringBoxActions}})

assert BRING_BOX_RULE_2 =
    (reqBringBox
        -> X(!['robot1].move['packingArea]
             W ['robot1].pick['boxGettingArea]))

assert BRING_BOX_RULE_3 =
    (reqBringBox
        -> X(!['robot1].pick['boxGettingArea]
             W supplyBox))

assert BRING_BOX_RULE_4 =
    (supplyBox
        -> X(!supplyBox W ['robot1].pick['boxGettingArea]))

assert BRING_BOX_RULE_5 =
    (['robot1].pick['boxGettingArea]
        -> X(!['robot1].move['boxGettingArea]
             W doneBringBox))

assert BRING_BOX_RULE_6 =
    (reqBringBox
        -> X(!doneBringBox W ['robot1].put['packingArea]))

// Ship
set ShipActions = {
    reqShip,
    ['robot1].move['packingArea],
    ['robot1].pick['packingArea],
    ['robot1].move['shippingArea],
    ['robot1].put['shippingArea],
    ship,
    doneShip
}

assert SHIP_RULE_0 =
    (!REQ_SHIP_TIMEING -> !reqShip)

assert SHIP_RULE_1 =
    (PROCESS_SHIP
        -> !{{ShippingRobotActions}\{ShipActions}})

assert SHIP_RULE_2 =
    (reqShip
        -> X(!['robot1].move['shippingArea]
             W ['robot1].pick['packingArea]))

assert SHIP_RULE_3 =
    (['robot1].move['shippingArea]
        -> X(!['robot1].move['packingArea]
             W doneShip))

assert SHIP_RULE_4 =
    (reqShip
        -> X(!ship W ['robot1].put['shippingArea]))

assert SHIP_RULE_5 =
    (ship -> X(!ship W doneShip))

assert SHIP_RULE_6 =
    (reqShip -> X(!doneShip W ship))

/*LTL Property*/

ltl_property P_SHIP_ROBOT_RULE =
    [](SHIP_ROBOT_RULE)

ltl_property P_REPLENISH_ROBOT_RULE =
    [](REPLENISH_ROBOT_RULE)

ltl_property P_REQUEST_TIMING_1 =
    [](REQUEST_TIMING_1)

ltl_property P_REQUEST_TIMING_2 =
    [](REQUEST_TIMING_2)

ltl_property P_REQUEST_TIMING_3 =
    [](REQUEST_TIMING_3)

ltl_property P_REQUEST_TIMING_4 =
    [](REQUEST_TIMING_4)

ltl_property P_REQUEST_ONLY_1 =
    [](REQUEST_ONLY_1)

ltl_property P_REQUEST_ONLY_2 =
    [](REQUEST_ONLY_2)

ltl_property P_ROBOT1_ARM_RULE_1 =
    [](ROBOT1_ARM_RULE_1)

ltl_property P_ROBOT1_ARM_RULE_2 =
    [](ROBOT1_ARM_RULE_2)

ltl_property P_ROBOT1_ARM_RULE_3 =
    [](ROBOT1_ARM_RULE_3)

ltl_property P_ROBOT2_ARM_RULE_1 =
    [](ROBOT2_ARM_RULE_1)

ltl_property P_ROBOT2_ARM_RULE_2 =
    [](ROBOT2_ARM_RULE_2)

ltl_property P_ROBOT2_ARM_RULE_3 =
    [](ROBOT2_ARM_RULE_3)

ltl_property P_REPLENISH_RULE_1 =
    [](REPLENISH_RULE_1)

ltl_property P_REPLENISH_RULE_2 =
    [](REPLENISH_RULE_2)

ltl_property P_REPLENISH_RULE01 =
    [](REPLENISH_RULE01)

ltl_property P_REPLENISH_RULE02 =
    [](REPLENISH_RULE02)

ltl_property P_BRING_RULE_1 =
    [](BRING_RULE_1)

ltl_property P_BRING_RULE_2 =
    [](BRING_RULE_2)

ltl_property P_BRING_RULE01 =
    [](BRING_RULE01)

ltl_property P_BRING_RULE02 =
    [](BRING_RULE02)

ltl_property P_AREA_RULE_1 =
    [](AREA_RULE_1)

ltl_property P_AREA_RULE_2 =
    [](AREA_RULE_2)

ltl_property P_AREA_RULE_3 =
    [](AREA_RULE_3)

ltl_property P_AREA_RULE_4 =
    [](AREA_RULE_4)

ltl_property P_AREA_RULE_5 =
    [](AREA_RULE_5)

ltl_property P_AREA_RULE_6 =
    [](AREA_RULE_6)

ltl_property P_AREA_RULE_7 =
    [](AREA_RULE_7)

ltl_property P_AREA_RULE_8 =
    [](AREA_RULE_8)

ltl_property P_REPLENISH_ITEM_RULE_1 =
    [](REPLENISH_ITEM_RULE_1)

ltl_property P_REPLENISH_ITEM_RULE_2 =
    [](REPLENISH_ITEM_RULE_2)

ltl_property P_REPLENISH_ITEM_RULE_3 =
    [](REPLENISH_ITEM_RULE_3)

ltl_property P_REPLENISH_ITEM_RULE_4 =
    [](REPLENISH_ITEM_RULE_4)

ltl_property P_REPLENISH_ITEM_RULE_5 =
    [](REPLENISH_ITEM_RULE_5)

ltl_property P_REPLENISH_ITEM_RULE_6 =
    [](REPLENISH_ITEM_RULE_6)

ltl_property P_REPLENISH_ITEM_RULE_7 =
    [](REPLENISH_ITEM_RULE_7)

ltl_property P_REPLENISH_ITEM_RULE_8 =
    [](REPLENISH_ITEM_RULE_8)

ltl_property P_REPLENISH_ITEM_RULE_9 =
    [](REPLENISH_ITEM_RULE_9)

ltl_property P_REPLENISH_ITEM_RULE_10 =
    [](REPLENISH_ITEM_RULE_10)

ltl_property P_REPLENISH_ITEM_RULE_11 =
    [](REPLENISH_ITEM_RULE_11)

ltl_property P_REPLENISH_ITEM_RULE_12 =
    [](REPLENISH_ITEM_RULE_12)

ltl_property P_REPLENISH_ITEM_RULE_13 =
    [](REPLENISH_ITEM_RULE_13)

ltl_property P_BRING_ITEM_RULE_1 =
    [](BRING_ITEM_RULE_1)

ltl_property P_BRING_ITEM_RULE_2 =
    [](BRING_ITEM_RULE_2)

ltl_property P_BRING_ITEM_RULE_3 =
    [](BRING_ITEM_RULE_3)

ltl_property P_BRING_ITEM_RULE_4 =
    [](BRING_ITEM_RULE_4)

ltl_property P_BRING_ITEM_RULE_5 =
    [](BRING_ITEM_RULE_5)

ltl_property P_BRING_ITEM_RULE_6 =
    [](BRING_ITEM_RULE_6)

ltl_property P_BRING_ITEM_RULE_7 =
    [](BRING_ITEM_RULE_7)

ltl_property P_BRING_ITEM_RULE_8 =
    [](BRING_ITEM_RULE_8)

ltl_property P_BRING_ITEM_RULE_9 =
    [](BRING_ITEM_RULE_9)

ltl_property P_BRING_ITEM_RULE_10 =
    [](BRING_ITEM_RULE_10)

ltl_property P_BRING_ITEM_RULE_11 =
    [](BRING_ITEM_RULE_11)

ltl_property P_BRING_ITEM_RULE_12 =
    [](BRING_ITEM_RULE_12)

ltl_property P_BRING_ITEM_RULE_13 =
    [](BRING_ITEM_RULE_13)

ltl_property P_BRING_BOX_RULE_1 =
    [](BRING_BOX_RULE_1)

ltl_property P_BRING_BOX_RULE_2 =
    [](BRING_BOX_RULE_2)

ltl_property P_BRING_BOX_RULE_3 =
    [](BRING_BOX_RULE_3)

ltl_property P_BRING_BOX_RULE_4 =
    [](BRING_BOX_RULE_4)

ltl_property P_BRING_BOX_RULE_5 =
    [](BRING_BOX_RULE_5)

ltl_property P_BRING_BOX_RULE_6 =
    [](BRING_BOX_RULE_6)

ltl_property P_SHIP_RULE_0 =
    [](SHIP_RULE_0)

ltl_property P_SHIP_RULE_1 =
    [](SHIP_RULE_1)

ltl_property P_SHIP_RULE_2 =
    [](SHIP_RULE_2)

ltl_property P_SHIP_RULE_3 =
    [](SHIP_RULE_3)

ltl_property P_SHIP_RULE_4 =
    [](SHIP_RULE_4)

ltl_property P_SHIP_RULE_5 =
    [](SHIP_RULE_5)

ltl_property P_SHIP_RULE_6 =
    [](SHIP_RULE_6)

//*****************************************************************************
// Collision monitor
//*****************************************************************************

set R1_Moves = {
    ['robot1].move[PodID],
    ['robot1].move[Area]
}

set R2_Moves = {
    ['robot2].move[PodID],
    ['robot2].move[Area]
}

fluent AT_R1_POD[p:PodID] =
    <['robot1].move[p],
     R1_Moves\{['robot1].move[p]}>

fluent AT_R1_AREA[a:Area] =
    <['robot1].move[a],
     R1_Moves\{['robot1].move[a]}>

fluent AT_R2_POD[p:PodID] =
    <['robot2].move[p],
     R2_Moves\{['robot2].move[p]}>

fluent AT_R2_AREA[a:Area] =
    <['robot2].move[a],
     R2_Moves\{['robot2].move[a]}>

assert NO_COLLISION = (
    forall[p:PodID]
        []!(AT_R1_POD[p] && AT_R2_POD[p])
    &&
    forall[a:Area]
        []!(AT_R1_AREA[a] && AT_R2_AREA[a])
)

ltl_property P_NO_COLLISION =
    NO_COLLISION

//*****************************************************************************
// Controller Specification
//*****************************************************************************

controllerSpec Specification =
{
    safety = {
        P_REPLENISH_ROBOT_RULE,
        P_SHIP_ROBOT_RULE,

        P_REQUEST_TIMING_1,
        P_REQUEST_TIMING_2,
        P_REQUEST_TIMING_3,
        P_REQUEST_TIMING_4,

        P_REQUEST_ONLY_1,
        P_REQUEST_ONLY_2,

        P_ROBOT1_ARM_RULE_1,
        P_ROBOT1_ARM_RULE_2,
        P_ROBOT1_ARM_RULE_3,

        P_ROBOT2_ARM_RULE_1,
        P_ROBOT2_ARM_RULE_2,
        P_ROBOT2_ARM_RULE_3,

        P_REPLENISH_RULE_2,
        P_BRING_RULE_2,

        P_AREA_RULE_1,
        P_AREA_RULE_2,
        P_AREA_RULE_3,
        P_AREA_RULE_4,
        P_AREA_RULE_5,
        P_AREA_RULE_6,
        P_AREA_RULE_7,
        P_AREA_RULE_8,

        P_REPLENISH_ITEM_RULE_1,
        P_REPLENISH_ITEM_RULE_2,
        P_REPLENISH_ITEM_RULE_3,
        P_REPLENISH_ITEM_RULE_4,
        P_REPLENISH_ITEM_RULE_5,
        P_REPLENISH_ITEM_RULE_6,
        P_REPLENISH_ITEM_RULE_7,
        P_REPLENISH_ITEM_RULE_8,
        P_REPLENISH_ITEM_RULE_9,
        P_REPLENISH_ITEM_RULE_10,
        P_REPLENISH_ITEM_RULE_12,
        P_REPLENISH_ITEM_RULE_13,

        P_BRING_ITEM_RULE_1,
        P_BRING_ITEM_RULE_2,
        P_BRING_ITEM_RULE_3,
        P_BRING_ITEM_RULE_4,
        P_BRING_ITEM_RULE_5,
        P_BRING_ITEM_RULE_6,
        P_BRING_ITEM_RULE_7,
        P_BRING_ITEM_RULE_8,
        P_BRING_ITEM_RULE_9,
        P_BRING_ITEM_RULE_10,
        P_BRING_ITEM_RULE_12,
        P_BRING_ITEM_RULE_13,

        P_BRING_BOX_RULE_1,
        P_BRING_BOX_RULE_2,
        P_BRING_BOX_RULE_3,
        P_BRING_BOX_RULE_4,
        P_BRING_BOX_RULE_5,
        P_BRING_BOX_RULE_6,

        P_SHIP_RULE_0,
        P_SHIP_RULE_1,
        P_SHIP_RULE_2,
        P_SHIP_RULE_3,
        P_SHIP_RULE_4,
        P_SHIP_RULE_5,
        P_SHIP_RULE_6,

        P_REPLENISH_RULE01,
        P_BRING_RULE01,
        P_REPLENISH_ITEM_RULE01_11,
        P_BRING_ITEM_RULE01_11,

        P_REPLENISH_RULE02,
        P_BRING_RULE02,
        P_REPLENISH_ITEM_RULE02_11,
        P_BRING_ITEM_RULE02_11,

        P_NO_COLLISION
    }

    controllable = {ControllableActions}
}

//*****************************************************************************
// Controller
//*****************************************************************************

//Traditional Synthesis
||Environment = (KIVA_System).

controller ||TraditionalController =
    (Environment)~{Specification}.

// Proposal Synthesis
// controller ||Controller =
//     (['robot1]:INITIAL(1)
//      || ['robot2]:INITIAL(PodNum)
//      || SHIP_AREA
//      || BOX_GET_AREA
//      || ITEM_GET_AREA
//      || PACK_AREA
//      || REQUEST_1
//      || REQUEST_2
//      || REQUEST_3
//      || REQUEST_4)~{Specification}.

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

// Environment-level permission/confirmation actions.
// A move can occur only after its corresponding permitMove action.
set MovePermissionActions = {
    [Robot].permitMove[PodID],
    [Robot].permitMove[Area]
}

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

set RequestActions = {
    reqReplenishItem,
    reqBringItem,
    reqBringBox,
    reqShip,
    replenish[PodID],
    bring[PodID]
}

set DoneActions = {
    doneReplenishItem,
    doneBringItem,
    doneBringBox,
    doneShip
}

set ShippingRobotActions = {
    ['robot1].move[PodID],
    ['robot1].pick[PodID],
    ['robot1].put[PodID],
    ['robot1].move[Area],
    ['robot1].pick[Area],
    ['robot1].put[Area]
}

set ReplenishRobotActions = {
    ['robot2].move[PodID],
    ['robot2].pick[PodID],
    ['robot2].put[PodID],
    ['robot2].move[Area],
    ['robot2].pick[Area],
    ['robot2].put[Area]
}

// Move itself is now an environmental/uncontrollable action.
// The controller controls the preceding permission action.
set ControllableActions = {
    MovePermissionActions,
    AreaActions,
    ArmActions,
    DoneActions,
    reqReplenishItem,
    reqBringItem,
    reqBringBox,
    reqShip
}

set MonitorableActions = {AreaActions}

set AllActions = {
    MoveActions,
    MovePermissionActions,
    AreaActions,
    ArmActions,
    RequestActions,
    DoneActions
}

//******************************************************************************************//
// Environment Model
//******************************************************************************************//

INITIAL(N = 1) = MOVEMENT[N]

//------------------------------------------------------------------------------------------
// Pod movement
//------------------------------------------------------------------------------------------

MOVEMENT[i:PodID] = (
      permitMove[ii:PodID]       -> move[ii] -> MOVEMENT[ii]
    | permitMove['shippingArea]  -> move['shippingArea]    -> SHIP_AREA
    | permitMove['boxGettingArea] -> move['boxGettingArea] -> BOX_GET_AREA
    | permitMove['itemGettingArea] -> move['itemGettingArea] -> ITEM_GET_AREA
    | permitMove['packingArea]   -> move['packingArea]     -> PACK_AREA
    | pick[i]                    -> MOVEMENT[i]
    | put[i]                     -> MOVEMENT[i]
).

//------------------------------------------------------------------------------------------
// Area states
//------------------------------------------------------------------------------------------

SHIP_AREA = (
      permitMove[ii:PodID]        -> move[ii] -> MOVEMENT[ii]
    | permitMove['boxGettingArea] -> move['boxGettingArea] -> BOX_GET_AREA
    | permitMove['itemGettingArea] -> move['itemGettingArea] -> ITEM_GET_AREA
    | permitMove['packingArea]    -> move['packingArea] -> PACK_AREA
    | pick['shippingArea]         -> SHIP_AREA
    | put['shippingArea]          -> SHIP_AREA
    | ship                        -> SHIP_AREA
).

BOX_GET_AREA = (
      permitMove[ii:PodID]         -> move[ii] -> MOVEMENT[ii]
    | permitMove['shippingArea]    -> move['shippingArea] -> SHIP_AREA
    | permitMove['itemGettingArea] -> move['itemGettingArea] -> ITEM_GET_AREA
    | permitMove['packingArea]     -> move['packingArea] -> PACK_AREA
    | pick['boxGettingArea]        -> BOX_GET_AREA
    | put['boxGettingArea]         -> BOX_GET_AREA
    | supplyBox                    -> BOX_GET_AREA
).

ITEM_GET_AREA = (
      permitMove[ii:PodID]        -> move[ii] -> MOVEMENT[ii]
    | permitMove['shippingArea]   -> move['shippingArea] -> SHIP_AREA
    | permitMove['boxGettingArea] -> move['boxGettingArea] -> BOX_GET_AREA
    | permitMove['packingArea]    -> move['packingArea] -> PACK_AREA
    | pick['itemGettingArea]     -> ITEM_GET_AREA
    | put['itemGettingArea]      -> ITEM_GET_AREA
    | supplyItem                  -> ITEM_GET_AREA
    | getItem                    -> ITEM_GET_AREA
).

PACK_AREA = (
      permitMove[ii:PodID]        -> move[ii] -> MOVEMENT[ii]
    | permitMove['shippingArea]   -> move['shippingArea] -> SHIP_AREA
    | permitMove['boxGettingArea] -> move['boxGettingArea] -> BOX_GET_AREA
    | permitMove['itemGettingArea] -> move['itemGettingArea] -> ITEM_GET_AREA
    | pick['packingArea]          -> PACK_AREA
    | put['packingArea]           -> PACK_AREA
    | pack                        -> PACK_AREA
).

//------------------------------------------------------------------------------------------
// Requests
//------------------------------------------------------------------------------------------

REQUEST_1 = (
    reqReplenishItem
    -> replenish[PodID]
    -> doneReplenishItem
    -> REQUEST_1
).

REQUEST_2 = (
    reqBringItem
    -> bring[PodID]
    -> doneBringItem
    -> REQUEST_2
).

REQUEST_3 = (
    reqBringBox
    -> doneBringBox
    -> REQUEST_3
).

REQUEST_4 = (
    reqShip
    -> doneShip
    -> REQUEST_4
).

||KIVA_System = (
      ['robot1]:INITIAL(1)
    || ['robot2]:INITIAL(PodNum)
    || SHIP_AREA
    || BOX_GET_AREA
    || ITEM_GET_AREA
    || PACK_AREA
    || REQUEST_1
    || REQUEST_2
    || REQUEST_3
    || REQUEST_4
).

//*****************************************************************************
// Requirement
//*****************************************************************************

/*Fluent*/
fluent REPLENISH[i:PodID] = <replenish[i], ['robot2].move[i]>
fluent BRING[i:PodID]     = <bring[i], ['robot2].move[i]>

fluent PROCESS_REQUEST =
    <RequestActions, DoneActions>

fluent PROCESS_REPLENISH_ITEM =
    <reqReplenishItem, doneReplenishItem>

fluent PROCESS_BRING_ITEM =
    <reqBringItem, doneBringItem>

fluent PROCESS_BRING_BOX =
    <reqBringBox, doneBringBox>

fluent PROCESS_SHIP =
    <reqShip, doneShip>

fluent REQ_SHIP_TIMEING =
    <pack, doneShip>

fluent SHIP_TIMING =
    <['robot1].put['shippingArea], doneShip>

fluent BOX_TIMING =
    <['robot1].move['boxGettingArea], ['robot1].pick['boxGettingArea]>

fluent ITEM_TIMING =
    <['robot2].put['itemGettingArea], ['robot2].pick['itemGettingArea]>

fluent GET_TIMING_1 =
    <['robot2].put['packingArea], ['robot2].pick['packingArea]>

fluent GET_TIMING_2 =
    <doneBringBox, ['robot2].put['packingArea]>

fluent PACK_TIMING_1 =
    <doneBringBox, reqShip>

fluent PACK_TIMING_2 =
    <doneBringItem, reqShip>

fluent RETURN[i:PodID] =
    <['robot2].pick[i], ['robot2].move[i]>

/*FLTL*/

// Two Robot
assert SHIP_ROBOT_RULE =
    !{['robot1].move['itemGettingArea]}

assert REPLENISH_ROBOT_RULE =
    !{['robot2].move['shippingArea], ['robot2].move['boxGettingArea]}

assert REQUEST_TIMING_1 =
    (reqBringBox -> X(!reqShip W doneBringBox))

assert REQUEST_TIMING_2 =
    (reqShip -> X(!reqBringBox W doneShip))

assert REQUEST_TIMING_3 =
    (reqReplenishItem -> X(!reqBringItem W doneReplenishItem))

assert REQUEST_TIMING_4 =
    (reqBringItem -> X(!reqReplenishItem W doneBringItem))

// Single Robot
assert REQUEST_ONLY_1 =
    (!(PROCESS_SHIP || PROCESS_BRING_BOX)
        ->
        !{
            ['robot1].move[PodID],
            ['robot1].move[Area],
            ['robot1].pick[PodID],
            ['robot1].put[PodID],
            ['robot1].pick[Area],
            ['robot1].put[Area]
        })

assert REQUEST_ONLY_2 =
    (!(PROCESS_REPLENISH_ITEM || PROCESS_BRING_ITEM)
        ->
        !{
            ['robot2].move[PodID],
            ['robot2].move[Area],
            ['robot2].pick[PodID],
            ['robot2].put[PodID],
            ['robot2].pick[Area],
            ['robot2].put[Area]
        })

assert ROBOT1_ARM_RULE_1 =
    (!(PROCESS_SHIP || PROCESS_BRING_BOX) -> !{Robot1_ArmActions})

assert ROBOT1_ARM_RULE_2 =
    ({Robot1_PickActions}
        -> X(!{Robot1_PickActions} W {Robot1_PutActions}))

assert ROBOT1_ARM_RULE_3 =
    ({Robot1_PutActions}
        -> X(!{Robot1_PutActions} W {Robot1_PickActions}))

assert ROBOT2_ARM_RULE_1 =
    (!(PROCESS_REPLENISH_ITEM || PROCESS_BRING_ITEM) -> !{Robot2_ArmActions})

assert ROBOT2_ARM_RULE_2 =
    ({Robot2_PickActions}
        -> X(!{Robot2_PickActions} W {Robot2_PutActions}))

assert ROBOT2_ARM_RULE_3 =
    ({Robot2_PutActions}
        -> X(!{Robot2_PutActions} W {Robot2_PickActions}))

// Replenish
assert REPLENISH_RULE_1 =
    (REPLENISH[PodID] -> !['robot2].move[PodID])

assert REPLENISH_RULE_2 =
    (replenish[PodID]
        -> X(!{{ReplenishRobotActions}\{['robot2].move[PodID]}}
             W ['robot2].move[PodID]))

assert REPLENISH_RULE01 =
    (REPLENISH[1] -> !['robot2].move[PodID])

assert REPLENISH_RULE02 =
    (REPLENISH[2] -> !['robot2].move[PodID])

// Bring
assert BRING_RULE_1 =
    (BRING[PodID] -> !['robot2].move[PodID])

assert BRING_RULE_2 =
    (bring[PodID]
        -> X(!{{ReplenishRobotActions}\{['robot2].move[PodID]}}
             W ['robot2].move[PodID]))

assert BRING_RULE01 =
    (BRING[1] -> !['robot2].move[PodID])

assert BRING_RULE02 =
    (BRING[2] -> !['robot2].move[PodID])

// Area
assert AREA_RULE_1 =
    (!SHIP_TIMING -> !ship)

assert AREA_RULE_2 =
    (!BOX_TIMING -> !supplyBox)

assert AREA_RULE_3 =
    (!ITEM_TIMING -> !supplyItem)

assert AREA_RULE_4 =
    (!(GET_TIMING_1 && !GET_TIMING_2 && PROCESS_BRING_ITEM)
        -> !getItem)

assert AREA_RULE_5 =
    (!PACK_TIMING_1 -> !pack)

assert AREA_RULE_6 =
    (!PACK_TIMING_2 -> !pack)

assert AREA_RULE_7 =
    (pack -> X(!pack W doneBringBox))

assert AREA_RULE_8 =
    (pack -> X(!pack W doneBringItem))

// Replenish item
set ReplenishItemActions = {
    reqReplenishItem,
    replenish[PodID],
    ['robot2].move[PodID],
    ['robot2].pick[PodID],
    ['robot2].move['itemGettingArea],
    ['robot2].put['itemGettingArea],
    supplyItem,
    ['robot2].pick['itemGettingArea],
    ['robot2].put[PodID],
    doneReplenishItem
}

assert REPLENISH_ITEM_RULE_1 =
    (PROCESS_REPLENISH_ITEM
        -> !{{ReplenishRobotActions}\{ReplenishItemActions}})

assert REPLENISH_ITEM_RULE_2 =
    (reqReplenishItem
        -> X(!{{ReplenishRobotActions}\{replenish[PodID]}}
             W replenish[PodID]))

assert REPLENISH_ITEM_RULE_3 =
    (reqReplenishItem
        -> X(!{['robot2].pick[PodID], ['robot2].pick[Area]}
             W ['robot2].move[PodID]))

assert REPLENISH_ITEM_RULE_4 =
    ((PROCESS_REPLENISH_ITEM && ['robot2].move[PodID])
        ->
        X(!{['robot2].move[PodID], ['robot2].move[Area]}
          W {['robot2].pick[PodID], ['robot2].put[PodID]}))

assert REPLENISH_ITEM_RULE_5 =
    (reqReplenishItem
        -> X(!['robot2].put[PodID] W ['robot2].pick[PodID]))

assert REPLENISH_ITEM_RULE_6 =
    ((PROCESS_REPLENISH_ITEM && ['robot2].pick[PodID])
        ->
        X(!{{ReplenishRobotActions}\{['robot2].move['itemGettingArea]}}
          W ['robot2].move['itemGettingArea]))

assert REPLENISH_ITEM_RULE_7 =
    (['robot2].move['itemGettingArea]
        -> X(!['robot2].move[PodID]
             W ['robot2].pick['itemGettingArea]))

assert REPLENISH_ITEM_RULE_8 =
    (['robot2].move['itemGettingArea]
        -> X(!['robot2].pick['itemGettingArea]
             W supplyItem))

assert REPLENISH_ITEM_RULE_9 =
    (supplyItem -> X(!supplyItem W doneReplenishItem))

assert REPLENISH_ITEM_RULE_10 =
    (['robot2].pick['itemGettingArea]
        -> X(!['robot2].put['itemGettingArea]
             W doneReplenishItem))

assert REPLENISH_ITEM_RULE_11 =
    ((PROCESS_REPLENISH_ITEM && RETURN[PodID])
        -> !['robot2].move[PodID])

assert REPLENISH_ITEM_RULE_12 =
    ((PROCESS_REPLENISH_ITEM && ['robot2].put[PodID])
        ->
        X(!{{ReplenishRobotActions}\{doneReplenishItem}}
          W doneReplenishItem))

assert REPLENISH_ITEM_RULE_13 =
    ((PROCESS_REPLENISH_ITEM && reqReplenishItem)
        -> X(!doneReplenishItem W ['robot2].put[PodID]))

// Explicit pod-specific return restrictions
assert REPLENISH_ITEM_RULE01_11 =
    (replenish[1]
        -> X(!{['robot2].move[PodID]}\{['robot2].move[1]}
             W doneReplenishItem))

assert REPLENISH_ITEM_RULE02_11 =
    (replenish[2]
        -> X(!{['robot2].move[PodID]}\{['robot2].move[2]}
             W doneReplenishItem))

// Bring item
set BringItemActions = {
    reqBringItem,
    bring[PodID],
    ['robot2].move[PodID],
    ['robot2].pick[PodID],
    ['robot2].move['packingArea],
    ['robot2].put['packingArea],
    getItem,
    ['robot2].pick['packingArea],
    ['robot2].put[PodID],
    doneBringItem
}

assert BRING_ITEM_RULE_1 =
    (PROCESS_BRING_ITEM
        -> !{{ReplenishRobotActions}\{BringItemActions}})

assert BRING_ITEM_RULE_2 =
    (reqBringItem
        -> X(!{{ReplenishRobotActions}\{bring[PodID]}}
             W bring[PodID]))

assert BRING_ITEM_RULE_3 =
    (reqBringItem
        -> X(!{['robot2].pick[PodID], ['robot2].pick[Area]}
             W ['robot2].move[PodID]))

assert BRING_ITEM_RULE_4 =
    ((PROCESS_BRING_ITEM && ['robot2].move[PodID])
        ->
        X(!{['robot2].move[PodID], ['robot2].move[Area]}
          W {['robot2].pick[PodID], ['robot2].put[PodID]}))

assert BRING_ITEM_RULE_5 =
    (reqBringItem
        -> X(!['robot2].put[PodID] W ['robot2].pick[PodID]))

assert BRING_ITEM_RULE_6 =
    ((PROCESS_BRING_ITEM && ['robot2].pick[PodID])
        ->
        X(!{{ReplenishRobotActions}\{['robot2].move['packingArea]}}
          W ['robot2].move['packingArea]))

assert BRING_ITEM_RULE_7 =
    ((PROCESS_BRING_ITEM && ['robot2].move['packingArea])
        -> X(!['robot2].move[PodID]
             W ['robot2].pick['packingArea]))

assert BRING_ITEM_RULE_8 =
    ((PROCESS_BRING_ITEM && ['robot2].move['packingArea])
        -> X(!['robot2].pick['packingArea]
             W getItem))

assert BRING_ITEM_RULE_9 =
    (getItem -> X(!getItem W doneBringItem))

assert BRING_ITEM_RULE_10 =
    ((PROCESS_BRING_ITEM && ['robot2].pick['packingArea])
        -> X(!['robot2].put['packingArea]
             W doneBringItem))

assert BRING_ITEM_RULE_11 =
    ((PROCESS_BRING_ITEM && RETURN[PodID])
        -> !['robot2].move[PodID])

assert BRING_ITEM_RULE_12 =
    ((PROCESS_BRING_ITEM && ['robot2].put[PodID])
        ->
        X(!{{ReplenishRobotActions}\{doneBringItem}}
          W doneBringItem))

assert BRING_ITEM_RULE_13 =
    ((PROCESS_BRING_ITEM && reqBringItem)
        -> X(!doneBringItem W ['robot2].put[PodID]))

assert BRING_ITEM_RULE01_11 =
    (bring[1]
        -> X(!{['robot2].move[PodID]}\{['robot2].move[1]}
             W doneBringItem))

assert BRING_ITEM_RULE02_11 =
    (bring[2]
        -> X(!{['robot2].move[PodID]}\{['robot2].move[2]}
             W doneBringItem))

// Bring box
set BringBoxActions = {
    reqBringBox,
    ['robot1].move['boxGettingArea],
    supplyBox,
    ['robot1].pick['boxGettingArea],
    ['robot1].move['packingArea],
    ['robot1].put['packingArea],
    doneBringBox
}

assert BRING_BOX_RULE_1 =
    (PROCESS_BRING_BOX
        -> !{{ShippingRobotActions}\{BringBoxActions}})

assert BRING_BOX_RULE_2 =
    (reqBringBox
        -> X(!['robot1].move['packingArea]
             W ['robot1].pick['boxGettingArea]))

assert BRING_BOX_RULE_3 =
    (reqBringBox
        -> X(!['robot1].pick['boxGettingArea]
             W supplyBox))

assert BRING_BOX_RULE_4 =
    (supplyBox
        -> X(!supplyBox W ['robot1].pick['boxGettingArea]))

assert BRING_BOX_RULE_5 =
    (['robot1].pick['boxGettingArea]
        -> X(!['robot1].move['boxGettingArea]
             W doneBringBox))

assert BRING_BOX_RULE_6 =
    (reqBringBox
        -> X(!doneBringBox W ['robot1].put['packingArea]))

// Ship
set ShipActions = {
    reqShip,
    ['robot1].move['packingArea],
    ['robot1].pick['packingArea],
    ['robot1].move['shippingArea],
    ['robot1].put['shippingArea],
    ship,
    doneShip
}

assert SHIP_RULE_0 =
    (!REQ_SHIP_TIMEING -> !reqShip)

assert SHIP_RULE_1 =
    (PROCESS_SHIP
        -> !{{ShippingRobotActions}\{ShipActions}})

assert SHIP_RULE_2 =
    (reqShip
        -> X(!['robot1].move['shippingArea]
             W ['robot1].pick['packingArea]))

assert SHIP_RULE_3 =
    (['robot1].move['shippingArea]
        -> X(!['robot1].move['packingArea]
             W doneShip))

assert SHIP_RULE_4 =
    (reqShip
        -> X(!ship W ['robot1].put['shippingArea]))

assert SHIP_RULE_5 =
    (ship -> X(!ship W doneShip))

assert SHIP_RULE_6 =
    (reqShip -> X(!doneShip W ship))

/*LTL Property*/

ltl_property P_SHIP_ROBOT_RULE =
    [](SHIP_ROBOT_RULE)

ltl_property P_REPLENISH_ROBOT_RULE =
    [](REPLENISH_ROBOT_RULE)

ltl_property P_REQUEST_TIMING_1 =
    [](REQUEST_TIMING_1)

ltl_property P_REQUEST_TIMING_2 =
    [](REQUEST_TIMING_2)

ltl_property P_REQUEST_TIMING_3 =
    [](REQUEST_TIMING_3)

ltl_property P_REQUEST_TIMING_4 =
    [](REQUEST_TIMING_4)

ltl_property P_REQUEST_ONLY_1 =
    [](REQUEST_ONLY_1)

ltl_property P_REQUEST_ONLY_2 =
    [](REQUEST_ONLY_2)

ltl_property P_ROBOT1_ARM_RULE_1 =
    [](ROBOT1_ARM_RULE_1)

ltl_property P_ROBOT1_ARM_RULE_2 =
    [](ROBOT1_ARM_RULE_2)

ltl_property P_ROBOT1_ARM_RULE_3 =
    [](ROBOT1_ARM_RULE_3)

ltl_property P_ROBOT2_ARM_RULE_1 =
    [](ROBOT2_ARM_RULE_1)

ltl_property P_ROBOT2_ARM_RULE_2 =
    [](ROBOT2_ARM_RULE_2)

ltl_property P_ROBOT2_ARM_RULE_3 =
    [](ROBOT2_ARM_RULE_3)

ltl_property P_REPLENISH_RULE_1 =
    [](REPLENISH_RULE_1)

ltl_property P_REPLENISH_RULE_2 =
    [](REPLENISH_RULE_2)

ltl_property P_REPLENISH_RULE01 =
    [](REPLENISH_RULE01)

ltl_property P_REPLENISH_RULE02 =
    [](REPLENISH_RULE02)

ltl_property P_BRING_RULE_1 =
    [](BRING_RULE_1)

ltl_property P_BRING_RULE_2 =
    [](BRING_RULE_2)

ltl_property P_BRING_RULE01 =
    [](BRING_RULE01)

ltl_property P_BRING_RULE02 =
    [](BRING_RULE02)

ltl_property P_AREA_RULE_1 =
    [](AREA_RULE_1)

ltl_property P_AREA_RULE_2 =
    [](AREA_RULE_2)

ltl_property P_AREA_RULE_3 =
    [](AREA_RULE_3)

ltl_property P_AREA_RULE_4 =
    [](AREA_RULE_4)

ltl_property P_AREA_RULE_5 =
    [](AREA_RULE_5)

ltl_property P_AREA_RULE_6 =
    [](AREA_RULE_6)

ltl_property P_AREA_RULE_7 =
    [](AREA_RULE_7)

ltl_property P_AREA_RULE_8 =
    [](AREA_RULE_8)

ltl_property P_REPLENISH_ITEM_RULE_1 =
    [](REPLENISH_ITEM_RULE_1)

ltl_property P_REPLENISH_ITEM_RULE_2 =
    [](REPLENISH_ITEM_RULE_2)

ltl_property P_REPLENISH_ITEM_RULE_3 =
    [](REPLENISH_ITEM_RULE_3)

ltl_property P_REPLENISH_ITEM_RULE_4 =
    [](REPLENISH_ITEM_RULE_4)

ltl_property P_REPLENISH_ITEM_RULE_5 =
    [](REPLENISH_ITEM_RULE_5)

ltl_property P_REPLENISH_ITEM_RULE_6 =
    [](REPLENISH_ITEM_RULE_6)

ltl_property P_REPLENISH_ITEM_RULE_7 =
    [](REPLENISH_ITEM_RULE_7)

ltl_property P_REPLENISH_ITEM_RULE_8 =
    [](REPLENISH_ITEM_RULE_8)

ltl_property P_REPLENISH_ITEM_RULE_9 =
    [](REPLENISH_ITEM_RULE_9)

ltl_property P_REPLENISH_ITEM_RULE_10 =
    [](REPLENISH_ITEM_RULE_10)

ltl_property P_REPLENISH_ITEM_RULE_11 =
    [](REPLENISH_ITEM_RULE_11)

ltl_property P_REPLENISH_ITEM_RULE_12 =
    [](REPLENISH_ITEM_RULE_12)

ltl_property P_REPLENISH_ITEM_RULE_13 =
    [](REPLENISH_ITEM_RULE_13)

ltl_property P_BRING_ITEM_RULE_1 =
    [](BRING_ITEM_RULE_1)

ltl_property P_BRING_ITEM_RULE_2 =
    [](BRING_ITEM_RULE_2)

ltl_property P_BRING_ITEM_RULE_3 =
    [](BRING_ITEM_RULE_3)

ltl_property P_BRING_ITEM_RULE_4 =
    [](BRING_ITEM_RULE_4)

ltl_property P_BRING_ITEM_RULE_5 =
    [](BRING_ITEM_RULE_5)

ltl_property P_BRING_ITEM_RULE_6 =
    [](BRING_ITEM_RULE_6)

ltl_property P_BRING_ITEM_RULE_7 =
    [](BRING_ITEM_RULE_7)

ltl_property P_BRING_ITEM_RULE_8 =
    [](BRING_ITEM_RULE_8)

ltl_property P_BRING_ITEM_RULE_9 =
    [](BRING_ITEM_RULE_9)

ltl_property P_BRING_ITEM_RULE_10 =
    [](BRING_ITEM_RULE_10)

ltl_property P_BRING_ITEM_RULE_11 =
    [](BRING_ITEM_RULE_11)

ltl_property P_BRING_ITEM_RULE_12 =
    [](BRING_ITEM_RULE_12)

ltl_property P_BRING_ITEM_RULE_13 =
    [](BRING_ITEM_RULE_13)

ltl_property P_BRING_BOX_RULE_1 =
    [](BRING_BOX_RULE_1)

ltl_property P_BRING_BOX_RULE_2 =
    [](BRING_BOX_RULE_2)

ltl_property P_BRING_BOX_RULE_3 =
    [](BRING_BOX_RULE_3)

ltl_property P_BRING_BOX_RULE_4 =
    [](BRING_BOX_RULE_4)

ltl_property P_BRING_BOX_RULE_5 =
    [](BRING_BOX_RULE_5)

ltl_property P_BRING_BOX_RULE_6 =
    [](BRING_BOX_RULE_6)

ltl_property P_SHIP_RULE_0 =
    [](SHIP_RULE_0)

ltl_property P_SHIP_RULE_1 =
    [](SHIP_RULE_1)

ltl_property P_SHIP_RULE_2 =
    [](SHIP_RULE_2)

ltl_property P_SHIP_RULE_3 =
    [](SHIP_RULE_3)

ltl_property P_SHIP_RULE_4 =
    [](SHIP_RULE_4)

ltl_property P_SHIP_RULE_5 =
    [](SHIP_RULE_5)

ltl_property P_SHIP_RULE_6 =
    [](SHIP_RULE_6)

//*****************************************************************************
// Collision monitor
//*****************************************************************************

set R1_Moves = {
    ['robot1].move[PodID],
    ['robot1].move[Area]
}

set R2_Moves = {
    ['robot2].move[PodID],
    ['robot2].move[Area]
}

fluent AT_R1_POD[p:PodID] =
    <['robot1].move[p],
     R1_Moves\{['robot1].move[p]}>

fluent AT_R1_AREA[a:Area] =
    <['robot1].move[a],
     R1_Moves\{['robot1].move[a]}>

fluent AT_R2_POD[p:PodID] =
    <['robot2].move[p],
     R2_Moves\{['robot2].move[p]}>

fluent AT_R2_AREA[a:Area] =
    <['robot2].move[a],
     R2_Moves\{['robot2].move[a]}>

assert NO_COLLISION = (
    forall[p:PodID]
        []!(AT_R1_POD[p] && AT_R2_POD[p])
    &&
    forall[a:Area]
        []!(AT_R1_AREA[a] && AT_R2_AREA[a])
)

ltl_property P_NO_COLLISION =
    NO_COLLISION

//*****************************************************************************
// Controller Specification
//*****************************************************************************

controllerSpec Specification =
{
    safety = {
        P_REPLENISH_ROBOT_RULE,
        P_SHIP_ROBOT_RULE,

        P_REQUEST_TIMING_1,
        P_REQUEST_TIMING_2,
        P_REQUEST_TIMING_3,
        P_REQUEST_TIMING_4,

        P_REQUEST_ONLY_1,
        P_REQUEST_ONLY_2,

        P_ROBOT1_ARM_RULE_1,
        P_ROBOT1_ARM_RULE_2,
        P_ROBOT1_ARM_RULE_3,

        P_ROBOT2_ARM_RULE_1,
        P_ROBOT2_ARM_RULE_2,
        P_ROBOT2_ARM_RULE_3,

        P_REPLENISH_RULE_2,
        P_BRING_RULE_2,

        P_AREA_RULE_1,
        P_AREA_RULE_2,
        P_AREA_RULE_3,
        P_AREA_RULE_4,
        P_AREA_RULE_5,
        P_AREA_RULE_6,
        P_AREA_RULE_7,
        P_AREA_RULE_8,

        P_REPLENISH_ITEM_RULE_1,
        P_REPLENISH_ITEM_RULE_2,
        P_REPLENISH_ITEM_RULE_3,
        P_REPLENISH_ITEM_RULE_4,
        P_REPLENISH_ITEM_RULE_5,
        P_REPLENISH_ITEM_RULE_6,
        P_REPLENISH_ITEM_RULE_7,
        P_REPLENISH_ITEM_RULE_8,
        P_REPLENISH_ITEM_RULE_9,
        P_REPLENISH_ITEM_RULE_10,
        P_REPLENISH_ITEM_RULE_12,
        P_REPLENISH_ITEM_RULE_13,

        P_BRING_ITEM_RULE_1,
        P_BRING_ITEM_RULE_2,
        P_BRING_ITEM_RULE_3,
        P_BRING_ITEM_RULE_4,
        P_BRING_ITEM_RULE_5,
        P_BRING_ITEM_RULE_6,
        P_BRING_ITEM_RULE_7,
        P_BRING_ITEM_RULE_8,
        P_BRING_ITEM_RULE_9,
        P_BRING_ITEM_RULE_10,
        P_BRING_ITEM_RULE_12,
        P_BRING_ITEM_RULE_13,

        P_BRING_BOX_RULE_1,
        P_BRING_BOX_RULE_2,
        P_BRING_BOX_RULE_3,
        P_BRING_BOX_RULE_4,
        P_BRING_BOX_RULE_5,
        P_BRING_BOX_RULE_6,

        P_SHIP_RULE_0,
        P_SHIP_RULE_1,
        P_SHIP_RULE_2,
        P_SHIP_RULE_3,
        P_SHIP_RULE_4,
        P_SHIP_RULE_5,
        P_SHIP_RULE_6,

        P_REPLENISH_RULE01,
        P_BRING_RULE01,
        P_REPLENISH_ITEM_RULE01_11,
        P_BRING_ITEM_RULE01_11,

        P_REPLENISH_RULE02,
        P_BRING_RULE02,
        P_REPLENISH_ITEM_RULE02_11,
        P_BRING_ITEM_RULE02_11,

        P_NO_COLLISION
    }

    controllable = {ControllableActions}
}

//*****************************************************************************
// Controller
//*****************************************************************************

//Traditional Synthesis
||Environment = (KIVA_System).

controller ||TraditionalController =
    (Environment)~{Specification}.

// Proposal Synthesis
// controller ||Controller =
//     (['robot1]:INITIAL(1)
//      || ['robot2]:INITIAL(PodNum)
//      || SHIP_AREA
//      || BOX_GET_AREA
//      || ITEM_GET_AREA
//      || PACK_AREA
//      || REQUEST_1
//      || REQUEST_2
//      || REQUEST_3
//      || REQUEST_4)~{Specification}.