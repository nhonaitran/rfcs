# Summary

Improve synthetic departure and arrival accuracy for adhoc flights.

# Motivation

Recent policy to make adhoc flights visible on our website has revealed an      
issue that we have long been suspected.  There are recurring instances where we
issue synthtic departure late and/or arrival early for many adhoc flights. This
manifests as tracks beginning or ending several miles from the origin or 
destination.

Prime examples of these problems are seen here, here, here, and here.  

# Proposed Solutions

The following proposed solutions to address late departure and early arrival 
issues are broken into two phases as follows:
* Phase 1
    * Improving groundspeed handling
    * Improving altitude handling
    * Encapulating airground determination logics
* Phase 2
    * Processing ground-positiona and position messages from surfacestream feed
    * Persisting rejecting position message for later evaluation


## Phase 1

### Improving Groundspeed Handling

#### Modifications to `refine_airground_switch` proc
We will include groundspeed check similarly to that in `tita-disriminator` in 
the `refine-airground-switch`, in addition to the existing checks using 
distance and elevation difference between the airport and the position being
processed.

Current logics to determine airground status of a position in the proc 
`refined_airground_switch` are:
https://github.flightaware.com/flightaware/feedstream_server/blob/e98c4a7c94c1a1e6449a645d3806b2ec247f1e8b/feed_interpreter/package/process_position_for_flightplan_forks.tcl#L1732-L1774

We update or set the airground field to G (ground) if the following rules are
met. Otherwise, leave airground field in position message as is.  
1. elevation difference between position and airport <= 400* or 500*, and
2. distance between position and airport <= 3 miles

*Notes: dependent on the groundspeed or true airspeed, the maximum elevation 
difference is 400ft if groundspeedd <= 100knots or true airspeed < 50knots; 
otherwise, it is set at 500ft.

A new rule using groundspeed and its source is added:
1. `gs_src`='A' and `gs`<=100 knots and `adsb_category` all except 'A7'?
2. `gs_src`='A' and `gs`<= 50 knots and `adsb_category` in ('A7')


#### Improving Altitude Handling
1. The current elevation difference threshold of 400f or 500ft (see above) is
high. We should consider decreasing this threshold to a more reasonable range 
like between 100-200ft instead.  

2. Replace existing check based on `alt` value with `alt_gnss` or corrected 
altitude adjusted for pressure.  We can use the existing geodesy library from 
ADS-B team as reference on how to compute correction for `alt_gnss` values.

3. Exclude altitude values from unreliable data sources such as MLAT.
  

#### Encapulating airground determination logics

Currently for position messages, we evaluate the airground status of the flight
at two main code paths:
* at the start when we match the position to a flight in the `tita-discrimator` proc, and 
* at the point when we are deciding to issue synthetic departure or arrival in the
`refine-airground-switch` proc.

In `tita-discriminator`, we use ground speed and altitude to determine the airground
status.  While in `refine-airground-switch`, we make the same determination based on
the distance and altitude difference between the position current location and the
airport closest to that position.  These two methods sometimes deliver conflicting 
airground status leading to the late departure/early arrival issue that we observe.  

We need to encapsulate these rules into a specific module whose purpose is to
compute airground flag for a position.  This serves as a single place where all
the rules reside for ease of unit testing as well as maintaining the rules.  This
ensures consistent evaluation of the current airground status for a position
throughout the hyperfeed codebase.

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
