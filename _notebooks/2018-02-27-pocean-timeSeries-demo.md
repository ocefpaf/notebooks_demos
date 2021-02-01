---
title: "Creating a CF-1.6 timeSeries using pocean"
layout: notebook

---


IOOS recommends to data providers that their netCDF files follow the CF-1.6 standard. In this notebook we will create a [CF-1.6 compliant](http://cfconventions.org/latest.html) file that follows file that follows the [Discrete Sampling Geometries](http://cfconventions.org/Data/cf-conventions/cf-conventions-1.7/build/ch09.html) (DSG) of a `timeSeries` from a pandas DataFrame.

The `pocean` module can handle all the DSGs described in the CF-1.6 document: `point`, `timeSeries`, `trajectory`, `profile`, `timeSeriesProfile`, and `trajectoryProfile`. These DSGs array may be represented in the netCDF file as:

- **orthogonal multidimensional**: when the coordinates along the element axis of the features are identical;
- **incomplete multidimensional**: when the features within a collection do not all have the same number but space is not an issue and using longest feature to all features is convenient;
- **contiguous ragged**: can be used if the size of each feature is known;
- **indexed ragged**: stores the features interleaved along the sample dimension in the data variable.

Here we will use the orthogonal multidimensional array to represent time-series data from am hypothetical current meter. We'll use fake data for this example for convenience.

Our fake data represents a current meter located at 10 meters depth collected last week.

<div class="prompt input_prompt">
In&nbsp;[1]:
</div>

```python
from datetime import datetime, timedelta

import numpy as np
import pandas as pd

x = np.arange(100, 110, 0.1)
start = datetime.now() - timedelta(days=7)

df = pd.DataFrame(
    {
        "time": [start + timedelta(days=n) for n in range(len(x))],
        "longitude": -48.6256,
        "latitude": -27.5717,
        "depth": 10,
        "u": np.sin(x),
        "v": np.cos(x),
        "station": "fake buoy",
    }
)


df.tail()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>time</th>
      <th>longitude</th>
      <th>latitude</th>
      <th>depth</th>
      <th>u</th>
      <th>v</th>
      <th>station</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>95</th>
      <td>2019-05-24 17:27:54.693087</td>
      <td>-48.6256</td>
      <td>-27.5717</td>
      <td>10</td>
      <td>0.440129</td>
      <td>-0.897934</td>
      <td>fake buoy</td>
    </tr>
    <tr>
      <th>96</th>
      <td>2019-05-25 17:27:54.693087</td>
      <td>-48.6256</td>
      <td>-27.5717</td>
      <td>10</td>
      <td>0.348287</td>
      <td>-0.937388</td>
      <td>fake buoy</td>
    </tr>
    <tr>
      <th>97</th>
      <td>2019-05-26 17:27:54.693087</td>
      <td>-48.6256</td>
      <td>-27.5717</td>
      <td>10</td>
      <td>0.252964</td>
      <td>-0.967476</td>
      <td>fake buoy</td>
    </tr>
    <tr>
      <th>98</th>
      <td>2019-05-27 17:27:54.693087</td>
      <td>-48.6256</td>
      <td>-27.5717</td>
      <td>10</td>
      <td>0.155114</td>
      <td>-0.987897</td>
      <td>fake buoy</td>
    </tr>
    <tr>
      <th>99</th>
      <td>2019-05-28 17:27:54.693087</td>
      <td>-48.6256</td>
      <td>-27.5717</td>
      <td>10</td>
      <td>0.055714</td>
      <td>-0.998447</td>
      <td>fake buoy</td>
    </tr>
  </tbody>
</table>
</div>



Let's take a look at our fake data.

<div class="prompt input_prompt">
In&nbsp;[2]:
</div>

```python
%matplotlib inline


import matplotlib.pyplot as plt
from oceans.plotting import stick_plot

q = stick_plot([t.to_pydatetime() for t in df["time"]], df["u"], df["v"])

ref = 1
qk = plt.quiverkey(
    q, 0.1, 0.85, ref, f"{ref} m s$^{-1}$", labelpos="N", coordinates="axes"
)

plt.xticks(rotation=70)
```


![png](data:image/png;base64,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ynVeRowYgdKlS+PYsWM4e/YsVq1aJdeBz5s3D++//z6mTZuGlJQUbNmyBYGBgfD09MTQoUPRpk0bLFq0CCdOnMB3330HDw8PLF++HFOnToWrqyt++OEH+Pv7Y+rUqbCyskLr1q3RvXt3PH78GIGBgZg3bx50Oh0qVKiAxo0b47PPPkPPnj2RlpaGqKgoREVFITo6GuvWrZPl9PR0bNmyJc+rnVo6kybxmjVrom7duvKsSQ8PD6xevRpDhgxB586d4ePjg2bNmqFp06by8rV5XYozR3x8vLyW+NWrV/Hhhx8CUG4UkXOth5CQENSuXRtFixZFTEwMQkJCMGPGDKSlpcHLywu//fYbdDod+vfvj19++QWlS5fG8OHD0bJlS3Tq1Ane3t64fv06tm7din379mH58uX4+eefERcXBxcXFzg4OGDOnDnw9vbG9u3b4ezsjJYtW2L//v04c+YM3nrrLVSqVAlhYWFYuXIlypUrhxIlSiA1NRVpaWlITk7G/v374e/vL39XrVo12NjYoFKlSvj4449RpkwZZGZmwtraGkSExMREnDt3DgBQuXJlefJFzocfUE7OyDnpiYjkzm78syk9KybjuI3bk52dLb+UHz58iD179iAxMRFxcXFISUlBdna26vFZWVl44403UKpUKQDKDRdOnjyJffv2ISMjA2lpaShSpAgqV66M+/fvY+HChUhMTETJkiXRpEkTXL58GcuWLcN7772HkSNHYvv27bhz5w7Gjx+P5ORkuLq6YvTo0di6dSsGDhyIr776Ct9//z3mzp2LSZMmYebMmZg4cSL69u2L9evXw9vbGz179sTOnTvRvXt3tGvXDl27dsXbb7+NN998E2FhYahVqxZatGiBEydOoFWrVvKkn/r166N69erw9/cH8Pxrp4SFhaFmzZoAgL1798qLcK1Zs0ZeL8jX1xd2dnYoWbIkLl68iOjoaDg4OODevXuYPXs2tm/fjvj4eJnAa9asCXd3d9jb26Nbt26YMmUKYmJisGnTJmzduhUdO3bEkCFDULduXXh7e6NChQpo3rw5srOzsWPHDhw5cgQ2NjawsbFBWFgY1q9fjzJlysizPokISUlJSEpKwoMHD/D777+jWLFiqFSpEsqVK4eKFSuiSpUqqFOnDipUqIDKlSujWLFi8m+ce//Q0ufgpbxKN97438sMpxAph4I5h5nZ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
)


`pocean.dsg` is relatively simple to use. The user must provide a DataFrame, like the one above, and a dictionary of attributes that maps to the data and adhere to the DSG conventions desired. 

Because we want the file to work seamlessly with ERDDAP we also added some ERDDAP specific attributes like `cdm_timeseries_variables`, and `subsetVariables`.

<div class="prompt input_prompt">
In&nbsp;[3]:
</div>

```python
attributes = {
    "global": {
        "title": "Fake mooring",
        "summary": "Vector current meter ADCP @ 10 m",
        "institution": "Restaurant at the end of the universe",
        "cdm_timeseries_variables": "station",
        "subsetVariables": "depth",
    },
    "longitude": {"units": "degrees_east", "standard_name": "longitude",},
    "latitude": {"units": "degrees_north", "standard_name": "latitude",},
    "z": {"units": "m", "standard_name": "depth", "positive": "down",},
    "u": {"units": "m/s", "standard_name": "eastward_sea_water_velocity",},
    "v": {"units": "m/s", "standard_name": "northward_sea_water_velocity",},
    "station": {"cf_role": "timeseries_id"},
}
```

We also need to map the our data axes to [`pocean`'s defaults](https://github.com/pyoceans/pocean-core/blob/master/pocean/utils.py#L50-L59). This step is not needed if the data axes are already named like the default ones.

<div class="prompt input_prompt">
In&nbsp;[4]:
</div>

```python
axes = {"t": "time", "x": "longitude", "y": "latitude", "z": "depth"}
```

<div class="prompt input_prompt">
In&nbsp;[5]:
</div>

```python
from pocean.dsg.timeseries.om import OrthogonalMultidimensionalTimeseries
from pocean.utils import downcast_dataframe

df = downcast_dataframe(df)  # safely cast depth np.int64 to np.int32
dsg = OrthogonalMultidimensionalTimeseries.from_dataframe(
    df, output="fake_buoy.nc", attributes=attributes, axes=axes,
)
```

The `OrthogonalMultidimensionalTimeseries` saves the DataFrame into a CF-1.6 TimeSeries DSG.

<div class="prompt input_prompt">
In&nbsp;[6]:
</div>

```python
!ncdump -h fake_buoy.nc
```
<div class="output_area"><div class="prompt"></div>
<pre>
    netcdf fake_buoy {
    dimensions:
    	station = 1 ;
    	time = 100 ;
    variables:
    	int crs ;
    	double time(time) ;
    		time:units = "seconds since 1990-01-01 00:00:00Z" ;
    		time:standard_name = "time" ;
    		time:axis = "T" ;
    	string station(station) ;
    		station:cf_role = "timeseries_id" ;
    		station:long_name = "station identifier" ;
    	double latitude(station) ;
    		latitude:axis = "Y" ;
    		latitude:units = "degrees_north" ;
    		latitude:standard_name = "latitude" ;
    	double longitude(station) ;
    		longitude:axis = "X" ;
    		longitude:units = "degrees_east" ;
    		longitude:standard_name = "longitude" ;
    	int depth(station) ;
    		depth:_FillValue = -9999 ;
    		depth:axis = "Z" ;
    	double u(station, time) ;
    		u:_FillValue = -9999.9 ;
    		u:units = "m/s" ;
    		u:standard_name = "eastward_sea_water_velocity" ;
    		u:coordinates = "time depth longitude latitude" ;
    	double v(station, time) ;
    		v:_FillValue = -9999.9 ;
    		v:units = "m/s" ;
    		v:standard_name = "northward_sea_water_velocity" ;
    		v:coordinates = "time depth longitude latitude" ;
    
    // global attributes:
    		:Conventions = "CF-1.6" ;
    		:date_created = "2019-02-25T20:27:00Z" ;
    		:featureType = "timeseries" ;
    		:cdm_data_type = "Timeseries" ;
    		:title = "Fake mooring" ;
    		:summary = "Vector current meter ADCP @ 10 m" ;
    		:institution = "Restaurant at the end of the universe" ;
    		:cdm_timeseries_variables = "station" ;
    		:subsetVariables = "depth" ;
    }

</pre>
</div>
 It also outputs the dsg object for inspection. Let us check a few things to see if our objects was created as expected. (Note that some of the metadata was "free" due t the built-in defaults in `pocean`.

<div class="prompt input_prompt">
In&nbsp;[7]:
</div>

```python
dsg.getncattr("featureType")
```




    'timeseries'



<div class="prompt input_prompt">
In&nbsp;[8]:
</div>

```python
type(dsg)
```




    pocean.dsg.timeseries.om.OrthogonalMultidimensionalTimeseries



In addition to standard `netCDF4-python` object `.variables` method `pocean`'s DSGs provides an "categorized" version of the variables in the `data_vars`, `ancillary_vars`, and the DSG axes methods.

<div class="prompt input_prompt">
In&nbsp;[9]:
</div>

```python
[(v.standard_name) for v in dsg.data_vars()]
```




    ['eastward_sea_water_velocity', 'northward_sea_water_velocity']



<div class="prompt input_prompt">
In&nbsp;[10]:
</div>

```python
dsg.axes("T")
```




    [<class 'netCDF4._netCDF4.Variable'>
     float64 time(time)
         units: seconds since 1990-01-01 00:00:00Z
         standard_name: time
         axis: T
     unlimited dimensions: 
     current shape = (100,)
     filling on, default _FillValue of 9.969209968386869e+36 used]



<div class="prompt input_prompt">
In&nbsp;[11]:
</div>

```python
dsg.axes("Z")
```




    [<class 'netCDF4._netCDF4.Variable'>
     int32 depth(station)
         _FillValue: -9999
         axis: Z
     unlimited dimensions: 
     current shape = (1,)
     filling on]



<div class="prompt input_prompt">
In&nbsp;[12]:
</div>

```python
dsg.vatts("station")
```




    {'cf_role': 'timeseries_id', 'long_name': 'station identifier'}



<div class="prompt input_prompt">
In&nbsp;[13]:
</div>

```python
dsg["station"][:]
```




    array(['fake buoy'], dtype=object)



<div class="prompt input_prompt">
In&nbsp;[14]:
</div>

```python
dsg.vatts("u")
```




    {'_FillValue': -9999.9,
     'units': 'm/s',
     'standard_name': 'eastward_sea_water_velocity',
     'coordinates': 'time depth longitude latitude'}



We can easily round-trip back to the pandas DataFrame object.

<div class="prompt input_prompt">
In&nbsp;[15]:
</div>

```python
dsg.to_dataframe().head()
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>t</th>
      <th>x</th>
      <th>y</th>
      <th>z</th>
      <th>station</th>
      <th>u</th>
      <th>v</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>2019-02-18 17:27:54.693087</td>
      <td>-48.6256</td>
      <td>-27.5717</td>
      <td>10</td>
      <td>fake buoy</td>
      <td>-0.506366</td>
      <td>0.862319</td>
    </tr>
    <tr>
      <th>1</th>
      <td>2019-02-19 17:27:54.693087</td>
      <td>-48.6256</td>
      <td>-27.5717</td>
      <td>10</td>
      <td>fake buoy</td>
      <td>-0.417748</td>
      <td>0.908563</td>
    </tr>
    <tr>
      <th>2</th>
      <td>2019-02-20 17:27:54.693087</td>
      <td>-48.6256</td>
      <td>-27.5717</td>
      <td>10</td>
      <td>fake buoy</td>
      <td>-0.324956</td>
      <td>0.945729</td>
    </tr>
    <tr>
      <th>3</th>
      <td>2019-02-21 17:27:54.693087</td>
      <td>-48.6256</td>
      <td>-27.5717</td>
      <td>10</td>
      <td>fake buoy</td>
      <td>-0.228917</td>
      <td>0.973446</td>
    </tr>
    <tr>
      <th>4</th>
      <td>2019-02-22 17:27:54.693087</td>
      <td>-48.6256</td>
      <td>-27.5717</td>
      <td>10</td>
      <td>fake buoy</td>
      <td>-0.130591</td>
      <td>0.991436</td>
    </tr>
  </tbody>
</table>
</div>



For more information on `pocean` please check the [API docs](https://pyoceans.github.io/pocean-core/docs/api/pocean.html).
<br>
Right click and choose Save link as... to
[download](https://raw.githubusercontent.com/ioos/notebooks_demos/master/notebooks/2018-02-27-pocean-timeSeries-demo.ipynb)
this notebook, or click [here](https://binder.pangeo.io/v2/gh/ioos/notebooks_demos/master?filepath=notebooks/2018-02-27-pocean-timeSeries-demo.ipynb) to run a live instance of this notebook.