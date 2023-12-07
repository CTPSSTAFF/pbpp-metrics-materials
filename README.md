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
	* LRSE_Routes -'routes' event table / feature class, from the MassDOT Road Inventory
	* LRSE_Speed_Limit - 'speed limit' event table / feature class, from the MassDOT Road Inventory
	* LRSE_SpeedRegulation - 'speed regulation' event table / feature class, from the MassDOT Road Inventory
	* TMC_proposal - undocumented / unknown
  * TMC_not_null_weighted_SR.xlsx - table produced by 1spatial.com mapping TMC \(here: Traffic\_ID\) to distance-weighed speed regulation
  
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
we will be using the _speed\_regulation_ value in the calcuations below to come up with a 'speed limit' for the remaining TMCs.

### Methodology for Calculating 'Speed Limit' for TMCs not in the CMP
This section documents the methodology for calculating the 'speed limit' for TMCs in the NPMRDS within the
Boston Region MPO area but that are __not__ included in the CMP. In point of fact, the process was run on
all NPMRDS TMCs in the MPO region, but it was used to obtain the speed limit only for those TMCs that 
are not part of the CMP. As noted above, the reason for this is that the speed limits for TMCs included in the CMP
were obtained as a result of a conflation between the TMC network and the Road Inventory that was partially automated,
but also subject to review and correction by humans. The methodology described here relies upon a purely programmatic 
conflation \(performed by 1spatial.com\) that was __not__ subjected to review and correction by humans; it is thus
judged less reliable.

#### Overview
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

#### Detailed Steps
1. Export the LRSE_SpeedRegulation feature class as a 'vanilla' table that will be used as an event table. The 'route identifier' field
in this table is __Route\_ID__; the 'from-measure field is __From_\Measure__; the 'to-measure' field is __To\_Measure__.
2. Export the TMC_Proposal feature class as a 'vanilla' table that will be used as an event table. The 'route identifier field 
in this table is __PROPOSAL\_LRS\_ROUTE\_ID__; the 'from-measure' field is __Begin\_Measure__; the 'to-measure' field is __End\_Measure__.
3. Run the __Overlay\_Route\_Events__ tool, saving the output to a table we'll call __TMC\_SpeedReg\_overlay\_ETbl__.

4. Calcuate the 'conflated_length' of each TMC - i.e., the length of the TMC that has been conflated to a MassDOT Route. This is _not_
necessarily the entire length of the TMC: some portion of any given TMC may have failed to have been conflated to any MassDOT Route.
The result of this is a table of \{ TMC\_ID, conflated\_length \} pairs we'll call the __tmc\_total\_conflated\_length__ table.

5. Calculate the 'weighted speed limit' for each TMC 'part': for each record in __TMC\_SpeedReg\_overlay\_ETbl__,
this is given by __speed__ * \(length of the TMC 'part' / total conflated length of the TMC\)




## TO BE CONTINUED