# pbpp-metrics-materials
Materials for the calculation of some federally-mandated roadway performance metrics.

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
