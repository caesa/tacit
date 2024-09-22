# TACIT

[**T**ransponder **A**ctivity **C**oalesced **I**nexpenively **T**o **U**serfriendly **S**torage](https://en.wikipedia.org/wiki/Tacitus)

## Brief

 - Fetches from source (airlabs) and message serialization handled by go
 - Database inserts handled by stored procedures in [PL/PGSQL](https://www.postgresql.org/docs/current/plpgsql.html) 

## Succint formatting (TAC strips/diffs)

### TAC STRIPs

A **TAC strip** is a succint format for indicating aircraft position; 

```text
hex    utx        pnt                    alt  spd hdg   advice
0D06DE 1719087418 [-89.668579,21.048532] 1838 518 321.0 [eE190,rXA-ACK,fMX,$AMX,#2712,<MMMD,>KATL]
```

Each line of a **TAC strip** contains the full set of parameters as retrieved from the source, with `advice` parameters appended keyed by a `CHAR(1)` identifier prefix:

```text
e: equipment
r: registration
f: flag
$: airline
#: flight number
<: departure airport
>: arrival airport
```

#### "ADVICE"?

Airline and flightnumber [are part of the ADSB standard CALLSIGN field](https://mode-s.org/decode/content/ads-b/2-identification.html), but equipment, registration, flag, departure/arrival **are not**, i.e. passed to us by the data source provider. With this in mind (and with some hard experience handling the data) these fields are treated with *deep suspicion*;

 - `departure/arrival` are used only as hints during actual geo positioning, and never trusted blindly
 - `flag` can behave erratically and is only permitted to change once per day
 - `equipment/registration` are only permitted to change to non-NULL

### TAC DIFFs

Postgres outputs **TAC diffs** which include:

 - the mandatory parameters `hex`, `utx`, `pnt`, `alt`, `spd` and `hdg` 
 - an (optional) `advice` array reporting only those advisories that have changed since the last frame. Advisories that have ceased are provided as NULL (denoted by a lowercase `n`)
 - (optional) a `brk` prefix that denotes the end of one logical journey and the beginnning of another (along with the `@apt` at which the journey was split, if any)

```text
brk?  apt?  hex    utx        pnt                    alt  spd hdg   advice?
            0D06DE 1719087418 [-89.668579,21.048532] 20   50  240.0 [@MMMD,=27L]
^TKOF @MMMD 0D06DE 1719087902 [-84.639748,23.827363] 150  518 321.0 [@n,=n,$AMX,#2712,<MMMD,>KATL]
            0D06DE 1719087949 [-84.624663,23.824624] 150  519 321.0
```

**diff** will also provide 2 additional advisories:

```text
@: airport (actual, current)
=: runway  
```

### /BRK and @APT

**Logical journeys** (either *flying direct from one airport to another* **or** *taxiing within an airport*) are considered complete if they begin and end on the ground. Perfect **flight** journeys are sandwiched with a single ground frame at each end, and this frame is **repeated** at the split between the flight and its adjacent ground (taxi) journeys.

Incomplete/imperfect journeys are simply split, with no frame repetition marrying the 2 ends of adjacent journeys. The `BRK` message contains *information about this split*, and *the decision made by the Postgres application*.

#### BRK prefixes

`BRK` can be prefixed with one of the following `CHAR(1)`

 - `/` : previous journey ended on previous frame, and a new one starts here
 - `&` : previous journey ends and new begins at this frame (only ever present at `&TDWN`)
 - `^` : previous journey ends and new begins at previous frame (only ever present at `^TKOF`)
 - `-` : journey begins some frame in the past (only present in journey slices from query results)
 - `+` : journey ends some frame in the future (only present in slices from query results)

#### BRK messages

All journeys, complete or not, are stored in the database with the beginning and end `BRK` messages `brk0` and `brk1`, where `brk1` (typically) matches that of the next journey's `brk0`. Consequently any journey at all contains information about **why** the journey was split, a	nd **where** it is believed the actual split took place. 

 - `TKOF` : a transition from ground to the air (within a reasonable time frame)
 - `TDWN` : a transition from the air to the ground (within a reasonable time frame)
 - `FLEW` : as `TKOF`, but we didn't receive a frame at or near actual takeoff
 - `LAND` : as `TDWN`, ditto
 - `WALK` : aircraft is on the ground; the last time we saw it was at some other airport
 - `TRUE` : we never saw an airport, but the `arr/dep` hints agree and test as reasonable
 - `CAME` : we never saw an airport, the previous `arr` was NULL but the current `dep` tests as reasonable
 - `WENT` : we never saw an airport, the `dep` is NULL but the previous `arr` tests as reasonable
 - `DVRT` : the previous `arr` and this `dep` disagree, but `dep` tests as reasonable
 - `YOYO` : an edge-case scenario (best explained in the code)
 - `GUES` : aircraft has no `arr/dep` hints but has been missing long enough to have landed somewhere, the most likely (by landing distance and scale) was chosen
 - `VOID` : aircraft missing for 12 hours with no `arr/dep` in either cursor
 - `VACA` : aircraft missing for 24 hours, all hints ignored

#### APT messages

All the above (except `VOID` and `VACA`) will be accompanied by a non-NULL `@apt` message. But only `TKOF` and `TDWN` (and taxi journeys with `WALK`, beginning with `LAND` or ending in`FLEW`) will be marked with a `grd0/grd1` of `TRUE`; consequently `grd0/grd1` are indicators of *absolute confidence* in `apt0/apt1`. 
