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
    // Added controllable permission points before requestLand/extendFlight
    permitRequest[P] -> requestLand[P] -> descend[P][h:Height] -> Holding[h] |
    permitExtend[P]  -> extendFlight[P] -> End
  ),
  Holding[h:Height] =
    if (h>0) then (descend[P][h-1] -> Holding[h-1])
             else (approach[P] -> LANDING[P]),
  LANDING[P] = (land[P] -> End),
  End = (control.all -> Airplane).

ResponseMonitor = (
  requestLand[p:Plane] -> descend[p][Height] -> ResponseMonitor |
  {extendFlight[Plane],descend[Plane][Height], approach[Plane]} -> ResponseMonitor ).

//******************************************************************************
// Requirement
//******************************************************************************/

fluent PROCESS_REQUESTLAND[p:Plane] = <requestLand[p], descend[p][Height]> 

ltl_property HeightMonitor_00 = [](descend[0][0] -> X(!descend[Plane][0] W land[0]))
ltl_property HeightMonitor_01 = [](descend[0][1] -> X(!descend[Plane][1] W descend[0][Height]))
ltl_property HeightMonitor_10 = [](descend[1][0] -> X(!descend[Plane][0] W land[1]))
ltl_property HeightMonitor_11 = [](descend[1][1] -> X(!descend[Plane][1] W descend[1][Height]))

ltl_property RequestMonitor = [](requestLand[Plane] -> X(!requestLand[Plane] W land[Plane]))
ltl_property RampMonitor = [](approach[Plane] -> X(!approach[Plane] W (land[Plane])))

//******************************************************************************
// Controller
//******************************************************************************/

controllerSpec Goal = {
	safety = {
		HeightMonitor_00, HeightMonitor_01,
		HeightMonitor_10, HeightMonitor_11,
		RampMonitor,
		RequestMonitor
	}
	controllable = {
		descend[Plane][Height],
		approach[Plane],
		permitRequest[Plane],
		permitExtend[Plane]
	}
	nonblocking
}

||Environment = (forall [p:Plane] Airplane(p) || ResponseMonitor).
controller ||TraditionalController = (Environment)~{Goal}.
```

### Explanation of changes

* Added **new controllable actions** `permitRequest[p]` and `permitExtend[p]` immediately before the previously uncontrollable actions:

  * `requestLand[p]`
  * `extendFlight[p]`
* This realizes the Additional Control Record by giving the controller a **permission/synchronization point** before those environment choices occur, allowing it to coordinate them with the existing controllable `descend[p][h]` actions (specifically the cited `descend[0][0]` / `descend[1][0]` situations).
* No safety properties were changed, and all original uncontrollable actions retain their original semantics—they are only gated by new controllable permissions.
