---
title: ""
layout: notebook

---
## Shore Station Compliance Checker Script

The IOOS Compliance Checker is a Python-based tool that helps users check the meta data compliance of a netCDF file. This software can be run in a web interface here: https://data.ioos.us/compliance/index.html The checker can also be run as a Python tool either on the command line or in a Python script.  This notebook demonstrates the python usage of the Compliance Checker.


### Purpose: 
Run the compliance checker python tool on a Scipps Pier shore station dataset to check for the metadata compliance.

The Scripps Pier automated shore station operated by Southern California Coastal Ocean Observing System (SCCOOS) at Scripps Institution of Oceanography (SIO) is mounted at a nominal depth of 5 meters MLLW. The instrument package includes a Seabird SBE 16plus SEACAT Conductivity, Temperature, and Pressure recorder, and a Seapoint Chlorophyll Fluorometer with a 0-50 ug/L gain setting.

### Dependencies: 
This script must be run in the "IOOS" environment for the compliance checker to work properly.

Written by: J.Bosch Feb. 10, 2017



<div class="prompt input_prompt">
In&nbsp;[1]:
</div>

```python
import compliance_checker

print(compliance_checker.__version__)
```
<div class="output_area"><div class="prompt"></div>
<pre>
    4.2.0+11.gd85b593

</pre>
</div>
<div class="prompt input_prompt">
In&nbsp;[2]:
</div>

```python
# First import the compliance checker and test that it is installed properly.
from compliance_checker.runner import CheckSuite, ComplianceChecker

# Load all available checker classes.
check_suite = CheckSuite()
check_suite.load_all_available_checkers()
```

<div class="prompt input_prompt">
In&nbsp;[3]:
</div>

```python
# Path to the Scripps Pier Data.


# See https://github.com/Unidata/netcdf-c/issues/1299
# for the reason we need to append `#fillmismatch` to the URL.
url = "http://data.ioos.us/thredds/dodsC/deployments/rutgers/ru29-20150623T1046/ru29-20150623T1046.nc3.nc#fillmismatch"
```

### Running Compliance Checker on the Scripps Pier shore station data
This code is written with all the arguments spelled out, following the usage instructions on the README section of compliance checker github page: https://github.com/ioos/compliance-checker

<div class="prompt input_prompt">
In&nbsp;[4]:
</div>

```python
output_file = "buoy_testCC.txt"

return_value, errors = ComplianceChecker.run_checker(
    ds_loc=url,
    checker_names=["cf", "acdd"],
    verbose=True,
    criteria="normal",
    skip_checks=None,
    output_filename=output_file,
    output_format="text",
)
```

<div class="prompt input_prompt">
In&nbsp;[5]:
</div>

```python
with open(output_file, "r") as f:
    print(f.read())
```
<div class="output_area"><div class="prompt"></div>
<pre>
    
    
    --------------------------------------------------------------------------------
                             IOOS Compliance Checker Report                         
                                        acdd:1.3                                    
    http://wiki.esipfed.org/index.php?title=Category:Attribute_Conventions_Dataset_Discovery
    --------------------------------------------------------------------------------
                                   Corrective Actions                               
    ru29-20150623T1046.nc3.nc#fillmismatch has 11 potential issues
    
    
                                   Highly Recommended                               
    --------------------------------------------------------------------------------
    Global Attributes
    * Conventions does not contain 'ACDD-1.3'
    
    variable "conductivity" missing the following attributes:
    * coverage_content
    
    variable "density" missing the following attributes:
    * coverage_content
    
    variable "platform_meta" missing the following attributes:
    * coverage_content
    * standard_name
    
    variable "pressure" missing the following attributes:
    * coverage_content
    
    variable "salinity" missing the following attributes:
    * coverage_content
    
    variable "temperature" missing the following attributes:
    * coverage_content
    
    variable "u" missing the following attributes:
    * coverage_content
    
    variable "v" missing the following attributes:
    * coverage_content
    
    
                                      Recommended                                   
    --------------------------------------------------------------------------------
    Global Attributes
    * geospatial_bounds not present
    * geospatial_bounds_crs not present
    * geospatial_bounds_vertical_crs not present
    * time_coverage_duration not present
    * time_coverage_resolution not present
    
    time_coverage_extents_match
    * Failed to retrieve and convert times for variables time.
    
    
    --------------------------------------------------------------------------------
                             IOOS Compliance Checker Report                         
                                         cf:1.6                                     
                                http://cfconventions.org                            
    --------------------------------------------------------------------------------
                                   Corrective Actions                               
    ru29-20150623T1046.nc3.nc#fillmismatch has 5 potential issues
    
    
                                         Errors                                     
    --------------------------------------------------------------------------------
    §3.4 Ancillary Data
    * lat_qc is not a variable in this dataset
    * lon_qc is not a variable in this dataset
    
    §3.5 Flags
    * precise_lat_qc's flag_meanings and flag_values should have the same number of elements.
    * precise_lon_qc's flag_meanings and flag_values should have the same number of elements.
    
    §5.6 Horizontal Coorindate Reference Systems, Grid Mappings, Projections
    * longitude is not associated with a coordinate defining true latitude and sharing a subset of dimensions
    * longitude is not associated with a coordinate defining true longitude and sharing a subset of dimensions
    * latitude is not associated with a coordinate defining true latitude and sharing a subset of dimensions
    * latitude is not associated with a coordinate defining true longitude and sharing a subset of dimensions
    * time is not associated with a coordinate defining true latitude and sharing a subset of dimensions
    * time is not associated with a coordinate defining true longitude and sharing a subset of dimensions
    
    §9.1 Features and feature types
    * Different feature types discovered in this dataset: mapped-grid (u, v, time, longitude, latitude), trajectory-profile-incomplete (pressure, temperature, conductivity, salinity, density, platform_meta, depth)
    
    
                                        Warnings                                    
    --------------------------------------------------------------------------------
    §2.3 Naming Conventions
    * attribute trajectory:_Encoding should begin with a letter and be composed of letters, digits, and underscores
    * attribute wmo_id:_Encoding should begin with a letter and be composed of letters, digits, and underscores
    

</pre>
</div>
This Compliance Checker Report can be used to identify where file meta data can be improved.  A strong meta data record allows for greater utility of the data for a broader audience of data analysts.
<br>
Right click and choose Save link as... to
[download](https://raw.githubusercontent.com/ioos/notebooks_demos/master/notebooks/2017-05-14-running_compliance_checker.ipynb)
this notebook, or click [here](https://binder.pangeo.io/v2/gh/ioos/notebooks_demos/master?filepath=notebooks/2017-05-14-running_compliance_checker.ipynb) to run a live instance of this notebook.