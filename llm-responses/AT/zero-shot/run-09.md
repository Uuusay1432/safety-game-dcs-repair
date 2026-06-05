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

// -----------------------------------------------------------------------------
// Landing permission protocol added to recover synthesizability.
// The controller explicitly grants permission before a plane may request landing.
// -----------------------------------------------------------------------------

Airplane(P=0) = (
    grantRequest[P] -> ACTIVE[P]),

ACTIVE[P] = (
    requestLand[P] -> descend[P][h:Height] -> Holding[h] |
    extendFlight[P] -> End),

Holding[h:Height] =
    if (h>0) then (descend[P][h-1] -> Holding[h-1])
             else (grantApproach[P] -> approach[P] -> LANDING[P]),

LANDING[P] = (land[P] -> End),

End = (control.all -> Airplane).

ResponseMonitor = (
  requestLand[p:Plane] -> descend[p][Height] -> ResponseMonitor |
  {extendFlight[Plane],descend[Plane][Height],
   approach[Plane],grantRequest[Plane],grantApproach[Plane]} -> ResponseMonitor ).


//******************************************************************************
// Requirement
//******************************************************************************/

fluent PROCESS_REQUESTLAND[p:Plane] = <requestLand[p], descend[p][Height]> 

ltl_property HeightMonitor_00
    = [](descend[0][0] -> X(!descend[Plane][0] W land[0]))

ltl_property HeightMonitor_01
    = [](descend[0][1] -> X(!descend[Plane][1] W descend[0][Height]))

ltl_property HeightMonitor_10
    = [](descend[1][0] -> X(!descend[Plane][0] W land[1]))

ltl_property HeightMonitor_11
    = [](descend[1][1] -> X(!descend[Plane][1] W descend[1][Height]))

ltl_property RequestMonitor
    = [](requestLand[Plane] -> X(!requestLand[Plane] W land[Plane]))

ltl_property RampMonitor
    = [](approach[Plane] -> X(!approach[Plane] W (land[Plane])))


//******************************************************************************
// Controller
//******************************************************************************/

controllerSpec Goal = {
	safety = {
		HeightMonitor_00,HeightMonitor_01,
		HeightMonitor_10,HeightMonitor_11,
		RampMonitor,
		RequestMonitor
	}
	controllable = {
		descend[Plane][Height],
		approach[Plane],
		grantRequest[Plane],
		grantApproach[Plane]
	}
	nonblocking
}

||Environment = (
    forall [p:Plane] Airplane(p)
    || ResponseMonitor
).

controller ||TraditionalController = (Environment)~{Goal}.
```

### Explanation of the repair

The unrealizability comes from the fact that the environment can autonomously issue `requestLand[p]` and later force conflicting descent/approach sequences before the controller has a way to serialize access to shared heights and the runway.

In particular:

* `RequestMonitor` forbids a second `requestLand[p]` before `land[p]`.
* The `HeightMonitor_*` properties require exclusive ownership of each altitude level until the corresponding lower transition or landing occurs.
* `RampMonitor` requires exclusive runway approach until landing.

Originally, the controller could only control `descend` and `approach`, but not the initial triggering of requests. Two airplanes could therefore simultaneously create obligations that the controller could not satisfy.

The repair introduces two explicit permission/synchronization actions:

* `grantRequest[p]` — controller authorizes plane `p` to start a landing sequence.
* `grantApproach[p]` — controller authorizes final runway approach.

This preserves all original safety properties while allowing the controller to serialize landing activity and prevent conflicting altitude/runway occupation. The original structure, actions, and branching behavior are otherwise preserved.
