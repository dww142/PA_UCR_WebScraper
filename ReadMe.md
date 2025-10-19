### PA UCR Web Scraper 
This repository is a set of tools for automating interaction with the Pennsylvania Uniform Crime Reporting (UCR) publicly available reports (https://www.ucr.pa.gov/PAUCRSPUBLIC/ReportsIndex/List). 

This project is written in Python (https://www.python.org/downloads/) using Jupyter notebooks.  
Recommended IDE: Visusl Studio Code with the Jupyter Notebook extension
- https://code.visualstudio.com/download
- https://code.visualstudio.com/docs/datascience/jupyter-notebooks

The scripts use Selenium (https://www.selenium.dev/documentation/webdriver/getting_started/first_script/?language=python) and the Chrome browser (not tested with other browsers).  

After download there is a workbook that uses Polars (https://docs.pola.rs/user-guide/getting-started/) to read the Excel files to a data frame and write a consolidated file out. 

[Web Scraper Notebook](AnnualSRSSummaryReport/srs_annual_report_web_scraper.ipynb)  
[File Reader/Consolidator Notebook](AnnualSRSSummaryReport/ReturnA/return_a_file_reader.ipynb)  

#### Annual SRS Summary Report
The starting point is specifically the Annual SRS Summary Report (https://www.ucr.pa.gov/PAUCRSPUBLIC/SRSReport/AnnualSRSSummary)

Which can be run for a selected time period and a selected set of counties (or agency or tag). 
When multiple counties are selected the data is combined into a single data set and cannot be dis-aggregated by county.  
In order to have a granular data set of records by county, the report must be run for each county for each time period, exported, then re-combined to create a more functional data set for analysis.  
The web scraper notebooks is set up to loop through a set of county and time period combinations, set parameters, generate the report, and export the report to Excel. 

Parameters
- Start Month-Year
  - Values here can be set directly with text - 3 character month, hyphen year (Jan - 2024)
- End Month-Year
  - Values here can be set directly with text - 3 character month, hyphen year (Dec - 2024)
  - Set to the same as Start Month-Year to run for a single month. 
- Report By
  - Set directly via text. Defaults to 'Agency', changing this to 'County' changes the values available in the 'County' input box. Sometimes changing this doesn't properly load the county options in the second box. There is a built in try/except/re-try to have some resilience to this (if it fails it re-tries once, if it fails twice in a row it logs the failure and moves on)
- Report Type
  - Set via text. Interested in 'Return A' mostly, but the scraper should work for any report type (the web scraper scope is to run/export/download the file, not concered with the content at this point)
- Agency / County / Tag
  - This changes based on the 'Report By' selection. 
  - Set by simulating a click in the input box, searching the list of counties and simulating a click on the desired county. 

Once the parameters are set it simulates a click on the 'Generate Report' button and uses 'Wait' functionality in Selenium (https://www.selenium.dev/documentation/webdriver/waits/) to determine when it is safe to simulate a click on the export to Excel option.  
Chrome should be set to automatically download the file and NOT prompt the user for approval or any input. 


##### Combining the downloaded files 
Under the AnnualSRSSummaryReport/ReturnA folder, there is a notebook to read the large number of downloaded files into a data frame. This is very specific to the format of the Exported Return A report - for example, the county name will always be in cell B16 - this will (probably) vary slightly from report to report.  
The Return A data table starts on cell B18, polars is pretty resilient about ignoring merged columns and just capturing the data into the frame.  
After merging the data, the resulting tables includes the following columns: 
- County
- County FIPS Code
- BeginDate
- EndDate
- Classification of Offenses
- ClassificationSortOrder
- Offenses Reported
- Unfounded
- Actual Offenses
- Total Offenses Cleared
- Clearances Involving Persons Under 18 Yr. of Age
- ReportRunDate
- SourceFile
  - This is included for troubleshooting missing, blank, or erroneus data to be able to check the individual files. 

The process pulls in Classification Of Offenses as a column. The values in this column include multiple types of records (detailed breakouts, subtotals, grand totals) in the same column (Sort Order is to replicate the order of these on the original report).  
The ClassificationOfOffenses.xlsx file creates a heirarchy of these values identifies record types 
- Classification Part (part 1/2 offenses label)
- Classification Category 
- Classification Sub Category 
- Classification Of Offenses 

Record Type
- Total (The Part 1/2 total rows from the original; sum includes all offenses)
- Detail (The lowest level of detail available for each offense; sum includes all offenses)
- Aggregate - not all offenses have an aggregate; Burglary is the aggregate of the detail breakdowns of burglary
- Aggregate Subtotal - only applies to the Sale/Manufacturing and Possession subtotal rows for Drug Violations; (Drug Abuse Violations is the aggregate). 

In theory the sum of Total records should match the sum of Detail; however the breakouts for Gambling to not add up to the Gambling subtotal row.  
<b>Recommendation:</b> filter to detail record types and use the classification hierarchy to calculate your own categorical aggregates and totals from the detail rows. 

The join between the consolidated data set and the classification categories includes a join on the derived Classification Sort Order column to ensure columns with the same name (i.e., Drug violations like 'Marijuana' is listed once for Sale and Manufacturing and once for Possession).  
RecordType is intended to indicate the level of detail of the Classification of Offense from the original report

Additional metadata - The ClassificationOfOffenses.xlsx file also contains 
- a list of counties and their corresponding FIPS codes, for joining counties to other publicly avialable data (i.e., Census ACS data) to include population statistics as context for analysis. 
- An extract of Census data containing county profile data about population, income, educational attainment, employment, and some housing data. 
  - This is based on another project not yet on Github (someday...)
  - Data Notes for this set: 
    - 2020 data is not available from the ACS survey in the Census (the numbers in this extract average together 2019 and 2021 data)
    - 2024 data is not yet released, this set includes records for 2024 containing 2023 data (basically: 2024 crime data can be joined to 2023 Census data)

## Already downloaded data 
AnnualSRSSummaryReport\ReturnA\YearlyData - contains individual files for 2008 - 2024 (Jan - Dec) for each county - ~1,000 files 
- These files are combined into AnnualSRSSummaryReport\ReturnA\SRSAnnualReturnAConsolidatedYearlyData.xlsx 
AnnualSRSSummaryReport\ReturnA\MonthlyData - contains individual county files for Jan - Sep of 2025 (Extracted Oct 19, 2025)
- These files are combined into AnnualSRSSummaryReport\ReturnA\SRSAnnualReturnAConsolidatedMonthlyData.xlsx

## Tableau Workbook
AnnualSRSSummaryReport\ReturnA\SRS_ReturnA_AnnualDataByCounty.twbx
- This is a packaged Tableau Workbook containing the 2008-2024 annual (yearly Jan-Dec) SRS data, blended (joined) to the ClassificationOfOffenses.xlsx file to get the Offense Categories and to the Census County Profile data set. 
It calculates some basic rates from the census data (Attainment rates, employment rates, labor force participation, occupancy/vacancy rates of housing) and some cross-data set rates such as incidence of offences per 1000 population. 
