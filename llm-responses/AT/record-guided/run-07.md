## Revised DCS Model

```fsp
//******************************************************************************
// Environment
//******************************************************************************/

const N = 2
const K = 2

const Planes  = N
const Heights = K

range Plane  = 0..(Planes-1)
range Height = 0..(Heights-1)

Airplane(P=0) = (
    requestLandPermit[P] -> requestLand[P] -> descend[P][h:Height] -> Holding[h] |
    extendFlightPermit[P] -> extendFlight[P] -> End),
  Holding[h:Height] =
    if (h>0) then (descend[P][h-1] -> Holding[h-1])
             else (approach[P] -> LANDING[P]),
  LANDING[P] = (land[P] -> End),
  End = (control.all -> Airplane).

ResponseMonitor = (
  requestLand[p:Plane] -> descend[p][Height] -> ResponseMonitor |
  {extendFlight[Plane],descend[Plane][Height], approach[Plane]}  -> ResponseMonitor ).

//******************************************************************************
// Requirement
//******************************************************************************/

fluent PROCESS_REQUESTLAND[p:Plane] = <requestLand[p], descend[p][Height]> 

ltl_property HeightMonitor_00	= [](descend[0][0] -> X(!descend[Plane][0] W land[0]))
ltl_property HeightMonitor_01	= [](descend[0][1] -> X(!descend[Plane][1] W descend[0][Height]))
ltl_property HeightMonitor_10	= [](descend[1][0] -> X(!descend[Plane][0] W land[1]))
ltl_property HeightMonitor_11	= [](descend[1][1] -> X(!descend[Plane][1] W descend[1][Height]))


ltl_property RequestMonitor   = [](requestLand[Plane] -> X(!requestLand[Plane] W land[Plane]))
ltl_property RampMonitor = [](approach[Plane] -> X(!approach[Plane] W (land[Plane])))

//******************************************************************************
// Controller
//******************************************************************************/
controllerSpec Goal = {
	safety = 	{
				HeightMonitor_00,HeightMonitor_01,
				HeightMonitor_10,HeightMonitor_11,
				RampMonitor,
				RequestMonitor
				}
	controllable = {
		descend[Plane][Height],
		approach[Plane],
		requestLandPermit[Plane],
		extendFlightPermit[Plane]
	}
	nonblocking
}

||Environment = (forall [p:Plane] Airplane(p) || ResponseMonitor).
controller ||TraditionalController = (Environment)~{Goal}.
```

### Explanation of the changes

Two new controllable actions were introduced:

* `requestLandPermit[P]`
* `extendFlightPermit[P]`

These actions are inserted immediately before the previously uncontrollable actions:

* `requestLand[P]`
* `extendFlight[P]`

This directly realizes the Additional Control Record by creating explicit permission/synchronization points before the environment can issue either landing requests or flight extensions.

The inserted controllable actions allow the controller to coordinate:

* `requestLand[P]` / `extendFlight[P]`
* with existing controllable actions:

  * `approach[P]`
  * `descend[P][0]`

as suggested by the safety-game analysis.

The original safety requirements, monitors, process structure, and uncontrollable action semantics were preserved. Only the minimal additional controllable mediation points were added.
