# Summary

Improve synthetic departure and arrival accuracy for adhoc flights.

# Motivation

Recent policy to make adhoc flights visible on our flight tracking webpage has
revealed an issue that we have long suspected.  There are recurring 
instances where we observe synthtic departure issued late or syntheic arrival
issued much early for many adhoc flights. This manifests as tracks beginning
or ending several miles from the origin or destination.

Prime examples of these problems are reported in [FT-5221](https://flightaware.atlassian.net/browse/FT-5221), 
[FT-6081](https://flightaware.atlassian.net/browse/FT-6081), [FT-6236](https://flightaware.atlassian.net/browse/FT-6236), [FT-6297](https://flightaware.atlassian.net/browse/FT-6297), and [FT-6753](https://flightaware.atlassian.net/browse/FT-6753).  

# Proposed Solutions

The following proposed solutions to address late departure and early arrival 
issues are broken into two phases as follows:
* Phase 1
    * Improving groundspeed handling
    * Improving altitude handling
    * Improving airport determination
    * Encapulating airground determination logics
* Phase 2
    * Processing ground-positiona and position messages from surfacestream feed
    * Persisting rejecting position message for later evaluation


## Phase 1

### Improving Groundspeed Handling

1. Add sanity check for rejecting obvious invalid values

2. Add sanity check for rejecting groundspeed from non-reliable sources

3. Improve airground determination with groundspeed information in addition
to existing distance and difference in elevation checks.  

Current logics to determine airground status of a position in the proc 
`refine_airground_switch` are:
https://github.flightaware.com/flightaware/feedstream_server/blob/e98c4a7c94c1a1e6449a645d3806b2ec247f1e8b/feed_interpreter/package/process_position_for_flightplan_forks.tcl#L1732-L1774

We update or set the airground field to G (ground) if the following rules are
met. Otherwise, leave airground field in position message as is.  
1. elevation difference between position and airport <= 400* or 500*, and
2. distance between position and airport <= 3 miles

*Notes: dependent on the groundspeed or true airspeed, the maximum elevation 
difference is 400ft if groundspeedd <= 100knots or true airspeed < 50knots; 
otherwise, it is set at 500ft.

Similar to `tita-discriminator`, we should modify `refine_airground_switch` 
proc to include the following new rules for setting airground flag to Ground:
1. `gs_src`='A' and `gs`<= 50 knots and `adsb_category` = 'A7' (helicopters)
2. `gs_src`='A' and `gs`<= 50 knots and `adsb_category` = 'B1-B6' (gliders, parachutes, ultralight, etc)
3. `gs_src`='A' and `gs`<=100 knots and `adsb_category` not ('A7', 'B1-B7') 


#### Improving Altitude Handling
1. Add sanity check for rejecting of obvious invalid altitude values. 

2. Add sanity check for rejecting altitude from unreliable data sources, 
such as MLAT.

3. The current elevation difference threshold of 400f or 500ft (see above) is
high. We should consider decreasing this threshold to a more reasonable range 
like between 100-200ft instead.  

4. Replace existing check based on `alt` value with `alt_gnss` or corrected 
altitude adjusted for pressure.  We can use the existing geodesy library from 
ADS-B team as reference on how to compute correction for `alt_gnss` values.

5. Fix bug(s) in hyperfeed that produce/emit invalid altitude values in 
`controlstream` but do not originate from `adsbparsedMux`. 

#### Improving Distance Calculation
The distance calculation in `refine_airground_switch` requires availability
of the origin and destination airport.  Currently, the airport information 
available in surfacestream is not persisted at all after being processed in 
hyperfeed for flight events.

We should be save airport data in `flightplan` data for later evaluation of 
airground status without having to invoke the expensive 
`feedstream::find_nearest_airport` proc.


#### Encapulating airground determination logics

Currently for position messages, we evaluate the airground status of the flight
at two main code paths:
* at the start when we match the position to a flight in the `tita-discrimator` 
proc, and 
* at the point when we are deciding to issue synthetic departure or arrival in the
`refine-airground-switch` proc.

In `tita-discriminator`, we use ground speed and altitude to determine the 
airground status.  While in `refine-airground-switch`, we make the same 
determination based on the distance and altitude difference between the position
current location and the airport closest to that position.  These two methods
sometimes deliver conflicting airground status leading to the late departure/
early arrival issue that we observe.  

We need to encapsulate these rules into a specific module whose purpose is to
compute airground flag for a position.  This serves as a centralized location 
where all of the identified rules reside for ease of unit testing as well as 
maintainance the rules.  This ensures consistent evaluation of the airground
status for a position throughout the hyperfeed codebase.

## Phase 2:

#### Process surface movement positions in addition to flight events

MMHF is currently ingesting surfacestream feed, however we are only interested in
the surface flight events such as taxi-start/stop, on/offblock, and avionics-off.

The surfacestream feed also contains ground-position and position messages that 
contain useful data such as airport information that could be used as origin or
destination, groundspeed, altitude, lat/lon coordinates that could hyperfeed 
could use just like a position message from ADS-B or Aireon feeds. 

The big drawback of this is it could introduce an big increase in data that
hyperfeed would need to process.  We would need evaluate MMHF performance if we
want to pursue this route.

#### Persistence of positions for later evaluation

Analysis of the flights mentioned above as use case revealed that MMHF throws away
positions once it determines that an adhoc flightplan cannot be created due to 
current airground status.  

Instead of throwing these messages way, MMHF should store these positions to perhaps
build a altitude/speed profile to aid in the determination of airground status. 
