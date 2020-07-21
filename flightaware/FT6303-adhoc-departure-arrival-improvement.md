# Summary

Improve synthetic departure and arrival accuracy for adhoc flights.

# Motivation

Why are we doing this? What use cases does it support? What is the expected
outcome?

Please focus on explaining the motivation so that if this RFC is not accepted,
the motivation could be used to develop alternative solutions. In other words,
enumerate the constraints you are trying to solve without coupling them too
closely to the solution you have in mind.

Recent policy to make adhoc flights visible on our website has revealed an 
issue that we have long been suspected.  There are recurring instances where
we issue synthtic departure late and/or arrival early for many adhoc flights.
Prime examples of these problems are seen here, here, here, and here.  


# Proposed Solutions

The following solutions to address late departure and early arrival issues are 
broken into two phases, outlined as follows:
* Phase 1 - short-term fixes to be implemented right away
    * Improving groundspeed handling
    * Improving altitude handling
    * Encapulating airground determination logics
* Phase 2: long-term fixes
    * Processing ground-positiona and position messages from surfacestream feed
    * Persisting rejecting position message for later evaluation


## Phase 1:

### Improving Groundspeed Handling

#### Include groundspeed check in `refine_airground_switch`
Existing logics in `refined_airground_switch` to determine the airground
status of a flight:
https://github.flightaware.com/flightaware/feedstream_server/blob/e98c4a7c94c1a1e6449a645d3806b2ec247f1e8b/feed_interpreter/package/process_position_for_flightplan_forks.tcl#L1732-L1774

We update or set the airground field to G (ground) if the following two conditions 
are met. Otherwise, leave airground field in position as is.  
1. elevation difference <= 400 or 500 *, and
2. distance to airport <= 3 miles

A new rule using groundspeed and its source is added:
3. `gs_src`='A' and `gs`<=100 knots and `adsb_category` all except 'A7'?
4. `gs_src`='A' and `gs`<= 50 knots and `adsb_category` in ('A7')


We will include groundspeed check similarly to that in `tita-disriminator` in the 
`refine-airground-switch` in addition to the existing distance and altitude delta.
The airground determination based on groundspeed is more granular with the inclusion
of the two fields, `gs_src` and `adsb_category`.  The airground breakpoint threshold
are set appropriate based on the data source and aircraft type included in the position
message.

#### Improving Altitude Handling
Based on analysis the accurracy of `alt_gnss`, the exisitng rules in altitude check
are replaced with an computed, adjusted altitude meassure, called `alt_pressured`, 
with the `alt_src` field.  Existing geodesy library can serve as good codebase to use
for computing good altitude gnss correction.

We will also replace the existing altitude threshold of 500ft difference between the 
position and the airport with a lower threshold as well.  The initial value to considered 
is 100-200ft.

Analysis of the reliablity of the altitude from different data sources also suggests
we reject altitude from certain feed such as MLAT.  

#### Encapulating airground determination logics

Currently for position messages, we evaluate the airground status of the flight
at two main junctions:
* at the start when we match the position to a flight in the `tita-discrimator` proc, and 
* at the point when we are deciding to issue synthetic departure or arrival in the
`refine-airground-switch` proc.

In `tita-discriminator`, we use ground speed and altitude to determine the airground
status.  While in `refine-airground-switch`, we make the same determination based on
the distance and altitude difference between the position current location and the
airport closest to that position.  These two methods sometimes deliver conflicting 
airground status leading to the late departure/early arrival issue that we observe.  

The solution includes introducing new module(s) for encapsulating the all existing
and new rules for determining current airground status of a flight.  The allows 
rigous testing as well as providing a centralized location for this.  This
is to ensure consistent evaluation of the current airground
status of a flight throughout the hyperfeed codebase.

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
