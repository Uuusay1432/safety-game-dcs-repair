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
// Added permission-based synchronization for request admission.
// The controller now explicitly authorizes a landing request before the
// environment can issue requestLand[p].
// -----------------------------------------------------------------------------

Airplane(P=0) = (
    grantRequest[P] -> REQUESTED[P]),

  REQUESTED[P] = (
      requestLand[P] -> descend[P][h:Height] -> Holding[h] |
      extendFlight[P] -> End),

  Holding[h:Height] =
    if (h>0) then (descend[P][h-1] -> Holding[h-1])
             else (approach[P] -> LANDING[P]),

  LANDING[P] = (land[P] -> End),

  End = (control.all -> Airplane).

ResponseMonitor = (
  requestLand[p:Plane] -> descend[p][Height] -> ResponseMonitor |
  {extendFlight[Plane],descend[Plane][Height], approach[Plane],
   grantRequest[Plane]} -> ResponseMonitor ).

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
                    grantRequest[Plane],
                    descend[Plane][Height],
                    approach[Plane]
                   }
	nonblocking
}

||Environment = (forall [p:Plane] Airplane(p) || ResponseMonitor).

controller ||TraditionalController = (Environment)~{Goal}.
```

### Explanation of the repair

The unrealizability comes primarily from `RequestMonitor`:

* `requestLand[p]` is originally **uncontrollable**.
* After one aircraft issues `requestLand[p]`, the monitor forbids any further `requestLand[*]` until `land[p]`.
* Since another airplane may independently and uncontrollably issue `requestLand`, the controller cannot guarantee the property.

To recover synthesizability while preserving the safety requirements, the environment was revised with a minimal permission/synchronization mechanism:

* Added controllable action:

  * `grantRequest[p]`
* Each airplane must first receive `grantRequest[p]` before it may execute `requestLand[p]`.

This converts landing-request admission into a controller-mediated synchronization point while preserving:

* all original safety monitors,
* the original airplane progression structure,
* the original action names and sequencing after a request is admitted.

The controller can now serialize requests and enforce the runway/height exclusion properties consistently.
