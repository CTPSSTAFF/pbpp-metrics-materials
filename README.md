# pbpp-metrics-materials
Materials to support the calculation of some federally-mandated roadway performance metrics.

## Contents
The contents of this repository:
* tmc-lists folder - contains list of all TMCs on the NPMRDS in the Boston Region MPO area in various forms:
  * npmrds_tmc_brmpo_nhs.csv - list of TMCs in CSV format
  * npmrds_tmc_brmpo_nhs_no_quotes.txt - comma-separated list of TMCs without quotes, for use when querying RITIS
  * npmrds_tmc_brmpo_nhs_with_quotes.txt - comma-separated list of TMCs with quotes, for use when performing database queries, e.g., with ArcGIS
* tmc-identification folder
  * TMC_Identification.csv - file obtained from RITIS when querying for the list of TMCs, above. Contains AADT data for each TMC. 
* sample-data folder - contains data returned by RITIS when querying the list of TMCs above for data on June 15, 2022
  * brmpo_npmrds_15June2022-all-vehicles.csv - CSV file from querying for data for all vehicles
  * brmpo_npmrds_15June2022_trucks.csv - CSV file returned from querying for data for trucks only
* speed-limits folder
  * tmcs_with_speed_limit_from_cmp.csv - CSV file of {TMC, speed_limit} pairs for TMCs that were included in the last CMP
  * tmcs_with_speed_limit_from_cmp_no_quotes.csv - the above list of TMCs, comma-delimited without quotes
  * tmcs_with_speed_limit_from_cmp_with_quotes.csv - the above list of TMCs, comma-delimited with quotes
* shapefile folder
  * Massachusetts.zip - Compressed NPMRDS TMC shapefile for Massachusetts, downloaded from [RITIS](https://npmrds.ritis.org/analytics/shapefiles)
  * Massachusetts.{dbf, prj, shp, shx} - Files extracted from the above ZIP archive
* massdot-materials folder
  * TMC-and-speed-events.gdb.zip - compressed file geodatabase containing:
    * Conflation_Vector - undocumented / unknown
	* LRS_Routes -'routes' event table / feature class, from the MassDOT Road Inventory
	* LRSE_Speed_Limit - 'speed limit' event table / feature class, from the MassDOT Road Inventory
	* LRSE_SpeedRegulation - 'speed regulation' event table / feature class, from the MassDOT Road Inventory
	* TMC_proposal - undocumented / unknown
    * TMC_not_null_weighted_SR.xlsx - table produced by 1spatial.com mapping TMC \(here: Traffic\_ID\) to distance-weighed speed regulation
* results folder
    * npmrds_tmc_brmpo.csv - final results table, in CSV format
	* working3.gdb.7z - 7z-compressed file Geodatabase containing intermediate results of running Approach #2, below
  
## Notes on the Contents of the massdot-materials folder
The __TMC\_not\_null\_weighted\_SR.xlsx__ spreadsheet was shpped to RITIS by Charles Major of MassDOT on September 26, 2023,
and shared with the author of this document by David Knudsen on November 22, 2023.
  
The compressed __TMC-and-speed-events.gdb__ geodatabase was obtained from MassDOT and shared with the author of this document
by David Knudsen on November 28, 2023.

Comments on the contents of this geodatabase from David Knudsen:
> You must filter time-wise in the two speed event layers by deleting records where To_Date IS NOT NULL! The two layers are up-to-the-minute current. 
> The LRSN_Routes layer is a couple months old: what I sent to 1Spatial to run their routines against. Their layer "TMC_PROPOSAL" dates to 2023-10-24, 
> based on the 2021 TMCs, I think. I don't know much about what all of their fields mean.

## Notes on Overall Methodology
The formula for calculating the Peak Hour Excessive Delay metrics, as defined by the FHWA is:

> Peak Hour Excessive Delay - ALL PHED during 15-minute increments during peak hours (eventually converted to hours)
> Traffic congestion will be measured by the annual hours of peak hour excessive delay (PHED) per capita on the NHS. 
> Excessive delay will be based on travel time at 20 miles per hour or 60 percent of the posted speed limit travel time, 
> whichever is greater, during in 15-minute intervals per vehicle. https://www.fhwa.dot.gov/tpm/faq.cfm#phed

In order to calculate this metric, for each TMC included in the analysis, we will need:
1. travel time data
2. the 'posted speed limit'

Item \(1\) can be obtained directly from RITIS. Item \(2\) is a bit more of a challenge.  
Even though MassDOT maintains posted speed limit data \(and 'speed regulation'\) data in the Road Inventory,
this data is stored in 'event tables' which are effectively overlayed on the underlying MassDOT route system.
The challenge here is that the relevant event tables do not segment the underlying route system in the same
way the TMC network does. In other words, the two systems, in general, don't 'line up.' This makes obtaining
the actual speed limit for a TMC \(as needed in these calculations\) less than straightforward.  

Of all the TMCs in the NPMRDS in the Boston MPO region, the majority - 2,767 - are part of the 
Congestion Management Process (\CMP\) TMC network. This is very good news, because as part of the CMP, CTPS
staff conflated the CMP-specific portion of the TMC network with the Road Inventory, and obtained the speed 
limit for each TMC. Athough much of this work was automated the final conflation required manual review and
correction, particulary for TMCs that were not on express highways. For all the TMCs that are part of the
CMP network, we will use the speed limit from the CMP as we have high confidence in its accuracy.

This leaves the question of how to obtain speed limits for the remaining 1,956 TMCs. The methodology we adopted
relies upon the work of a consulting firm, [1spatial.com](https://1spatial.com), that was hired by MassDOT to 
conflate the full INRIX TMCnetwork with the MassDOT Road Inventory a year or so ago. 
This conflation was performed completely programmatically;no review and correction by humans was invovled. 
As part of the conflation, 1spatial produced an event table / feature class of TMC IDs and the 
associated 'Weighted_Average_SR' - the distance-weighted _speed\_regulation_ for the TMC.
\(__Note:__ 1spatial is
headquartered in the UK, and its website is apparently hosted there. Due to geo-blocking on the CTPS firewall,
the 1spatial.com website is not visible from within the CTPS network.\)

While _speed\_regulation_ isn't identical to _speed\_limit_, Bob Frey of MassDOT's Office of Transportation
Planning has advised CTPS that it should be taken as more authoratative than _speed\_limit_. Consequently,
we will first attempt to use the _speed\_regulation_ value to come up with a 'speed limit' for the remaining TMCs.

### Approach 1 - Harvest Data from the Speed\_Regulation Event Table 
This section documents the methodology for calculating the 'speed limit' for TMCs in the NPMRDS within the
Boston Region MPO area but that are __not__ included in the CMP __using data in the MassDOT Speed\_Regulation Event Table.
In point of fact, the process was run on
all NPMRDS TMCs in the MPO region, but it was used to obtain the speed limit only for those TMCs that 
are not part of the CMP. As noted above, the reason for this is that the speed limits for TMCs included in the CMP
were obtained as a result of a conflation between the TMC network and the Road Inventory that was partially automated,
but also subject to review and correction by humans. The methodology described here relies upon a purely programmatic 
conflation \(performed by 1spatial.com\) that was __not__ subjected to review and correction by humans; it is thus
judged less reliable.

#### Approach 1 - Overview
The general approach taken begins by calcuating the intersection of 'TMC\_Proposal' and 'LRSE\_SpeedRegulation'.
From the result of the intersection the speed limit that applies along each 'TMC part' is calculated, and finally
the results from all the 'parts' comprising each TMC is aggregated. There is some hand-waving involved in this
description; all the nitty-gritty details are described in the 'Detailed Steps' section, below.

TMC\_Proposal is a feature class produced by 1spatial that indicates the 'from-measure' and 'to-measure' of each TMC with respect to
each MassDOT RouteID with which it is associated.
LRSE\_SpeedRegulation is a feature class taken from the MassDOT Road Inventory that indicates the 'from-measure' and 'to-measure' 
along a given MassDOT Route\_ID that a given speed regulation value applies.

The intersection is performed by the ESRI __Overlay\_Route\_Events__ tool. This tool takes event tables, rather than feature classes
as inputs. So, as a preparatory step each of the feature classes is exported to a 'vanilla' table in a Geodatabase. 

The basic approach taken is to use the $D = R * T$ formula to calculate the speed limit for each TMC.

#### Approach 1 - Detailed Steps
1. Export the LRSE_SpeedRegulation feature class as a 'vanilla' table that will be used as an event table. The 'route identifier' field
in this table is __Route\_ID__; the 'from-measure field is __From_Measure__; the 'to-measure' field is __To\_Measure__.

2. Export the TMC_Proposal feature class as a 'vanilla' table that will be used as an event table. The 'route identifier field 
in this table is __PROPOSAL\_LRS\_ROUTE\_ID__; the 'from-measure' field is __Begin\_Measure__; the 'to-measure' field is __End\_Measure__.

3. Run the __Overlay\_Route\_Events__ tool, saving the output to a table called __TMC\_SpeedReg\_overlay\_ETbl__. The parameters to
the tool invocation are as follows:
* Input event table: TMC\_Proposal\_ETbl
  * Route identifier field: ROPOSAL\_LRS\_ROUTE\_ID
  * Event Type: LINE
  * From-measure field: Begin\_Measure
  * To-measure field: To\_Measure
* Overlay event table: LRSE\_Speed\_Regulation\_ETbl
  * Route identifier field: Route\_ID
  * Event type: LINE
  * From-measure field: From\_Measure
  * To-measure field:: To\_Measure
* Type of overlay: INTERSECT
* Overlay \(output\) event table: TMC_Speed_Reg_overlay_ETbl
  * Route identifier field: PROPOSAL\_LRS\_ROUTE\_ID
  * Event type: LINE
  * From-Measure Field: Intersect_From
  * To-Measure Field: Intersect_To
  * Keep zero length line events: TRUE
  * Include all fields from input: TRUE
  * Build index: TRUE
The result of this overlay operation is a table of 'TMC fragments'.

4. Select all records with a NULL or 0-value __Speed__ \(i.e., speed regulation\) field in  __TMC\_SpeedReg\_overlay\_ETbl__, and __delete__ them.
NOTE: the '__Speed__' field in this table is aliased to '__Speed\_Limit__', making it difficult to spot unless one has turned off 'Show Field Alias'.

5. The __TMC\_SpeedReg\_overlay\_ETbl__ contains one record for each 'TMC piece'. Add a new field, __dist__ of type __double__ to this table,
and calculate its value as $abs(Intersect To - Intersect From)$. \(We need to use the _abs_ function here, because there are cases in which
the From measure can be Less than the To measure in an event table.\)

6. At this point, we have a __speed__ value and a __dist__ \(distance\) value for each 'TMC piece'. Given this, we can calcuate the __travel\_time__
for each 'TMC piece', using the $D = R * T$ formual. Add a field named __travel\_time__, of type __double__ to __TMC\_SpeedReg\_overlay\_ETbl__.
Calculate its value as $dist / speed$.

7. Calcualte the total length of each TMC, using the __Summary Statistics__ tool. The parameters to the tool invocation are:
* Input table: TMC\_SpeedReg\_overlay\_ETbl
* Output table: tmc\_dist\_Tbl
* Statistics field: dist
* Statistics type: SUM
* Case field: TMC  
The result of this operation is a table of \{ TMC, FREQUENCY, SUM_distance \} triples. The FREQUENCY column can be ignored for our purposes.

8. Calculate the total travel time along the entire length of the TMC, using the __Summary Statistics__ tool. The parameters to the tool invocation are:
* Input table: TMC\_SpeedReg\_overlay\_ETbl
* Output table: tmc\_travel\_time\_Tbl
* Statistics field: travel\_time
* Statistics type: SUM
* Case field: TMC  
The result of this operation is a table of \{ TMC, FREQUENCY, SUM_travel_time \} triples. Again, the FREQUENCY column can be ignored for our purposes.

9. Now that we have the total length and total travel time for each entire TMC, we can calculate the speed limit for each entire TMC
using the $R = D / T$ formula. Join __tmc\_dist\_Tbl__ and __tmc\_travel\_time\_Tbl__ on 'TMC', and export the result to __tmc\_speed\_limit\_Tbl__.

10. Calculate a 'draft' speed limit for each TMC:
* Add a field __speed\_limit\_draft__, field of type __double__, to __tmc\_speed\_limit\_Tbl__. Calculate its value as $SUM dist / SUM travel time$.

11. In general, the 'draft' speed limit values may not be integral; even if they are integers, they may not be multiples of 5 \(as are speed limits\).
Add a new field, __speed\_limit__, of type __long__ to __tmc\_speed\_limit\_Tbl__; cacluate its value to to $int(5 * round(draft speed limit / 5))$ 
to ensure that the  resulting calculated speed limit is an integral multiple of 5. 
This gives a speed limit for all TMCs in Massachusetts, based on MassDOT's Speed_Regulation event table and 1spatial's conflation of the TMC network with the Road Inventory. 

12. As noted above, as part of our work on the CMP we have speed limit data that has been subject to review and correction by a human for all TMCs
in the CMP network. When we have a speed limit for a TMC from the CMP, we will use it in favor of the one calculated above. For all other TMCs,
we will use the speed limit as calculated above. We accomplish this as follows:
* Join __tmc\_speed\_limit\_Tbl__ to __cmp_2019\_tmc\_speed\_limit__ on the 'TMC' field.
* Export the result to __non\_cmp\_tmc\_speed\_limit\_Tbl__
* Select all records for which the __speed\_limit__ from the CMP speed limit table is NULL, and __delete__ them.
* Prune this table by removing all fields except __TMC__ and the __speed\_limit__ field that came from the joined table,
and save the result in __non\_cmp\_tmc\_speed\_limit\_Tbl\_pruned__.
The result is a table of speed limits for TMCs that are _not_ part of the CMP network. Note that this includes TMCs
that are outside of the Boston Region MPO area.

13. Merge the __non\_cmp\_tmc\_speed\_limit\_Tbl\_pruned__ table and the __cmp\_2019\_tmc\_speed\_limit__ table,
producing the __final\_MA\_tmc\_speed\_limit\_Tbl__. As the name of this file indicates, it includes TMCs 
throughout the sate, including those ouside the Boston MPO region. 

When these TMCs were removed from the table, we found that it did __not__ yeild speed limit data for any TMCs for which
we didn't already have a speed limit from the CMP.

### Approach 2 - Harvest Data from the Speed\_Limit Event Table
__The approach of harvesting speed limit data from the Speed\_Regulation event table having borne
no fruit, we turn now to harvesting this data from the Speed\_Limit event table.__

Although there are many more events in the Speed\_Limit event table than in the Speed\_Regulation event table,
all of the data in it hasn't been vetted recently, and it is regarded as the less authoritative of the two.
Nonetheless, it is the only alternative available.

Obtaining speed limit data from he Speed\_Limit event table is much more difficult than obtaining it \(if available\)
from the Speed\_Regulation event table: the Speed\Limit table only carries speed limit data on the __primary__ route; this is 
a challenge when two routes are _concurrent_ - which is typically the case for non-limited-access roads.
For example: Where Route 62 EB and Route 62 WB are concurrent, speed limit data is carried only in
events on Route 62 __EB__ \(the primary route direction\); speed limit data for the corresponding section
of Route 62 WB is carried in the __opposing\_speed\_limit__ event on Route 62 EB. This makes processing
much more complicated.

#### Approach 2 - Detailed Steps
Inputs:
* MassDOT __LRS\_Routes__ feature class
* MassDOT Speed\_Limit \(__LRSE\_Speed\_Limit__\) event feature class
* TMC\_proposal feature class - this is the feature class produced by 1spatial conflating the INRIX TMC network with the MassDOT Road Inventory

1. 'Clean up' the __LRSE\_Speed\_Limit__ FC. The approach here is to populate the __Op\_Dir\_SL__ \(opposing direction speed limit\)
with the value of the __Speed\_Lim__ \(primary direction speed limit\) field, whenever __Op\_Dir\_SL__ contains no
useful information. This is the case when its value is NULL, or 0, or 99. 
* Select all records for which __Op\_Dir\_SL__ is NULL, 0, or 99.
* From these, select all records for which __Speed\_Lim__ is NULL, 0, or 99, and __delete__ them. These records have no speed limit data that 
can usefully participate in the following calculations.
* For the remaining records, set the __Op\_Dir\_SL__ field to the value of the __Speed\_Lim__  field.
* Select all records for which the __Op\_Dir\_SL__ field is _still_ NULL, 0, or 99, and __delete__ them. \(These records
had NULL, 0, or 99 values in their __Speed\_Lim__ field to begin with.\)
* Delete all records from the Speed\_Limit FC which have a __shape\_length__ of 0.
* Save the results of these 'cleanup' steps to a new feature class: __LRSE\_Speed\_Limit\_clean__

2. Calculate the _bearing_ of the geometry of each feature in the __LRSE\_Speed\_Limit\_clean__ FC.
This is most efficiently performed by running the __Add Geometry Attributes__ tool, specifying the __LINE\_BEARING__ parameter.
This tool _alters its input FC_, writing the angular bearing \(in __degrees__\) to it in an attribute named __BEARING__.
The tool offers no options to specify the name of this attribute; it is hard-wired to 'BEARING'.
In a subsequent processing step \(Step 8\), we will calculate the bearing of the TMC features 'located' against the 
LRS\_Route route system, we need to rename the BEARING field produced here to something else: __LRSE\_spd\_lim\_bearing__.  
In summary:
* Calculate the bearing of the  __LRSE\_Speed\_Limit\_clean__ FC using the __Add Geometry Attributes__ tool.
* Rename this field to __LRSE\_spd\_lim\_bearing__ using the __Alter Field__ tool.

3. \(Preparation for Step 4.\) Make a spatial selection on the __LRS\_Routes__ FC: select all features that CONTAIN 
the __LRSE\_Speed\_Limit\_clean__ FC.
\(David K. recommends using the CONTAINs selection criterion rather than INTERSECTs, as the latter can pick up 'touching' orthogonal routes that 
have nothing to do with the routes we want in the spatial overlay.\) Save the selected features as the __LRS\_Routes\_selected__ FC.

4. Spatially intersect the __LRSE\_Speed\_Limit\_clean__ FC and the __LRS\_Routes\_selected FC__ \(after Step 3. has been executed on it\).
Save the result in a new feature class called __intersect\_FC__.
\(Here David K. suggests that some experimentation with the overlay tool may be needed: Intersection is similar to INNER 
JOIN, Identity to LEFT OUTER JOIN, and Union to FULL OUTER JOIN.\)

5. Use the __Locate Features Along Routes__ tool to locate __intersect\_FC__ on the __LRS\_Routes\_selected__ route system feature class.
The result is an event table we'll call __located\_features\_ET__. The parameters to the tool invocation are:
* Input Features: intersect\_FC
* Input Route Features: LRS\_Routes\_selected 
* Route Identifier Field: Route\_ID 
* Output Event Table: located\_features\_ET 
* Output Event Table Properties:
  * Route Identifier Field: RID
  * Event Type: LINE 
  * From-Measure Field: FMEAS
  * To-Measure Field: TMEAS
  * Include all fields from input: TRUE 
  * Use M Direction Offseting: TRUE \(__check this__\)

6. Convert __located\_features\_ET__ into a feature class __located\_features\_FC__. Detailed steps:
* Run the __Make Route Event Layer__ tool with the parameters:
  * Input Route Fetures: LRS\_Routes\_selected 
  * Route Identifier Field: Route\_ID 
  * Input Event Table: located\_features\_ET 
  * Event Table Properties:
    * Route Identifier Field: RID 
	* Event Type: LINE 
	* From-Measure Field: FMEAS 
	* To-Measure Field: TMEAS
	* Name of Table or View: __located\_features\_View__
* Save the __located\_features\_View__ \(an in-memory data structure \) as a feature class: __located\_features\_FC__

7. Prune features from __located\_features\_FC__: 
* First, select all features from __located\_features\_FC__ for which the input route identifier \(__Route\_ID__\)
matches the route identifier \(__RID__\) of the feature against which it was located, and export this to a new feature 
class: __located\_features\_FC\_pruned__. \(This excludes all features for which __Route\_ID__ doesn't equal __RID__ from the following steps.\)
* Second, select all features from __located\_features\_FC\_pruned__ whose __Shape\_length__ is 0, and delete them. \(Records with 0-length
geometry can be artifacts of overlay operations.\)

8. Calculate the bearing of the geometry of each feature in the __located\_features\_FC\_pruned__ FC;
this takes into account the _output_ geometry.
This field is calculated using the same method as in Step \(2\):
* Calculate the bearing of the __located\_features\_FC\_pruned__ FC using the __Add Geometry Attributes__ tool.
* Rename this field to __locd\_feats\_FC_pruned\_bearing__ using the __Alter Field__ tool.

9. Export __located\_features\_FC\_pruned__  as a table, called __located\_features\_FC\_pruned\_ET__, for use as an input to the Step 11.

10. Export the TMC\_proposal feature class as a table: __TMC\_Proposal\_ETbl__.

11. Use the __Overlay Route Events__ tool to perform a tabular overlay of __TMC\_Proposal\_ETbl__ with  __located\_features\_FC\_pruned\_ET__, 
to produce __overlay\_output\_table__. The parameters to the tool invocation are as follows:
* Input event table: TMC\_Proposal\_ETbl
  * Route identifier field: ROPOSAL\_LRS\_ROUTE\_ID
  * Event Type: LINE
  * From-measure field: Begin\_Measure
  * To-measure field: To\_Measure
* Overlay event table: located\_features\_FC\_pruned\_ET
  * Route identifier field: RID
  * Event type: LINE
  * From-measure field: FMEAS
  * To-measure field: TMEAS
* Type of overlay: INTERSECT
* Overlay \(output\) event table: overlay\_output_\table
  * Route identifier field: PROPOSAL\_LRS\_ROUTE\_ID
  * Event type: LINE
  * From-Measure Field: Intersect_From
  * To-Measure Field: Intersect_To
  * Keep zero length line events: TRUE  \(__not sure if this is a good idea__\)
  * Include all fields from input: TRUE
  * Build index: TRUE  
  
The result of this overlay operation is a table of 'TMC fragments'.

12. __Sanity Check__: Select all records in __overlay\_output\_table__ with a __Speed\_Lim__ value that is NULL, 0, or 99, and __delete__ them.
Note: The __Speed\_Lim__ field in this table is aliased to __Speed\_Limit__, which may make it difficult to spot.

13. Add a new field, __computed\_speed\_limit__, of type __long__, to __overlay\_output\_table__.

14. Calculate the value of __computed\_speed\_limit__.
Whether the value of __Speed\_Lim__ \(speed_limit\) or __Op\_Dir\_SL__ \(opposing_speed_limit\) is used is determined by
whether the two bearings 'align'.

Given that the two bearings are expressed in degrees, the pseudo-code for this is as follows:
```
	bearing_delta = locd_feats_FC_pruned_bearing - LRSE_spd_lim_bearing
	
	if abs(bearing_delta) < 180 then
		normalized_bearing_delta = abs(bearing_delta)
	else
		normalized_bearing_delta = 360 - abs(bearing_delta)
	end_if
	
	if normalized_bearing_delta > 90 then	
		computed_speed_limit = Op_Dir_SL
	else
		computed_speed_limit  = Speed_Lim
	end_if
```
15. At this point, we will have a speed limit for each 'TMC piece' resulting from the tabular overlay operation \(Step 11\).
We then proceed to calcuate a speed limit for each _entire_ TMC using the method described  Steps \(5\) throuhg \(11\) in Approach #1; 
in summary, these steps are:
  * Calcluate a __dist__ for each 'TMC piece'
  * Calcuate a __travel\_time__ for each 'TMC piece'
  * Calcuate a total distance for each entire TMC
  * Calcuate a total travel time for each entire TMC
  * Calcuate a 'draft speed limit' for each entire TMC, using the  $R = D / T$  formula
  * Round this value to an integral multiple of 5 MPH \(speed limits are integral multiples of 5 MPH\)
  
The details follow:  

16. Calcluate a __dist__ for each 'TMC piece': The __overlay\_output\_table__  contains one record for each 'TMC piece'.
Add a new field, __dist__ of type __double__ to this table,
and calculate its value as $abs(Intersect To - Intersect From)$. \(We need to use the _abs_ function here, because there are cases in which
the From measure can be Less than the To measure in an event table.\)

17. Calcuate a __travel\_time__ for each 'TMC piece': At this point, we have a __calculated\_speed\limit__ \(i.e., __speed__\) value
and a __dist__ \(distance\) value for each 'TMC piece'. Given this, we can calcuate the __travel\_time__
for each 'TMC piece', using the $D = R * T$ formula. Add a field named __travel\_time__, of type __double__ to __overlay\_output\_table__.
Calculate its value as $dist / speed$.

18. Calcualte the total length \(distance\) of each TMC, using the __Summary Statistics__ tool. The parameters to the tool invocation are:
* Input table: overlay\_output\_table
* Output table: tmc\_dist\_Tbl
* Statistics field: dist
* Statistics type: SUM
* Case field: TMC  
The result of this operation is a table of \{ TMC, FREQUENCY, SUM_distance \} triples. The FREQUENCY column can be ignored for our purposes.

19. Calculate the total travel time along the entire length of the TMC, using the __Summary Statistics__ tool. The parameters to the tool invocation are:
* Input table: overlay\_output\_table
* Output table: tmc\_travel\_time\_Tbl
* Statistics field: travel\_time
* Statistics type: SUM
* Case field: TMC  
The result of this operation is a table of \{ TMC, FREQUENCY, SUM_travel_time \} triples. Again, the FREQUENCY column can be ignored for our purposes.

20. Now that we have the total length and total travel time for each entire TMC, we can calculate a 'draft speed limit' for each entire TMC
using the $R = D / T$ formula. Preparatory step: join __tmc\_dist\_Tbl__ and __tmc\_travel\_time\_Tbl__ on 'TMC', and export the result to __tmc\_speed\_limit\_Tbl__.

21. Calcuate a 'draft speed limit' for each entire TMC:
* Add a field __speed\_limit\_draft__, field of type __double__, to __tmc\_speed\_limit\_Tbl__. 
* Calculate its value as $SUM dist / SUM travel time$, i.e., using the  $R = D / T$  formula.

21. Round the 'draft speed limit' to an integral multiple of 5 MPH. In general, the 'draft' speed limit values may not be integral, 
and even if integral, the values may not be multiples of 5 \(as are speed limits\).
Add a new field, __speed\_limit\_final__, of type __long__ to __tmc\_speed\_limit\_Tbl__; calculate its value to to $int(5 * round(draft speed limit / 5))$ 
to ensure that the  resulting calculated speed limit is an integral multiple of 5. 
This gives a speed limit for all TMCs in Massachusetts for which:
* 1spatail successfully conflated the TMC to one or more MassDOT routes, and
* speed limit data was availble in the Speed\_Limit event table in the MassDOT Road Inventory

#### Approach 2 - Final Steps
The final steps entail preparing a shapefile of the NPMRDS TMCs in the Boston MPO region to which are added the following
attributes:
* speed limit calculated by 2019 CMP
* speed limit calcluated from 'Approach 2'
* a 'final' speed limit to use in calculating roadway performance metrics required for the Performnace-Based Planning and
Programming dashboard.

The steps are as follows:
1. Project the 'Massachusetts.shp' shapefile \(the NPMRDS TMC shapefile for Massachusetts\) to EPSG:26986.
2. Select the features in \(1\) that intersect the Boston Region MPO \(mpodata.mpodata.CTPS\_BRMPO\_BOUNDARY\_POLY\).
3. Export \(2\) to a new feature class __npmrds\_tmc\_brmpo__.
4. Add three new attributes to __npmrds\_tmc\_brmpo__, each of type __long__: __cmp\_speed\_limit__, __calc\_speed\_limit__, and 
__final\_speed\_limit__.
5. Join __npmrds\_tmc\_brmpo__ to the table of speed limits produced by the 2019 CMP: __cmp\_2019\_cmp_speed\_limit__, on 'tmc',
keeping all records.
6. Calculate the value of the __cmp\_speed\_limit__ attribute to that of the _speed\_limit__ column in the __cmp\_2019\_cmp_speed\_limit__ table;
then remove the join.
7. Calculate the value of the __calc\_speed\_limit__ attribute to that of the __final\_speed\_limit__ column in the __tmc\_speed\_limit\_Tbl__ table;
then remove the join.
8. Populate the __speed\_limit\_final__ attribute. The basic approach is to use the speed limit from the CMP whenever it is available, even if
a different value was calcuated by Approach #2. The rationale for this is that the speed limits calculated for the CMP were subject to 
review after first having been calculated by an automated process, whereas those from the Road Inventory were the result of an entirely automated conflation.
The details:
* Select all records for which __cmp\_speed\_limit__ IS NOT NULL; set the value of __final\_speed\_limit__ to the value of __cmp\_speed\_limit__ . Then clear the selection.
* Select all records for which __cmp\_speed\_limit__ IS NULL AND __calc\_speed\_limit__ IS NOT NULL; 
set the value of __final\_speed\_limit__ to the value of __calc\_speed\_limit__. Then clear the selection.

Finally, export the attribute table of __npmrds\_tmc\_brmpo__ to a CSV file for use in subsequent analysis and the performance dashboard.
Drop all columns except _tmc__ and __final\_speed\_limit__.

#### Approach 2 - Summary of Results
The following table summarizes the source of speed limit data \(if any\) for the 5143 TMCs
in the Boston Region MPO area:
| Factor | Number of TMCs |
| ------------ | ----------- |
| Total number of TMCs | 5143 |
| Speed limit from 2019 CMP | 2853 |
| Speed limit from 2022 Road Inventory using Approach #2 | 1387 |
| No speed limit | 903 |