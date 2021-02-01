---
title: "Investigating ocean models skill for sea surface height with IOOS catalog and Python"
layout: notebook

---


The IOOS [catalog](https://ioos.noaa.gov/data/catalog) offers access to hundreds of datasets and data access services provided by the 11 regional associations.
In the past we demonstrate how to tap into those datasets to obtain sea [surface temperature data from observations](http://ioos.github.io/notebooks_demos/notebooks/2016-12-19-exploring_csw),
[coastal velocity from high frequency radar data](http://ioos.github.io/notebooks_demos/notebooks/2017-12-15-finding_HFRadar_currents),
and a simple model vs observation visualization of temperatures for the [Boston Light Swim competition](http://ioos.github.io/notebooks_demos/notebooks/2016-12-22-boston_light_swim).

In this notebook we'll demonstrate a step-by-step workflow on how ask the catalog for a specific variable, extract only the model data, and match the nearest model grid point to an observation. The goal is to create an automated skill score for quick assessment of ocean numerical models.


The first cell is only to reduce iris' noisy output,
the notebook start on cell [2] with the definition of the parameters:
- start and end dates for the search;
- experiment name;
- a bounding of the region of interest;
- SOS variable name for the observations;
- Climate and Forecast standard names;
- the units we want conform the variables into;
- catalogs we want to search.

<div class="prompt input_prompt">
In&nbsp;[1]:
</div>

```python
import warnings

# Suppresing warnings for a "pretty output."
warnings.simplefilter("ignore")
```

<div class="prompt input_prompt">
In&nbsp;[2]:
</div>

```python
%%writefile config.yaml

date:
    start: 2018-2-28 00:00:00
    stop: 2018-3-5 00:00:00

run_name: 'latest'

region:
    bbox: [-71.20, 41.40, -69.20, 43.74]
    crs: 'urn:ogc:def:crs:OGC:1.3:CRS84'

sos_name: 'water_surface_height_above_reference_datum'

cf_names:
    - sea_surface_height
    - sea_surface_elevation
    - sea_surface_height_above_geoid
    - sea_surface_height_above_sea_level
    - water_surface_height_above_reference_datum
    - sea_surface_height_above_reference_ellipsoid

units: 'm'

catalogs:
    - https://data.ioos.us/csw
```
<div class="output_area"><div class="prompt"></div>
<pre>
    Overwriting config.yaml

</pre>
</div>
To keep track of the information we'll setup a `config` variable and output them on the screen for bookkeeping.

<div class="prompt input_prompt">
In&nbsp;[3]:
</div>

```python
import os
import shutil
from datetime import datetime

from ioos_tools.ioos import parse_config

config = parse_config("config.yaml")

# Saves downloaded data into a temporary directory.
save_dir = os.path.abspath(config["run_name"])
if os.path.exists(save_dir):
    shutil.rmtree(save_dir)
os.makedirs(save_dir)

fmt = "{:*^64}".format
print(fmt("Saving data inside directory {}".format(save_dir)))
print(fmt(" Run information "))
print("Run date: {:%Y-%m-%d %H:%M:%S}".format(datetime.utcnow()))
print("Start: {:%Y-%m-%d %H:%M:%S}".format(config["date"]["start"]))
print("Stop: {:%Y-%m-%d %H:%M:%S}".format(config["date"]["stop"]))
print(
    "Bounding box: {0:3.2f}, {1:3.2f},"
    "{2:3.2f}, {3:3.2f}".format(*config["region"]["bbox"])
)
```
<div class="output_area"><div class="prompt"></div>
<pre>
    Saving data inside directory /home/filipe/IOOS/notebooks_demos/notebooks/latest
    *********************** Run information ************************
    Run date: 2018-11-30 13:25:17
    Start: 2018-02-28 00:00:00
    Stop: 2018-03-05 00:00:00
    Bounding box: -71.20, 41.40,-69.20, 43.74

</pre>
</div>
To interface with the IOOS catalog we will use the [Catalogue Service for the Web (CSW)](https://live.osgeo.org/en/standards/csw_overview.html) endpoint and [python's OWSLib library](https://geopython.github.io/OWSLib).

The cell below creates the [Filter Encoding Specification (FES)](http://www.opengeospatial.org/standards/filter) with configuration we specified in cell [2]. The filter is composed of:
- `or` to catch any of the standard names;
- `not` some names we do not want to show up in the results;
- `date range` and `bounding box` for the time-space domain of the search.

<div class="prompt input_prompt">
In&nbsp;[4]:
</div>

```python
def make_filter(config):
    from owslib import fes
    from ioos_tools.ioos import fes_date_filter

    kw = dict(
        wildCard="*", escapeChar="\\", singleChar="?", propertyname="apiso:Subject"
    )

    or_filt = fes.Or(
        [fes.PropertyIsLike(literal=("*%s*" % val), **kw) for val in config["cf_names"]]
    )

    not_filt = fes.Not([fes.PropertyIsLike(literal="GRIB-2", **kw)])

    begin, end = fes_date_filter(config["date"]["start"], config["date"]["stop"])

    bbox_crs = fes.BBox(config["region"]["bbox"], crs=config["region"]["crs"])

    filter_list = [fes.And([bbox_crs, begin, end, or_filt, not_filt])]
    return filter_list


filter_list = make_filter(config)
```

We need to wrap `OWSlib.csw.CatalogueServiceWeb` object with a custom function,
` get_csw_records`, to be able to paginate over the results.

In the cell below we loop over all the catalogs returns and extract the OPeNDAP endpoints.

<div class="prompt input_prompt">
In&nbsp;[5]:
</div>

```python
from ioos_tools.ioos import get_csw_records, service_urls
from owslib.csw import CatalogueServiceWeb

dap_urls = []
print(fmt(" Catalog information "))
for endpoint in config["catalogs"]:
    print("URL: {}".format(endpoint))
    try:
        csw = CatalogueServiceWeb(endpoint, timeout=120)
    except Exception as e:
        print("{}".format(e))
        continue
    csw = get_csw_records(csw, filter_list, esn="full")
    OPeNDAP = service_urls(csw.records, identifier="OPeNDAP:OPeNDAP")
    odp = service_urls(
        csw.records, identifier="urn:x-esri:specification:ServiceType:odp:url"
    )
    dap = OPeNDAP + odp
    dap_urls.extend(dap)

    print("Number of datasets available: {}".format(len(csw.records.keys())))

    for rec, item in csw.records.items():
        print("{}".format(item.title))
    if dap:
        print(fmt(" DAP "))
        for url in dap:
            print("{}.html".format(url))
    print("\n")

# Get only unique endpoints.
dap_urls = list(set(dap_urls))
```
<div class="output_area"><div class="prompt"></div>
<pre>
    ********************* Catalog information **********************
    URL: https://data.ioos.us/csw
    Number of datasets available: 13
    urn:ioos:station:NOAA.NOS.CO-OPS:8447386 station, Fall River, MA
    urn:ioos:station:NOAA.NOS.CO-OPS:8447435 station, Chatham, Lydia Cove, MA
    urn:ioos:station:NOAA.NOS.CO-OPS:8447930 station, Woods Hole, MA
    Coupled Northwest Atlantic Prediction System (CNAPS)
    HYbrid Coordinate Ocean Model (HYCOM): Global
    NECOFS (FVCOM) - Scituate - Latest Forecast
    NECOFS Massachusetts (FVCOM) - Boston - Latest Forecast
    ROMS ESPRESSO Real-Time Operational IS4DVAR Forecast System Version 2 (NEW) 2013-present FMRC Averages
    ROMS ESPRESSO Real-Time Operational IS4DVAR Forecast System Version 2 (NEW) 2013-present FMRC History
    urn:ioos:station:NOAA.NOS.CO-OPS:8418150 station, Portland, ME
    urn:ioos:station:NOAA.NOS.CO-OPS:8419317 station, Wells, ME
    urn:ioos:station:NOAA.NOS.CO-OPS:8423898 station, Fort Point, NH
    urn:ioos:station:NOAA.NOS.CO-OPS:8443970 station, Boston, MA
    ***************************** DAP ******************************
    http://oos.soest.hawaii.edu/thredds/dodsC/pacioos/hycom/global.html
    http://tds.marine.rutgers.edu/thredds/dodsC/roms/espresso/2013_da/avg/ESPRESSO_Real-Time_v2_Averages_Best.html
    http://tds.marine.rutgers.edu/thredds/dodsC/roms/espresso/2013_da/his/ESPRESSO_Real-Time_v2_History_Best.html
    http://thredds.secoora.org/thredds/dodsC/SECOORA_NCSU_CNAPS.nc.html
    http://www.smast.umassd.edu:8080/thredds/dodsC/FVCOM/NECOFS/Forecasts/NECOFS_FVCOM_OCEAN_BOSTON_FORECAST.nc.html
    http://www.smast.umassd.edu:8080/thredds/dodsC/FVCOM/NECOFS/Forecasts/NECOFS_FVCOM_OCEAN_SCITUATE_FORECAST.nc.html
    https://opendap.co-ops.nos.noaa.gov/ioos-dif-sos/images/tide_gauge.jpg.html
    
    

</pre>
</div>
We found 10 dataset endpoints but only 9 of them have the proper metadata for us to identify the OPeNDAP endpoint,
those that contain either `OPeNDAP:OPeNDAP` or `urn:x-esri:specification:ServiceType:odp:url` scheme.
Unfortunately we lost the `COAWST` model in the process.

The next step is to ensure there are no observations in the list of endpoints.
We want only the models for now.

<div class="prompt input_prompt">
In&nbsp;[6]:
</div>

```python
from ioos_tools.ioos import is_station
from timeout_decorator import TimeoutError

# Filter out some station endpoints.
non_stations = []
for url in dap_urls:
    try:
        if not is_station(url):
            non_stations.append(url)
    except (IOError, OSError, RuntimeError, TimeoutError) as e:
        print("Could not access URL {}.html\n{!r}".format(url, e))

dap_urls = non_stations

print(fmt(" Filtered DAP "))
for url in dap_urls:
    print("{}.html".format(url))
```
<div class="output_area"><div class="prompt"></div>
<pre>
    Could not access URL https://opendap.co-ops.nos.noaa.gov/ioos-dif-sos/images/tide_gauge.jpg.html
    OSError(-90, 'NetCDF: file not found')
    ************************* Filtered DAP *************************
    http://www.smast.umassd.edu:8080/thredds/dodsC/FVCOM/NECOFS/Forecasts/NECOFS_FVCOM_OCEAN_BOSTON_FORECAST.nc.html
    http://www.smast.umassd.edu:8080/thredds/dodsC/FVCOM/NECOFS/Forecasts/NECOFS_FVCOM_OCEAN_SCITUATE_FORECAST.nc.html
    http://thredds.secoora.org/thredds/dodsC/SECOORA_NCSU_CNAPS.nc.html
    http://tds.marine.rutgers.edu/thredds/dodsC/roms/espresso/2013_da/avg/ESPRESSO_Real-Time_v2_Averages_Best.html
    http://tds.marine.rutgers.edu/thredds/dodsC/roms/espresso/2013_da/his/ESPRESSO_Real-Time_v2_History_Best.html
    http://oos.soest.hawaii.edu/thredds/dodsC/pacioos/hycom/global.html

</pre>
</div>
Now we have a nice list of all the models available in the catalog for the domain we specified.
We still need to find the observations for the same domain.
To accomplish that we will use the `pyoos` library and search the [SOS CO-OPS](https://opendap.co-ops.nos.noaa.gov/ioos-dif-sos/) services using the virtually the same configuration options from the catalog search.

<div class="prompt input_prompt">
In&nbsp;[7]:
</div>

```python
from pyoos.collectors.coops.coops_sos import CoopsSos

collector_coops = CoopsSos()

collector_coops.set_bbox(config["region"]["bbox"])
collector_coops.end_time = config["date"]["stop"]
collector_coops.start_time = config["date"]["start"]
collector_coops.variables = [config["sos_name"]]

ofrs = collector_coops.server.offerings
title = collector_coops.server.identification.title
print(fmt(" Collector offerings "))
print("{}: {} offerings".format(title, len(ofrs)))
```
<div class="output_area"><div class="prompt"></div>
<pre>
    ********************* Collector offerings **********************
    NOAA.NOS.CO-OPS SOS: 1229 offerings

</pre>
</div>
To make it easier to work with the data we extract the time-series as pandas tables and interpolate them to a common 1-hour interval index.

<div class="prompt input_prompt">
In&nbsp;[8]:
</div>

```python
import pandas as pd
from ioos_tools.ioos import collector2table

data = collector2table(
    collector=collector_coops,
    config=config,
    col="water_surface_height_above_reference_datum (m)",
)

df = dict(
    station_name=[s._metadata.get("station_name") for s in data],
    station_code=[s._metadata.get("station_code") for s in data],
    sensor=[s._metadata.get("sensor") for s in data],
    lon=[s._metadata.get("lon") for s in data],
    lat=[s._metadata.get("lat") for s in data],
    depth=[s._metadata.get("depth") for s in data],
)

pd.DataFrame(df).set_index("station_code")
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
      <th>station_name</th>
      <th>sensor</th>
      <th>lon</th>
      <th>lat</th>
      <th>depth</th>
    </tr>
    <tr>
      <th>station_code</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>8418150</th>
      <td>Portland, ME</td>
      <td>urn:ioos:sensor:NOAA.NOS.CO-OPS:8418150:A1</td>
      <td>-70.2461</td>
      <td>43.6561</td>
      <td>None</td>
    </tr>
    <tr>
      <th>8419317</th>
      <td>Wells, ME</td>
      <td>urn:ioos:sensor:NOAA.NOS.CO-OPS:8419317:B1</td>
      <td>-70.5633</td>
      <td>43.3200</td>
      <td>None</td>
    </tr>
    <tr>
      <th>8443970</th>
      <td>Boston, MA</td>
      <td>urn:ioos:sensor:NOAA.NOS.CO-OPS:8443970:Y1</td>
      <td>-71.0503</td>
      <td>42.3539</td>
      <td>None</td>
    </tr>
    <tr>
      <th>8447386</th>
      <td>Fall River, MA</td>
      <td>urn:ioos:sensor:NOAA.NOS.CO-OPS:8447386:B1</td>
      <td>-71.1641</td>
      <td>41.7043</td>
      <td>None</td>
    </tr>
    <tr>
      <th>8447435</th>
      <td>Chatham, Lydia Cove, MA</td>
      <td>urn:ioos:sensor:NOAA.NOS.CO-OPS:8447435:A1</td>
      <td>-69.9510</td>
      <td>41.6885</td>
      <td>None</td>
    </tr>
    <tr>
      <th>8447930</th>
      <td>Woods Hole, MA</td>
      <td>urn:ioos:sensor:NOAA.NOS.CO-OPS:8447930:A1</td>
      <td>-70.6711</td>
      <td>41.5236</td>
      <td>None</td>
    </tr>
  </tbody>
</table>
</div>



<div class="prompt input_prompt">
In&nbsp;[9]:
</div>

```python
index = pd.date_range(
    start=config["date"]["start"].replace(tzinfo=None),
    end=config["date"]["stop"].replace(tzinfo=None),
    freq="1H",
)

# Preserve metadata with `reindex`.
observations = []
for series in data:
    _metadata = series._metadata
    series.index = series.index.tz_localize(None)
    obs = series.reindex(index=index, limit=1, method="nearest")
    obs._metadata = _metadata
    observations.append(obs)
```

The next cell saves those time-series as CF-compliant netCDF files on disk,
to make it easier to access them later.

<div class="prompt input_prompt">
In&nbsp;[10]:
</div>

```python
import iris
from ioos_tools.tardis import series2cube

attr = dict(
    featureType="timeSeries",
    Conventions="CF-1.6",
    standard_name_vocabulary="CF-1.6",
    cdm_data_type="Station",
    comment="Data from http://opendap.co-ops.nos.noaa.gov",
)


cubes = iris.cube.CubeList([series2cube(obs, attr=attr) for obs in observations])

outfile = os.path.join(save_dir, "OBS_DATA.nc")
iris.save(cubes, outfile)
```

We still need to read the model data from the list of endpoints we found.

The next cell takes care of that.
We use `iris`, and a set of custom functions from the `ioos_tools` library,
that downloads only the data in the domain we requested.

<div class="prompt input_prompt">
In&nbsp;[11]:
</div>

```python
from ioos_tools.ioos import get_model_name
from ioos_tools.tardis import is_model, proc_cube, quick_load_cubes
from iris.exceptions import ConstraintMismatchError, CoordinateNotFoundError, MergeError

print(fmt(" Models "))
cubes = dict()
for k, url in enumerate(dap_urls):
    print("\n[Reading url {}/{}]: {}".format(k + 1, len(dap_urls), url))
    try:
        cube = quick_load_cubes(url, config["cf_names"], callback=None, strict=True)
        if is_model(cube):
            cube = proc_cube(
                cube,
                bbox=config["region"]["bbox"],
                time=(config["date"]["start"], config["date"]["stop"]),
                units=config["units"],
            )
        else:
            print("[Not model data]: {}".format(url))
            continue
        mod_name = get_model_name(url)
        cubes.update({mod_name: cube})
    except (
        RuntimeError,
        ValueError,
        ConstraintMismatchError,
        CoordinateNotFoundError,
        IndexError,
    ) as e:
        print("Cannot get cube for: {}\n{}".format(url, e))
```
<div class="output_area"><div class="prompt"></div>
<pre>
    **************************** Models ****************************
    
    [Reading url 1/6]: http://www.smast.umassd.edu:8080/thredds/dodsC/FVCOM/NECOFS/Forecasts/NECOFS_FVCOM_OCEAN_BOSTON_FORECAST.nc
    
    [Reading url 2/6]: http://www.smast.umassd.edu:8080/thredds/dodsC/FVCOM/NECOFS/Forecasts/NECOFS_FVCOM_OCEAN_SCITUATE_FORECAST.nc
    
    [Reading url 3/6]: http://thredds.secoora.org/thredds/dodsC/SECOORA_NCSU_CNAPS.nc
    
    [Reading url 4/6]: http://tds.marine.rutgers.edu/thredds/dodsC/roms/espresso/2013_da/avg/ESPRESSO_Real-Time_v2_Averages_Best
    
    [Reading url 5/6]: http://tds.marine.rutgers.edu/thredds/dodsC/roms/espresso/2013_da/his/ESPRESSO_Real-Time_v2_History_Best
    
    [Reading url 6/6]: http://oos.soest.hawaii.edu/thredds/dodsC/pacioos/hycom/global

</pre>
</div>
Now we can match each observation time-series with its closest grid point (0.08 of a degree) on each model.
This is a complex and laborious task! If you are running this interactively grab a coffee and sit comfortably :-)

Note that we are also saving the model time-series to files that align with the observations we saved before.

<div class="prompt input_prompt">
In&nbsp;[12]:
</div>

```python
import iris
from ioos_tools.tardis import (
    add_station,
    ensure_timeseries,
    get_nearest_water,
    make_tree,
)
from iris.pandas import as_series

for mod_name, cube in cubes.items():
    fname = "{}.nc".format(mod_name)
    fname = os.path.join(save_dir, fname)
    print(fmt(" Downloading to file {} ".format(fname)))
    try:
        tree, lon, lat = make_tree(cube)
    except CoordinateNotFoundError:
        print("Cannot make KDTree for: {}".format(mod_name))
        continue
    # Get model series at observed locations.
    raw_series = dict()
    for obs in observations:
        obs = obs._metadata
        station = obs["station_code"]
        try:
            kw = dict(k=10, max_dist=0.08, min_var=0.01)
            args = cube, tree, obs["lon"], obs["lat"]
            try:
                series, dist, idx = get_nearest_water(*args, **kw)
            except RuntimeError as e:
                print("Cannot download {!r}.\n{}".format(cube, e))
                series = None
        except ValueError:
            status = "No Data"
            print("[{}] {}".format(status, obs["station_name"]))
            continue
        if not series:
            status = "Land   "
        else:
            raw_series.update({station: series})
            series = as_series(series)
            status = "Water  "
        print("[{}] {}".format(status, obs["station_name"]))
    if raw_series:  # Save cube.
        for station, cube in raw_series.items():
            cube = add_station(cube, station)
        try:
            cube = iris.cube.CubeList(raw_series.values()).merge_cube()
        except MergeError as e:
            print(e)
        ensure_timeseries(cube)
        try:
            iris.save(cube, fname)
        except AttributeError:
            # FIXME: we should patch the bad attribute instead of removing everything.
            cube.attributes = {}
            iris.save(cube, fname)
        del cube
    print("Finished processing [{}]".format(mod_name))
```
<div class="output_area"><div class="prompt"></div>
<pre>
     Downloading to file /home/filipe/IOOS/notebooks_demos/notebooks/latest/Forecasts-NECOFS_FVCOM_OCEAN_BOSTON_FORECAST.nc 
    [No Data] Portland, ME
    [No Data] Wells, ME
    [Water  ] Boston, MA
    [No Data] Fall River, MA
    [No Data] Chatham, Lydia Cove, MA
    [No Data] Woods Hole, MA
    Finished processing [Forecasts-NECOFS_FVCOM_OCEAN_BOSTON_FORECAST]
     Downloading to file /home/filipe/IOOS/notebooks_demos/notebooks/latest/Forecasts-NECOFS_FVCOM_OCEAN_SCITUATE_FORECAST.nc 
    [No Data] Portland, ME
    [No Data] Wells, ME
    [No Data] Boston, MA
    [No Data] Fall River, MA
    [No Data] Chatham, Lydia Cove, MA
    [No Data] Woods Hole, MA
    Finished processing [Forecasts-NECOFS_FVCOM_OCEAN_SCITUATE_FORECAST]
     Downloading to file /home/filipe/IOOS/notebooks_demos/notebooks/latest/SECOORA_NCSU_CNAPS.nc 
    [Land   ] Portland, ME
    [Water  ] Wells, ME
    [Land   ] Boston, MA
    [Land   ] Fall River, MA
    [Land   ] Chatham, Lydia Cove, MA
    [Water  ] Woods Hole, MA
    Finished processing [SECOORA_NCSU_CNAPS]
     Downloading to file /home/filipe/IOOS/notebooks_demos/notebooks/latest/roms_2013_da_avg-ESPRESSO_Real-Time_v2_Averages_Best.nc 
    [No Data] Portland, ME
    [No Data] Wells, ME
    [Land   ] Boston, MA
    [Land   ] Fall River, MA
    [Water  ] Chatham, Lydia Cove, MA
    [Water  ] Woods Hole, MA
    Finished processing [roms_2013_da_avg-ESPRESSO_Real-Time_v2_Averages_Best]
     Downloading to file /home/filipe/IOOS/notebooks_demos/notebooks/latest/roms_2013_da-ESPRESSO_Real-Time_v2_History_Best.nc 
    [No Data] Portland, ME
    [No Data] Wells, ME
    [Land   ] Boston, MA
    [Land   ] Fall River, MA
    [Water  ] Chatham, Lydia Cove, MA
    [Water  ] Woods Hole, MA
    Finished processing [roms_2013_da-ESPRESSO_Real-Time_v2_History_Best]
     Downloading to file /home/filipe/IOOS/notebooks_demos/notebooks/latest/pacioos_hycom-global.nc 
    [Land   ] Portland, ME
    [Water  ] Wells, ME
    [Land   ] Boston, MA
    [Land   ] Fall River, MA
    [Water  ] Chatham, Lydia Cove, MA
    [Land   ] Woods Hole, MA
    Finished processing [pacioos_hycom-global]

</pre>
</div>
With the matched set of models and observations time-series it is relatively easy to compute skill score metrics on them. In cells [13] to [16] we apply both mean bias and root mean square errors to the time-series.

<div class="prompt input_prompt">
In&nbsp;[13]:
</div>

```python
from ioos_tools.ioos import stations_keys


def rename_cols(df, config):
    cols = stations_keys(config, key="station_name")
    return df.rename(columns=cols)
```

<div class="prompt input_prompt">
In&nbsp;[14]:
</div>

```python
from ioos_tools.ioos import load_ncs
from ioos_tools.skill_score import apply_skill, mean_bias

dfs = load_ncs(config)

df = apply_skill(dfs, mean_bias, remove_mean=False, filter_tides=False)
skill_score = dict(mean_bias=df.to_dict())

# Filter out stations with no valid comparison.
df.dropna(how="all", axis=1, inplace=True)
df = df.applymap("{:.2f}".format).replace("nan", "--")
```

<div class="prompt input_prompt">
In&nbsp;[15]:
</div>

```python
from ioos_tools.skill_score import rmse

dfs = load_ncs(config)

df = apply_skill(dfs, rmse, remove_mean=True, filter_tides=False)
skill_score["rmse"] = df.to_dict()

# Filter out stations with no valid comparison.
df.dropna(how="all", axis=1, inplace=True)
df = df.applymap("{:.2f}".format).replace("nan", "--")
```

<div class="prompt input_prompt">
In&nbsp;[16]:
</div>

```python
import pandas as pd

# Stringfy keys.
for key in skill_score.keys():
    skill_score[key] = {str(k): v for k, v in skill_score[key].items()}

mean_bias = pd.DataFrame.from_dict(skill_score["mean_bias"])
mean_bias = mean_bias.applymap("{:.2f}".format).replace("nan", "--")

skill_score = pd.DataFrame.from_dict(skill_score["rmse"])
skill_score = skill_score.applymap("{:.2f}".format).replace("nan", "--")
```

Last but not least we can assemble a GIS map, cells [17-23],
with the time-series plot for the observations and models,
and the corresponding skill scores.

<div class="prompt input_prompt">
In&nbsp;[17]:
</div>

```python
import folium
from ioos_tools.ioos import get_coordinates


def make_map(bbox, **kw):
    line = kw.pop("line", True)
    zoom_start = kw.pop("zoom_start", 5)

    lon = (bbox[0] + bbox[2]) / 2
    lat = (bbox[1] + bbox[3]) / 2
    m = folium.Map(
        width="100%", height="100%", location=[lat, lon], zoom_start=zoom_start
    )

    if line:
        p = folium.PolyLine(
            get_coordinates(bbox), color="#FF0000", weight=2, opacity=0.9,
        )
        p.add_to(m)
    return m
```

<div class="prompt input_prompt">
In&nbsp;[18]:
</div>

```python
bbox = config["region"]["bbox"]

m = make_map(bbox, zoom_start=8, line=True, layers=True)
```

<div class="prompt input_prompt">
In&nbsp;[19]:
</div>

```python
all_obs = stations_keys(config)

from glob import glob
from operator import itemgetter

import iris
from folium.plugins import MarkerCluster

iris.FUTURE.netcdf_promote = True

big_list = []
for fname in glob(os.path.join(save_dir, "*.nc")):
    if "OBS_DATA" in fname:
        continue
    cube = iris.load_cube(fname)
    model = os.path.split(fname)[1].split("-")[-1].split(".")[0]
    lons = cube.coord(axis="X").points
    lats = cube.coord(axis="Y").points
    stations = cube.coord("station_code").points
    models = [model] * lons.size
    lista = zip(models, lons.tolist(), lats.tolist(), stations.tolist())
    big_list.extend(lista)

big_list.sort(key=itemgetter(3))
df = pd.DataFrame(big_list, columns=["name", "lon", "lat", "station"])
df.set_index("station", drop=True, inplace=True)
groups = df.groupby(df.index)


locations, popups = [], []
for station, info in groups:
    sta_name = all_obs[station]
    for lat, lon, name in zip(info.lat, info.lon, info.name):
        locations.append([lat, lon])
        popups.append("[{}]: {}".format(name, sta_name))

MarkerCluster(locations=locations, popups=popups, name="Cluster").add_to(m)
```

<div class="prompt input_prompt">
In&nbsp;[20]:
</div>

```python
titles = {
    "coawst_4_use_best": "COAWST_4",
    "pacioos_hycom-global": "HYCOM",
    "NECOFS_GOM3_FORECAST": "NECOFS_GOM3",
    "NECOFS_FVCOM_OCEAN_MASSBAY_FORECAST": "NECOFS_MassBay",
    "NECOFS_FVCOM_OCEAN_BOSTON_FORECAST": "NECOFS_Boston",
    "SECOORA_NCSU_CNAPS": "SECOORA/CNAPS",
    "roms_2013_da_avg-ESPRESSO_Real-Time_v2_Averages_Best": "ESPRESSO Avg",
    "roms_2013_da-ESPRESSO_Real-Time_v2_History_Best": "ESPRESSO Hist",
    "OBS_DATA": "Observations",
}
```

<div class="prompt input_prompt">
In&nbsp;[21]:
</div>

```python
from itertools import cycle

from bokeh.embed import file_html
from bokeh.models import HoverTool, Legend
from bokeh.palettes import Category20
from bokeh.plotting import figure
from bokeh.resources import CDN
from folium import IFrame

# Plot defaults.
colors = Category20[20]
colorcycler = cycle(colors)
tools = "pan,box_zoom,reset"
width, height = 750, 250


def make_plot(df, station):
    p = figure(
        toolbar_location="above",
        x_axis_type="datetime",
        width=width,
        height=height,
        tools=tools,
        title=str(station),
    )
    leg = []
    for column, series in df.iteritems():
        series.dropna(inplace=True)
        if not series.empty:
            if "OBS_DATA" not in column:
                bias = mean_bias[str(station)][column]
                skill = skill_score[str(station)][column]
                line_color = next(colorcycler)
                kw = dict(alpha=0.65, line_color=line_color)
            else:
                skill = bias = "NA"
                kw = dict(alpha=1, color="crimson")
            line = p.line(
                x=series.index,
                y=series.values,
                line_width=5,
                line_cap="round",
                line_join="round",
                **kw
            )
            leg.append(("{}".format(titles.get(column, column)), [line]))
            p.add_tools(
                HoverTool(
                    tooltips=[
                        ("Name", "{}".format(titles.get(column, column))),
                        ("Bias", bias),
                        ("Skill", skill),
                    ],
                    renderers=[line],
                )
            )
    legend = Legend(items=leg, location=(0, 60))
    legend.click_policy = "mute"
    p.add_layout(legend, "right")
    p.yaxis[0].axis_label = "Water Height (m)"
    p.xaxis[0].axis_label = "Date/time"
    return p


def make_marker(p, station):
    lons = stations_keys(config, key="lon")
    lats = stations_keys(config, key="lat")

    lon, lat = lons[station], lats[station]
    html = file_html(p, CDN, station)
    iframe = IFrame(html, width=width + 40, height=height + 80)

    popup = folium.Popup(iframe, max_width=2650)
    icon = folium.Icon(color="green", icon="stats")
    marker = folium.Marker(location=[lat, lon], popup=popup, icon=icon)
    return marker
```

<div class="prompt input_prompt">
In&nbsp;[22]:
</div>

```python
dfs = load_ncs(config)

for station in dfs:
    sta_name = all_obs[station]
    df = dfs[station]
    if df.empty:
        continue
    p = make_plot(df, station)
    marker = make_marker(p, station)
    marker.add_to(m)

folium.LayerControl().add_to(m)
```

<div class="prompt input_prompt">
In&nbsp;[23]:
</div>

```python
def embed_map(m):
    from IPython.display import HTML

    m.save("index.html")
    with open("index.html") as f:
        html = f.read()

    iframe = '<iframe srcdoc="{srcdoc}" style="width: 100%; height: 750px; border: none"></iframe>'
    srcdoc = html.replace('"', "&quot;")
    return HTML(iframe.format(srcdoc=srcdoc))


embed_map(m)
```




<iframe srcdoc="<!DOCTYPE html>
<head>    
    <meta http-equiv=&quot;content-type&quot; content=&quot;text/html; charset=UTF-8&quot; />
    <script>L_PREFER_CANVAS=false; L_NO_TOUCH=false; L_DISABLE_3D=false;</script>
    <script src=&quot;https://cdn.jsdelivr.net/npm/leaflet@1.3.4/dist/leaflet.js&quot;></script>
    <script src=&quot;https://ajax.googleapis.com/ajax/libs/jquery/1.11.1/jquery.min.js&quot;></script>
    <script src=&quot;https://maxcdn.bootstrapcdn.com/bootstrap/3.2.0/js/bootstrap.min.js&quot;></script>
    <script src=&quot;https://cdnjs.cloudflare.com/ajax/libs/Leaflet.awesome-markers/2.0.2/leaflet.awesome-markers.js&quot;></script>
    <link rel=&quot;stylesheet&quot; href=&quot;https://cdn.jsdelivr.net/npm/leaflet@1.3.4/dist/leaflet.css&quot;/>
    <link rel=&quot;stylesheet&quot; href=&quot;https://maxcdn.bootstrapcdn.com/bootstrap/3.2.0/css/bootstrap.min.css&quot;/>
    <link rel=&quot;stylesheet&quot; href=&quot;https://maxcdn.bootstrapcdn.com/bootstrap/3.2.0/css/bootstrap-theme.min.css&quot;/>
    <link rel=&quot;stylesheet&quot; href=&quot;https://maxcdn.bootstrapcdn.com/font-awesome/4.6.3/css/font-awesome.min.css&quot;/>
    <link rel=&quot;stylesheet&quot; href=&quot;https://cdnjs.cloudflare.com/ajax/libs/Leaflet.awesome-markers/2.0.2/leaflet.awesome-markers.css&quot;/>
    <link rel=&quot;stylesheet&quot; href=&quot;https://rawcdn.githack.com/python-visualization/folium/master/folium/templates/leaflet.awesome.rotate.css&quot;/>
    <style>html, body {width: 100%;height: 100%;margin: 0;padding: 0;}</style>
    <style>#map {position:absolute;top:0;bottom:0;right:0;left:0;}</style>

    <meta name=&quot;viewport&quot; content=&quot;width=device-width,
        initial-scale=1.0, maximum-scale=1.0, user-scalable=no&quot; />
    <style>#map_54cb488c2c244a8daafaa65ad6b9b54b {
        position: relative;
        width: 100.0%;
        height: 100.0%;
        left: 0.0%;
        top: 0.0%;
        }
    </style>
    <script src=&quot;https://cdnjs.cloudflare.com/ajax/libs/leaflet.markercluster/1.1.0/leaflet.markercluster.js&quot;></script>
    <link rel=&quot;stylesheet&quot; href=&quot;https://cdnjs.cloudflare.com/ajax/libs/leaflet.markercluster/1.1.0/MarkerCluster.css&quot;/>
    <link rel=&quot;stylesheet&quot; href=&quot;https://cdnjs.cloudflare.com/ajax/libs/leaflet.markercluster/1.1.0/MarkerCluster.Default.css&quot;/>
</head>
<body>    

    <div class=&quot;folium-map&quot; id=&quot;map_54cb488c2c244a8daafaa65ad6b9b54b&quot; ></div>
</body>
<script>    


        var bounds = null;


    var map_54cb488c2c244a8daafaa65ad6b9b54b = L.map(
        'map_54cb488c2c244a8daafaa65ad6b9b54b', {
        center: [42.57, -70.2],
        zoom: 8,
        maxBounds: bounds,
        layers: [],
        worldCopyJump: false,
        crs: L.CRS.EPSG3857,
        zoomControl: true,
        });



    var tile_layer_378680a0ecc34181a212e41b10fee026 = L.tileLayer(
        'https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png',
        {
        &quot;attribution&quot;: null,
        &quot;detectRetina&quot;: false,
        &quot;maxNativeZoom&quot;: 18,
        &quot;maxZoom&quot;: 18,
        &quot;minZoom&quot;: 0,
        &quot;noWrap&quot;: false,
        &quot;opacity&quot;: 1,
        &quot;subdomains&quot;: &quot;abc&quot;,
        &quot;tms&quot;: false
}).addTo(map_54cb488c2c244a8daafaa65ad6b9b54b);

                var poly_line_f9f8e202681d40a9b28404b58db76e21 = L.polyline(
                    [[41.4, -71.2], [41.4, -69.2], [43.74, -69.2], [43.74, -71.2], [41.4, -71.2]],
                    {
  &quot;bubblingMouseEvents&quot;: true,
  &quot;color&quot;: &quot;#FF0000&quot;,
  &quot;dashArray&quot;: null,
  &quot;dashOffset&quot;: null,
  &quot;fill&quot;: false,
  &quot;fillColor&quot;: &quot;#FF0000&quot;,
  &quot;fillOpacity&quot;: 0.2,
  &quot;fillRule&quot;: &quot;evenodd&quot;,
  &quot;lineCap&quot;: &quot;round&quot;,
  &quot;lineJoin&quot;: &quot;round&quot;,
  &quot;noClip&quot;: false,
  &quot;opacity&quot;: 0.9,
  &quot;smoothFactor&quot;: 1.0,
  &quot;stroke&quot;: true,
  &quot;weight&quot;: 2
}
                    )
                    .addTo(map_54cb488c2c244a8daafaa65ad6b9b54b);


            var marker_cluster_b7855381e5c049d9b9e9f1975d065c66 = L.markerClusterGroup({});
            map_54cb488c2c244a8daafaa65ad6b9b54b.addLayer(marker_cluster_b7855381e5c049d9b9e9f1975d065c66);


        var marker_a680310f341f4e47ad86118b2788f925 = L.marker(
            [43.26400000000003, -70.55996999999998],
            {
                icon: new L.Icon.Default()
                }
            ).addTo(marker_cluster_b7855381e5c049d9b9e9f1975d065c66);


            var popup_d7b388cdee9c4bea9afad1691d26fdd0 = L.popup({maxWidth: '300'

            });


                var html_798586dc7937475e92de398580908435 = $(`<div id=&quot;html_798586dc7937475e92de398580908435&quot; style=&quot;width: 100.0%; height: 100.0%;&quot;>[global]: Wells, ME</div>`)[0];
                popup_d7b388cdee9c4bea9afad1691d26fdd0.setContent(html_798586dc7937475e92de398580908435);


            marker_a680310f341f4e47ad86118b2788f925.bindPopup(popup_d7b388cdee9c4bea9afad1691d26fdd0)
            ;




        var marker_f6cbd564e3154c51b04de05e48e860bf = L.marker(
            [43.251140093105406, -70.54411764699331],
            {
                icon: new L.Icon.Default()
                }
            ).addTo(marker_cluster_b7855381e5c049d9b9e9f1975d065c66);


            var popup_299112f20a7e4c0693b3bfb09d715f8f = L.popup({maxWidth: '300'

            });


                var html_cbc8523aac754d5dbf9a407bf43cb8d5 = $(`<div id=&quot;html_cbc8523aac754d5dbf9a407bf43cb8d5&quot; style=&quot;width: 100.0%; height: 100.0%;&quot;>[SECOORA_NCSU_CNAPS]: Wells, ME</div>`)[0];
                popup_299112f20a7e4c0693b3bfb09d715f8f.setContent(html_cbc8523aac754d5dbf9a407bf43cb8d5);


            marker_f6cbd564e3154c51b04de05e48e860bf.bindPopup(popup_299112f20a7e4c0693b3bfb09d715f8f)
            ;




        var marker_fb5bd37496254ce9a556f2ba46158a6a = L.marker(
            [42.35387420654297, -71.05035400390625],
            {
                icon: new L.Icon.Default()
                }
            ).addTo(marker_cluster_b7855381e5c049d9b9e9f1975d065c66);


            var popup_21508c0fc9904786866432930217c7d2 = L.popup({maxWidth: '300'

            });


                var html_6d6bd8d38eb64ac6a101005e136b6f2b = $(`<div id=&quot;html_6d6bd8d38eb64ac6a101005e136b6f2b&quot; style=&quot;width: 100.0%; height: 100.0%;&quot;>[NECOFS_FVCOM_OCEAN_BOSTON_FORECAST]: Boston, MA</div>`)[0];
                popup_21508c0fc9904786866432930217c7d2.setContent(html_6d6bd8d38eb64ac6a101005e136b6f2b);


            marker_fb5bd37496254ce9a556f2ba46158a6a.bindPopup(popup_21508c0fc9904786866432930217c7d2)
            ;




        var marker_dce5aa625db3451b87b2b77f8cba9a7a = L.marker(
            [41.66425377919582, -69.9143933867417],
            {
                icon: new L.Icon.Default()
                }
            ).addTo(marker_cluster_b7855381e5c049d9b9e9f1975d065c66);


            var popup_61e9cc65a3924d4dac35d1c83ae2e789 = L.popup({maxWidth: '300'

            });


                var html_27731274d3934f498a45650002b2a94f = $(`<div id=&quot;html_27731274d3934f498a45650002b2a94f&quot; style=&quot;width: 100.0%; height: 100.0%;&quot;>[Time_v2_History_Best]: Chatham, Lydia Cove, MA</div>`)[0];
                popup_61e9cc65a3924d4dac35d1c83ae2e789.setContent(html_27731274d3934f498a45650002b2a94f);


            marker_dce5aa625db3451b87b2b77f8cba9a7a.bindPopup(popup_61e9cc65a3924d4dac35d1c83ae2e789)
            ;




        var marker_b2f6cde116f34a4695ed9c2b55bd04bc = L.marker(
            [41.67080000000001, -69.9199700000001],
            {
                icon: new L.Icon.Default()
                }
            ).addTo(marker_cluster_b7855381e5c049d9b9e9f1975d065c66);


            var popup_4d8e5d1328ec4e04a96ad87cdf5d8a5f = L.popup({maxWidth: '300'

            });


                var html_a1fb349f606c4a518f01607678c19b38 = $(`<div id=&quot;html_a1fb349f606c4a518f01607678c19b38&quot; style=&quot;width: 100.0%; height: 100.0%;&quot;>[global]: Chatham, Lydia Cove, MA</div>`)[0];
                popup_4d8e5d1328ec4e04a96ad87cdf5d8a5f.setContent(html_a1fb349f606c4a518f01607678c19b38);


            marker_b2f6cde116f34a4695ed9c2b55bd04bc.bindPopup(popup_4d8e5d1328ec4e04a96ad87cdf5d8a5f)
            ;




        var marker_dbec627d740f444a885e9615465c2fb7 = L.marker(
            [41.66425377919582, -69.9143933867417],
            {
                icon: new L.Icon.Default()
                }
            ).addTo(marker_cluster_b7855381e5c049d9b9e9f1975d065c66);


            var popup_234ccec39d474eeaab739c228289f49f = L.popup({maxWidth: '300'

            });


                var html_05be1fa73daa4d729c1a41669a60b0ba = $(`<div id=&quot;html_05be1fa73daa4d729c1a41669a60b0ba&quot; style=&quot;width: 100.0%; height: 100.0%;&quot;>[Time_v2_Averages_Best]: Chatham, Lydia Cove, MA</div>`)[0];
                popup_234ccec39d474eeaab739c228289f49f.setContent(html_05be1fa73daa4d729c1a41669a60b0ba);


            marker_dbec627d740f444a885e9615465c2fb7.bindPopup(popup_234ccec39d474eeaab739c228289f49f)
            ;




        var marker_e961686fcab3417fae8d936f82555ff8 = L.marker(
            [41.492152566660415, -70.64910206422738],
            {
                icon: new L.Icon.Default()
                }
            ).addTo(marker_cluster_b7855381e5c049d9b9e9f1975d065c66);


            var popup_131cc4bacb554680aeb8b66c91c3aec1 = L.popup({maxWidth: '300'

            });


                var html_924fed6292b04eaa8c407937fd074bc3 = $(`<div id=&quot;html_924fed6292b04eaa8c407937fd074bc3&quot; style=&quot;width: 100.0%; height: 100.0%;&quot;>[Time_v2_History_Best]: Woods Hole, MA</div>`)[0];
                popup_131cc4bacb554680aeb8b66c91c3aec1.setContent(html_924fed6292b04eaa8c407937fd074bc3);


            marker_e961686fcab3417fae8d936f82555ff8.bindPopup(popup_131cc4bacb554680aeb8b66c91c3aec1)
            ;




        var marker_4f45a420b4b34c108492a6d6d6d08e32 = L.marker(
            [41.5137596760154, -70.64140271482802],
            {
                icon: new L.Icon.Default()
                }
            ).addTo(marker_cluster_b7855381e5c049d9b9e9f1975d065c66);


            var popup_92156b626eb049939f13157ff3f09225 = L.popup({maxWidth: '300'

            });


                var html_5d34fa242a28492aaa38343179071686 = $(`<div id=&quot;html_5d34fa242a28492aaa38343179071686&quot; style=&quot;width: 100.0%; height: 100.0%;&quot;>[SECOORA_NCSU_CNAPS]: Woods Hole, MA</div>`)[0];
                popup_92156b626eb049939f13157ff3f09225.setContent(html_5d34fa242a28492aaa38343179071686);


            marker_4f45a420b4b34c108492a6d6d6d08e32.bindPopup(popup_92156b626eb049939f13157ff3f09225)
            ;




        var marker_a266572303af4dbabfb2d25580a9de24 = L.marker(
            [41.492152566660415, -70.64910206422738],
            {
                icon: new L.Icon.Default()
                }
            ).addTo(marker_cluster_b7855381e5c049d9b9e9f1975d065c66);


            var popup_82c8900dc17e4de99851a6344aa7956f = L.popup({maxWidth: '300'

            });


                var html_486c58f8b2dc4defaf33d942347f4a9c = $(`<div id=&quot;html_486c58f8b2dc4defaf33d942347f4a9c&quot; style=&quot;width: 100.0%; height: 100.0%;&quot;>[Time_v2_Averages_Best]: Woods Hole, MA</div>`)[0];
                popup_82c8900dc17e4de99851a6344aa7956f.setContent(html_486c58f8b2dc4defaf33d942347f4a9c);


            marker_a266572303af4dbabfb2d25580a9de24.bindPopup(popup_82c8900dc17e4de99851a6344aa7956f)
            ;




        var marker_6ee1646089ae40d492f2b17ae4657255 = L.marker(
            [43.6561, -70.2461],
            {
                icon: new L.Icon.Default()
                }
            ).addTo(map_54cb488c2c244a8daafaa65ad6b9b54b);



                var icon_d3098a8a4d1e43958bf987b84060d96c = L.AwesomeMarkers.icon({
                    icon: 'stats',
                    iconColor: 'white',
                    markerColor: 'green',
                    prefix: 'glyphicon',
                    extraClasses: 'fa-rotate-0'
                    });
                marker_6ee1646089ae40d492f2b17ae4657255.setIcon(icon_d3098a8a4d1e43958bf987b84060d96c);


            var popup_84bcb54ba8dc46168ee360997c8dcabc = L.popup({maxWidth: '2650'

            });


                var i_frame_39617a0242f34d36b372cee78eb2df19 = $(`<iframe src=&quot;data:text/html;charset=utf-8;base64,CiAgICAKCgoKPCFET0NUWVBFIGh0bWw+CjxodG1sIGxhbmc9ImVuIj4KICAKICA8aGVhZD4KICAgIAogICAgICA8bWV0YSBjaGFyc2V0PSJ1dGYtOCI+CiAgICAgIDx0aXRsZT44NDE4MTUwPC90aXRsZT4KICAgICAgCiAgICAgIAogICAgICAgIAogICAgICAgICAgCiAgICAgICAgPGxpbmsgcmVsPSJzdHlsZXNoZWV0IiBocmVmPSJodHRwczovL2Nkbi5weWRhdGEub3JnL2Jva2VoL3JlbGVhc2UvYm9rZWgtMS4wLjEubWluLmNzcyIgdHlwZT0idGV4dC9jc3MiIC8+CiAgICAgICAgCiAgICAgICAgCiAgICAgICAgICAKICAgICAgICA8c2NyaXB0IHR5cGU9InRleHQvamF2YXNjcmlwdCIgc3JjPSJodHRwczovL2Nkbi5weWRhdGEub3JnL2Jva2VoL3JlbGVhc2UvYm9rZWgtMS4wLjEubWluLmpzIj48L3NjcmlwdD4KICAgICAgICA8c2NyaXB0IHR5cGU9InRleHQvamF2YXNjcmlwdCI+CiAgICAgICAgICAgIEJva2VoLnNldF9sb2dfbGV2ZWwoImluZm8iKTsKICAgICAgICA8L3NjcmlwdD4KICAgICAgICAKICAgICAgCiAgICAgIAogICAgCiAgPC9oZWFkPgogIAogIAogIDxib2R5PgogICAgCiAgICAgIAogICAgICAgIAogICAgICAgICAgCiAgICAgICAgICAKICAgICAgICAgICAgCiAgICAgICAgICAgICAgPGRpdiBjbGFzcz0iYmstcm9vdCIgaWQ9Ijk1OTY1YjgwLTE3NjItNDY0YS04ODE3LTY0M2NlYTRlNmE0ZCI+PC9kaXY+CiAgICAgICAgICAgIAogICAgICAgICAgCiAgICAgICAgCiAgICAgIAogICAgICAKICAgICAgICA8c2NyaXB0IHR5cGU9ImFwcGxpY2F0aW9uL2pzb24iIGlkPSIxMjAwIj4KICAgICAgICAgIHsiZjVkOWI2Y2YtNWMwZi00MDE4LTk0ZDctNWE0N2RjMTE0MDJmIjp7InJvb3RzIjp7InJlZmVyZW5jZXMiOlt7ImF0dHJpYnV0ZXMiOnt9LCJpZCI6IjEwNDIiLCJ0eXBlIjoiRGF0ZXRpbWVUaWNrRm9ybWF0dGVyIn0seyJhdHRyaWJ1dGVzIjp7Im51bV9taW5vcl90aWNrcyI6NSwidGlja2VycyI6W3siaWQiOiIxMDQ1IiwidHlwZSI6IkFkYXB0aXZlVGlja2VyIn0seyJpZCI6IjEwNDYiLCJ0eXBlIjoiQWRhcHRpdmVUaWNrZXIifSx7ImlkIjoiMTA0NyIsInR5cGUiOiJBZGFwdGl2ZVRpY2tlciJ9LHsiaWQiOiIxMDQ4IiwidHlwZSI6IkRheXNUaWNrZXIifSx7ImlkIjoiMTA0OSIsInR5cGUiOiJEYXlzVGlja2VyIn0seyJpZCI6IjEwNTAiLCJ0eXBlIjoiRGF5c1RpY2tlciJ9LHsiaWQiOiIxMDUxIiwidHlwZSI6IkRheXNUaWNrZXIifSx7ImlkIjoiMTA1MiIsInR5cGUiOiJNb250aHNUaWNrZXIifSx7ImlkIjoiMTA1MyIsInR5cGUiOiJNb250aHNUaWNrZXIifSx7ImlkIjoiMTA1NCIsInR5cGUiOiJNb250aHNUaWNrZXIifSx7ImlkIjoiMTA1NSIsInR5cGUiOiJNb250aHNUaWNrZXIifSx7ImlkIjoiMTA1NiIsInR5cGUiOiJZZWFyc1RpY2tlciJ9XX0sImlkIjoiMTAxMyIsInR5cGUiOiJEYXRldGltZVRpY2tlciJ9LHsiYXR0cmlidXRlcyI6e30sImlkIjoiMTAxOCIsInR5cGUiOiJCYXNpY1RpY2tlciJ9LHsiYXR0cmlidXRlcyI6eyJhY3RpdmVfZHJhZyI6ImF1dG8iLCJhY3RpdmVfaW5zcGVjdCI6ImF1dG8iLCJhY3RpdmVfbXVsdGkiOm51bGwsImFjdGl2ZV9zY3JvbGwiOiJhdXRvIiwiYWN0aXZlX3RhcCI6ImF1dG8iLCJ0b29scyI6W3siaWQiOiIxMDIyIiwidHlwZSI6IlBhblRvb2wifSx7ImlkIjoiMTAyMyIsInR5cGUiOiJCb3hab29tVG9vbCJ9LHsiaWQiOiIxMDI0IiwidHlwZSI6IlJlc2V0VG9vbCJ9LHsiaWQiOiIxMDM2IiwidHlwZSI6IkhvdmVyVG9vbCJ9XX0sImlkIjoiMTAyNSIsInR5cGUiOiJUb29sYmFyIn0seyJhdHRyaWJ1dGVzIjp7ImRpbWVuc2lvbiI6MSwicGxvdCI6eyJpZCI6IjEwMDIiLCJzdWJ0eXBlIjoiRmlndXJlIiwidHlwZSI6IlBsb3QifSwidGlja2VyIjp7ImlkIjoiMTAxOCIsInR5cGUiOiJCYXNpY1RpY2tlciJ9fSwiaWQiOiIxMDIxIiwidHlwZSI6IkdyaWQifSx7ImF0dHJpYnV0ZXMiOnsic291cmNlIjp7ImlkIjoiMTAzMSIsInR5cGUiOiJDb2x1bW5EYXRhU291cmNlIn19LCJpZCI6IjEwMzUiLCJ0eXBlIjoiQ0RTVmlldyJ9LHsiYXR0cmlidXRlcyI6eyJib3R0b21fdW5pdHMiOiJzY3JlZW4iLCJmaWxsX2FscGhhIjp7InZhbHVlIjowLjV9LCJmaWxsX2NvbG9yIjp7InZhbHVlIjoibGlnaHRncmV5In0sImxlZnRfdW5pdHMiOiJzY3JlZW4iLCJsZXZlbCI6Im92ZXJsYXkiLCJsaW5lX2FscGhhIjp7InZhbHVlIjoxLjB9LCJsaW5lX2NvbG9yIjp7InZhbHVlIjoiYmxhY2sifSwibGluZV9kYXNoIjpbNCw0XSwibGluZV93aWR0aCI6eyJ2YWx1ZSI6Mn0sInBsb3QiOm51bGwsInJlbmRlcl9tb2RlIjoiY3NzIiwicmlnaHRfdW5pdHMiOiJzY3JlZW4iLCJ0b3BfdW5pdHMiOiJzY3JlZW4ifSwiaWQiOiIxMDI2IiwidHlwZSI6IkJveEFubm90YXRpb24ifSx7ImF0dHJpYnV0ZXMiOnsiZGF0YV9zb3VyY2UiOnsiaWQiOiIxMDMxIiwidHlwZSI6IkNvbHVtbkRhdGFTb3VyY2UifSwiZ2x5cGgiOnsiaWQiOiIxMDMyIiwidHlwZSI6IkxpbmUifSwiaG92ZXJfZ2x5cGgiOm51bGwsIm11dGVkX2dseXBoIjpudWxsLCJub25zZWxlY3Rpb25fZ2x5cGgiOnsiaWQiOiIxMDMzIiwidHlwZSI6IkxpbmUifSwic2VsZWN0aW9uX2dseXBoIjpudWxsLCJ2aWV3Ijp7ImlkIjoiMTAzNSIsInR5cGUiOiJDRFNWaWV3In19LCJpZCI6IjEwMzQiLCJ0eXBlIjoiR2x5cGhSZW5kZXJlciJ9LHsiYXR0cmlidXRlcyI6eyJtb250aHMiOlswLDQsOF19LCJpZCI6IjEwNTQiLCJ0eXBlIjoiTW9udGhzVGlja2VyIn0seyJhdHRyaWJ1dGVzIjp7fSwiaWQiOiIxMDIyIiwidHlwZSI6IlBhblRvb2wifSx7ImF0dHJpYnV0ZXMiOnsibGFiZWwiOnsidmFsdWUiOiJPYnNlcnZhdGlvbnMifSwicmVuZGVyZXJzIjpbeyJpZCI6IjEwMzQiLCJ0eXBlIjoiR2x5cGhSZW5kZXJlciJ9XX0sImlkIjoiMTAzOSIsInR5cGUiOiJMZWdlbmRJdGVtIn0seyJhdHRyaWJ1dGVzIjp7Im92ZXJsYXkiOnsiaWQiOiIxMDI2IiwidHlwZSI6IkJveEFubm90YXRpb24ifX0sImlkIjoiMTAyMyIsInR5cGUiOiJCb3hab29tVG9vbCJ9LHsiYXR0cmlidXRlcyI6eyJheGlzX2xhYmVsIjoiV2F0ZXIgSGVpZ2h0IChtKSIsImZvcm1hdHRlciI6eyJpZCI6IjEwNDQiLCJ0eXBlIjoiQmFzaWNUaWNrRm9ybWF0dGVyIn0sInBsb3QiOnsiaWQiOiIxMDAyIiwic3VidHlwZSI6IkZpZ3VyZSIsInR5cGUiOiJQbG90In0sInRpY2tlciI6eyJpZCI6IjEwMTgiLCJ0eXBlIjoiQmFzaWNUaWNrZXIifX0sImlkIjoiMTAxNyIsInR5cGUiOiJMaW5lYXJBeGlzIn0seyJhdHRyaWJ1dGVzIjp7ImNhbGxiYWNrIjpudWxsfSwiaWQiOiIxMDA0IiwidHlwZSI6IkRhdGFSYW5nZTFkIn0seyJhdHRyaWJ1dGVzIjp7ImNhbGxiYWNrIjpudWxsfSwiaWQiOiIxMDA2IiwidHlwZSI6IkRhdGFSYW5nZTFkIn0seyJhdHRyaWJ1dGVzIjp7ImNhbGxiYWNrIjpudWxsLCJkYXRhIjp7IngiOnsiX19uZGFycmF5X18iOiJBQUNBVnBzZGRrSUFBR2pGbmgxMlFnQUFVRFNpSFhaQ0FBQTRvNlVkZGtJQUFDQVNxUjEyUWdBQUNJR3NIWFpDQUFEdzc2OGRka0lBQU5oZXN4MTJRZ0FBd00yMkhYWkNBQUNvUExvZGRrSUFBSkNydlIxMlFnQUFlQnJCSFhaQ0FBQmdpY1FkZGtJQUFFajR4eDEyUWdBQU1HZkxIWFpDQUFBWTFzNGRka0lBQUFCRjBoMTJRZ0FBNkxQVkhYWkNBQURRSXRrZGRrSUFBTGlSM0IxMlFnQUFvQURnSFhaQ0FBQ0liK01kZGtJQUFIRGU1aDEyUWdBQVdFM3FIWFpDQUFCQXZPMGRka0lBQUNncjhSMTJRZ0FBRUpyMEhYWkNBQUQ0Q1BnZGRrSUFBT0IzK3gxMlFnQUF5T2IrSFhaQ0FBQ3dWUUllZGtJQUFKakVCUjUyUWdBQWdETUpIblpDQUFCb29nd2Vka0lBQUZBUkVCNTJRZ0FBT0lBVEhuWkNBQUFnN3hZZWRrSUFBQWhlR2g1MlFnQUE4TXdkSG5aQ0FBRFlPeUVlZGtJQUFNQ3FKQjUyUWdBQXFCa29IblpDQUFDUWlDc2Vka0lBQUhqM0xoNTJRZ0FBWUdZeUhuWkNBQUJJMVRVZWRrSUFBREJFT1I1MlFnQUFHTE04SG5aQ0FBQUFJa0FlZGtJQUFPaVFReDUyUWdBQTBQOUdIblpDQUFDNGJrb2Vka0lBQUtEZFRSNTJRZ0FBaUV4UkhuWkNBQUJ3dTFRZWRrSUFBRmdxV0I1MlFnQUFRSmxiSG5aQ0FBQW9DRjhlZGtJQUFCQjNZaDUyUWdBQStPVmxIblpDQUFEZ1ZHa2Vka0lBQU1qRGJCNTJRZ0FBc0RKd0huWkNBQUNZb1hNZWRrSUFBSUFRZHg1MlFnQUFhSDk2SG5aQ0FBQlE3bjBlZGtJQUFEaGRnUjUyUWdBQUlNeUVIblpDQUFBSU80Z2Vka0lBQVBDcGl4NTJRZ0FBMkJpUEhuWkNBQURBaDVJZWRrSUFBS2oybFI1MlFnQUFrR1daSG5aQ0FBQjQxSndlZGtJQUFHQkRvQjUyUWdBQVNMS2pIblpDQUFBd0lhY2Vka0lBQUJpUXFoNTJRZ0FBQVArdEhuWkNBQURvYmJFZWRrSUFBTkRjdEI1MlFnQUF1RXU0SG5aQ0FBQ2d1cnNlZGtJQUFJZ3B2eDUyUWdBQWNKakNIblpDQUFCWUI4WWVka0lBQUVCMnlSNTJRZ0FBS09YTUhuWkNBQUFRVk5BZWRrSUFBUGpDMHg1MlFnQUE0REhYSG5aQ0FBRElvTm9lZGtJQUFMQVAzaDUyUWdBQW1IN2hIblpDQUFDQTdlUWVka0lBQUdoYzZCNTJRZ0FBVU12ckhuWkNBQUE0T3U4ZWRrSUFBQ0NwOGg1MlFnQUFDQmoySG5aQ0FBRHdodmtlZGtJQUFOajEvQjUyUWdBQXdHUUFIM1pDQUFDbzB3TWZka0lBQUpCQ0J4OTJRZ0FBZUxFS0gzWkNBQUJnSUE0ZmRrSUFBRWlQRVI5MlFnQUFNUDRVSDNaQ0FBQVliUmdmZGtJQUFBRGNHeDkyUWdBQTZFb2ZIM1pDQUFEUXVTSWZka0lBQUxnb0poOTJRZ0FBb0pjcEgzWkNBQUNJQmkwZmRrSUFBSEIxTUI5MlFnQUFXT1F6SDNaQ0FBQkFVemNmZGtJPSIsImR0eXBlIjoiZmxvYXQ2NCIsInNoYXBlIjpbMTIxXX0sInkiOnsiX19uZGFycmF5X18iOiJTT0Y2Rks1SEFrRGZUNDJYYmhJR1FPU2xtOFFnc0FkQTAwMWlFRmc1QmtETG9VVzI4LzBCUUFSV0RpMnluZmMvN253L05WNjY1VCtCbFVPTGJPZTdQd2lzSEZwa082Ky9JOXY1Zm1xOHhEOWpFRmc1dE1qbVA3dEpEQUlyaC9nL1Nnd0NLNGNXQTBDMHlIYStueG9JUUJGWU9iVElkZ3BBK241cXZIU1RDVUFBQUFBQUFBQUdRQWlzSEZwa08vOC9GSzVINFhvVThEK3luZStueGt2SFA1SHRmRDgxWHRLL3pjek16TXpNMUw4SzE2TndQUXFuUDIzbis2bngwdWsvaGV0UnVCNkYrejhVcmtmaGVoUUVRRWpoZWhTdVJ3aEF4U0N3Y21pUkNVRDRVK09sbThRSFFCMWFaRHZmVHdOQU85OVBqWmR1K0QrYW1abVptWm5sUHlHd2NtaVI3YncvV21RNzMwK050NythbVptWm1abkpQMkRsMENMYitlby9jVDBLMTZOdyt6OEdnWlZEaTJ3RlFLUndQUXJYb3dwQTE2TndQUXJYREVBdHNwM3ZwOFlMUU5OTlloQllPUWRBbk1RZ3NISm9BRURKZHI2ZkdpL3hQLzNVZU9rbU1jZy9QelZldWtrTTByOXQ1L3VwOGRMTnYvM1VlT2ttTWNnL3RNaDJ2cDhhN3o4M2lVRmc1ZEQrUDdiei9kUjQ2UVZBdjU4YUw5MGtDa0NYYmhLRHdNb0tRQy9kSkFhQmxRaEFabVptWm1abUEwQ0JsVU9MYk9mM1AveXA4ZEpOWXVRL1ZPT2xtOFFnc0QrTWJPZjdxZkdpdjNFOUN0ZWpjTlUvalpkdUVvUEE4RCt1UitGNkZLNEFRQ1VHZ1pWRGl3aEEyczczVStPbERVQTFYcnBKREFJUVFCK0Y2MUc0SGc5QXhrczNpVUZnQ2tDSlFXRGwwQ0lEUUxiei9kUjQ2ZlkvQWl1SEZ0bk80ejhoc0hKb2tlM01QL1Q5MUhqcEp0RS9rdTE4UHpWZTZqOWFaRHZmVDQzNVA3VElkcjZmR2dSQVptWm1abVptQ2tEUDkxUGpwWnNOUUJLRHdNcWhSUTVBY1QwSzE2TndDMEJVNDZXYnhDQUZRTytueGtzM2lmcy92NThhTDkwazdqL2wwQ0xiK1g3YVA3K2ZHaS9kSk5ZL0RpMnluZStuNWo5emFKSHRmRC8zUHdyWG8zQTlDZ05BZDc2ZkdpL2RDVUJVNDZXYnhDQU9RUExTVFdJUVdBOUFZT1hRSXR2NURFRG8rNm54MGswSVFEMEsxNk53UFFGQS9LbngwazFpOUQrb3hrczNpVUhnUC9Zb1hJL0M5ZEEvNlB1cDhkSk4yajhiTDkwa0JvSHRQNU1ZQkZZT0xmdy93TXFoUmJiekJFQk9ZaEJZT2JRS1FCRllPYlRJZGcxQWZUODFYcnBKRFVERDlTaGNqOElKUUJTdVIrRjZGQVJBRjluTzkxUGorVDhSV0RtMHlIYnFQMWc1dE1oMnZ0Yy9XRG0weUhhKzF6OTlQelZldWtub1B4c3YzU1FHZ2ZjL3R2UDkxSGpwQWtEcEpqRUlyQndKUUZZT0xiS2Q3d3hBNzZmR1N6ZUpEVUJwa2UxOFB6VUxRQXJYbzNBOUNnWkFQUXJYbzNBOS9qOW1abVptWm1id1ArRjZGSzVINGRvLytGUGpwWnZFMEQ4PSIsImR0eXBlIjoiZmxvYXQ2NCIsInNoYXBlIjpbMTIxXX19LCJzZWxlY3RlZCI6eyJpZCI6IjEwNTgiLCJ0eXBlIjoiU2VsZWN0aW9uIn0sInNlbGVjdGlvbl9wb2xpY3kiOnsiaWQiOiIxMDU5IiwidHlwZSI6IlVuaW9uUmVuZGVyZXJzIn19LCJpZCI6IjEwMzEiLCJ0eXBlIjoiQ29sdW1uRGF0YVNvdXJjZSJ9LHsiYXR0cmlidXRlcyI6eyJheGlzX2xhYmVsIjoiRGF0ZS90aW1lIiwiZm9ybWF0dGVyIjp7ImlkIjoiMTA0MiIsInR5cGUiOiJEYXRldGltZVRpY2tGb3JtYXR0ZXIifSwicGxvdCI6eyJpZCI6IjEwMDIiLCJzdWJ0eXBlIjoiRmlndXJlIiwidHlwZSI6IlBsb3QifSwidGlja2VyIjp7ImlkIjoiMTAxMyIsInR5cGUiOiJEYXRldGltZVRpY2tlciJ9fSwiaWQiOiIxMDEyIiwidHlwZSI6IkRhdGV0aW1lQXhpcyJ9LHsiYXR0cmlidXRlcyI6e30sImlkIjoiMTA1OCIsInR5cGUiOiJTZWxlY3Rpb24ifSx7ImF0dHJpYnV0ZXMiOnt9LCJpZCI6IjEwNTkiLCJ0eXBlIjoiVW5pb25SZW5kZXJlcnMifSx7ImF0dHJpYnV0ZXMiOnsiZGF5cyI6WzEsOCwxNSwyMl19LCJpZCI6IjEwNTAiLCJ0eXBlIjoiRGF5c1RpY2tlciJ9LHsiYXR0cmlidXRlcyI6eyJtb250aHMiOlswLDIsNCw2LDgsMTBdfSwiaWQiOiIxMDUzIiwidHlwZSI6Ik1vbnRoc1RpY2tlciJ9LHsiYXR0cmlidXRlcyI6eyJiYXNlIjoyNCwibWFudGlzc2FzIjpbMSwyLDQsNiw4LDEyXSwibWF4X2ludGVydmFsIjo0MzIwMDAwMC4wLCJtaW5faW50ZXJ2YWwiOjM2MDAwMDAuMCwibnVtX21pbm9yX3RpY2tzIjowfSwiaWQiOiIxMDQ3IiwidHlwZSI6IkFkYXB0aXZlVGlja2VyIn0seyJhdHRyaWJ1dGVzIjp7InBsb3QiOm51bGwsInRleHQiOiI4NDE4MTUwIn0sImlkIjoiMTAwMSIsInR5cGUiOiJUaXRsZSJ9LHsiYXR0cmlidXRlcyI6eyJsaW5lX2FscGhhIjowLjEsImxpbmVfY2FwIjoicm91bmQiLCJsaW5lX2NvbG9yIjoiIzFmNzdiNCIsImxpbmVfam9pbiI6InJvdW5kIiwibGluZV93aWR0aCI6NSwieCI6eyJmaWVsZCI6IngifSwieSI6eyJmaWVsZCI6InkifX0sImlkIjoiMTAzMyIsInR5cGUiOiJMaW5lIn0seyJhdHRyaWJ1dGVzIjp7Im1vbnRocyI6WzAsNl19LCJpZCI6IjEwNTUiLCJ0eXBlIjoiTW9udGhzVGlja2VyIn0seyJhdHRyaWJ1dGVzIjp7InBsb3QiOnsiaWQiOiIxMDAyIiwic3VidHlwZSI6IkZpZ3VyZSIsInR5cGUiOiJQbG90In0sInRpY2tlciI6eyJpZCI6IjEwMTMiLCJ0eXBlIjoiRGF0ZXRpbWVUaWNrZXIifX0sImlkIjoiMTAxNiIsInR5cGUiOiJHcmlkIn0seyJhdHRyaWJ1dGVzIjp7ImRheXMiOlsxLDQsNywxMCwxMywxNiwxOSwyMiwyNSwyOF19LCJpZCI6IjEwNDkiLCJ0eXBlIjoiRGF5c1RpY2tlciJ9LHsiYXR0cmlidXRlcyI6e30sImlkIjoiMTA0NCIsInR5cGUiOiJCYXNpY1RpY2tGb3JtYXR0ZXIifSx7ImF0dHJpYnV0ZXMiOnsiZGF5cyI6WzEsMiwzLDQsNSw2LDcsOCw5LDEwLDExLDEyLDEzLDE0LDE1LDE2LDE3LDE4LDE5LDIwLDIxLDIyLDIzLDI0LDI1LDI2LDI3LDI4LDI5LDMwLDMxXX0sImlkIjoiMTA0OCIsInR5cGUiOiJEYXlzVGlja2VyIn0seyJhdHRyaWJ1dGVzIjp7ImJlbG93IjpbeyJpZCI6IjEwMTIiLCJ0eXBlIjoiRGF0ZXRpbWVBeGlzIn1dLCJsZWZ0IjpbeyJpZCI6IjEwMTciLCJ0eXBlIjoiTGluZWFyQXhpcyJ9XSwicGxvdF9oZWlnaHQiOjI1MCwicGxvdF93aWR0aCI6NzUwLCJyZW5kZXJlcnMiOlt7ImlkIjoiMTAxMiIsInR5cGUiOiJEYXRldGltZUF4aXMifSx7ImlkIjoiMTAxNiIsInR5cGUiOiJHcmlkIn0seyJpZCI6IjEwMTciLCJ0eXBlIjoiTGluZWFyQXhpcyJ9LHsiaWQiOiIxMDIxIiwidHlwZSI6IkdyaWQifSx7ImlkIjoiMTAyNiIsInR5cGUiOiJCb3hBbm5vdGF0aW9uIn0seyJpZCI6IjEwMzQiLCJ0eXBlIjoiR2x5cGhSZW5kZXJlciJ9LHsiaWQiOiIxMDM4IiwidHlwZSI6IkxlZ2VuZCJ9XSwicmlnaHQiOlt7ImlkIjoiMTAzOCIsInR5cGUiOiJMZWdlbmQifV0sInRpdGxlIjp7ImlkIjoiMTAwMSIsInR5cGUiOiJUaXRsZSJ9LCJ0b29sYmFyIjp7ImlkIjoiMTAyNSIsInR5cGUiOiJUb29sYmFyIn0sInRvb2xiYXJfbG9jYXRpb24iOiJhYm92ZSIsInhfcmFuZ2UiOnsiaWQiOiIxMDA0IiwidHlwZSI6IkRhdGFSYW5nZTFkIn0sInhfc2NhbGUiOnsiaWQiOiIxMDA4IiwidHlwZSI6IkxpbmVhclNjYWxlIn0sInlfcmFuZ2UiOnsiaWQiOiIxMDA2IiwidHlwZSI6IkRhdGFSYW5nZTFkIn0sInlfc2NhbGUiOnsiaWQiOiIxMDEwIiwidHlwZSI6IkxpbmVhclNjYWxlIn19LCJpZCI6IjEwMDIiLCJzdWJ0eXBlIjoiRmlndXJlIiwidHlwZSI6IlBsb3QifSx7ImF0dHJpYnV0ZXMiOnt9LCJpZCI6IjEwMjQiLCJ0eXBlIjoiUmVzZXRUb29sIn0seyJhdHRyaWJ1dGVzIjp7fSwiaWQiOiIxMDEwIiwidHlwZSI6IkxpbmVhclNjYWxlIn0seyJhdHRyaWJ1dGVzIjp7ImRheXMiOlsxLDE1XX0sImlkIjoiMTA1MSIsInR5cGUiOiJEYXlzVGlja2VyIn0seyJhdHRyaWJ1dGVzIjp7fSwiaWQiOiIxMDA4IiwidHlwZSI6IkxpbmVhclNjYWxlIn0seyJhdHRyaWJ1dGVzIjp7ImJhc2UiOjYwLCJtYW50aXNzYXMiOlsxLDIsNSwxMCwxNSwyMCwzMF0sIm1heF9pbnRlcnZhbCI6MTgwMDAwMC4wLCJtaW5faW50ZXJ2YWwiOjEwMDAuMCwibnVtX21pbm9yX3RpY2tzIjowfSwiaWQiOiIxMDQ2IiwidHlwZSI6IkFkYXB0aXZlVGlja2VyIn0seyJhdHRyaWJ1dGVzIjp7ImNhbGxiYWNrIjpudWxsLCJyZW5kZXJlcnMiOlt7ImlkIjoiMTAzNCIsInR5cGUiOiJHbHlwaFJlbmRlcmVyIn1dLCJ0b29sdGlwcyI6W1siTmFtZSIsIk9ic2VydmF0aW9ucyJdLFsiQmlhcyIsIk5BIl0sWyJTa2lsbCIsIk5BIl1dfSwiaWQiOiIxMDM2IiwidHlwZSI6IkhvdmVyVG9vbCJ9LHsiYXR0cmlidXRlcyI6eyJjbGlja19wb2xpY3kiOiJtdXRlIiwiaXRlbXMiOlt7ImlkIjoiMTAzOSIsInR5cGUiOiJMZWdlbmRJdGVtIn1dLCJsb2NhdGlvbiI6WzAsNjBdLCJwbG90Ijp7ImlkIjoiMTAwMiIsInN1YnR5cGUiOiJGaWd1cmUiLCJ0eXBlIjoiUGxvdCJ9fSwiaWQiOiIxMDM4IiwidHlwZSI6IkxlZ2VuZCJ9LHsiYXR0cmlidXRlcyI6eyJtYW50aXNzYXMiOlsxLDIsNV0sIm1heF9pbnRlcnZhbCI6NTAwLjAsIm51bV9taW5vcl90aWNrcyI6MH0sImlkIjoiMTA0NSIsInR5cGUiOiJBZGFwdGl2ZVRpY2tlciJ9LHsiYXR0cmlidXRlcyI6e30sImlkIjoiMTA1NiIsInR5cGUiOiJZZWFyc1RpY2tlciJ9LHsiYXR0cmlidXRlcyI6eyJtb250aHMiOlswLDEsMiwzLDQsNSw2LDcsOCw5LDEwLDExXX0sImlkIjoiMTA1MiIsInR5cGUiOiJNb250aHNUaWNrZXIifSx7ImF0dHJpYnV0ZXMiOnsibGluZV9jYXAiOiJyb3VuZCIsImxpbmVfY29sb3IiOiJjcmltc29uIiwibGluZV9qb2luIjoicm91bmQiLCJsaW5lX3dpZHRoIjo1LCJ4Ijp7ImZpZWxkIjoieCJ9LCJ5Ijp7ImZpZWxkIjoieSJ9fSwiaWQiOiIxMDMyIiwidHlwZSI6IkxpbmUifV0sInJvb3RfaWRzIjpbIjEwMDIiXX0sInRpdGxlIjoiQm9rZWggQXBwbGljYXRpb24iLCJ2ZXJzaW9uIjoiMS4wLjEifX0KICAgICAgICA8L3NjcmlwdD4KICAgICAgICA8c2NyaXB0IHR5cGU9InRleHQvamF2YXNjcmlwdCI+CiAgICAgICAgICAoZnVuY3Rpb24oKSB7CiAgICAgICAgICAgIHZhciBmbiA9IGZ1bmN0aW9uKCkgewogICAgICAgICAgICAgIEJva2VoLnNhZmVseShmdW5jdGlvbigpIHsKICAgICAgICAgICAgICAgIChmdW5jdGlvbihyb290KSB7CiAgICAgICAgICAgICAgICAgIGZ1bmN0aW9uIGVtYmVkX2RvY3VtZW50KHJvb3QpIHsKICAgICAgICAgICAgICAgICAgICAKICAgICAgICAgICAgICAgICAgdmFyIGRvY3NfanNvbiA9IGRvY3VtZW50LmdldEVsZW1lbnRCeUlkKCcxMjAwJykudGV4dENvbnRlbnQ7CiAgICAgICAgICAgICAgICAgIHZhciByZW5kZXJfaXRlbXMgPSBbeyJkb2NpZCI6ImY1ZDliNmNmLTVjMGYtNDAxOC05NGQ3LTVhNDdkYzExNDAyZiIsInJvb3RzIjp7IjEwMDIiOiI5NTk2NWI4MC0xNzYyLTQ2NGEtODgxNy02NDNjZWE0ZTZhNGQifX1dOwogICAgICAgICAgICAgICAgICByb290LkJva2VoLmVtYmVkLmVtYmVkX2l0ZW1zKGRvY3NfanNvbiwgcmVuZGVyX2l0ZW1zKTsKICAgICAgICAgICAgICAgIAogICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICAgIGlmIChyb290LkJva2VoICE9PSB1bmRlZmluZWQpIHsKICAgICAgICAgICAgICAgICAgICBlbWJlZF9kb2N1bWVudChyb290KTsKICAgICAgICAgICAgICAgICAgfSBlbHNlIHsKICAgICAgICAgICAgICAgICAgICB2YXIgYXR0ZW1wdHMgPSAwOwogICAgICAgICAgICAgICAgICAgIHZhciB0aW1lciA9IHNldEludGVydmFsKGZ1bmN0aW9uKHJvb3QpIHsKICAgICAgICAgICAgICAgICAgICAgIGlmIChyb290LkJva2VoICE9PSB1bmRlZmluZWQpIHsKICAgICAgICAgICAgICAgICAgICAgICAgZW1iZWRfZG9jdW1lbnQocm9vdCk7CiAgICAgICAgICAgICAgICAgICAgICAgIGNsZWFySW50ZXJ2YWwodGltZXIpOwogICAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgICAgICAgYXR0ZW1wdHMrKzsKICAgICAgICAgICAgICAgICAgICAgIGlmIChhdHRlbXB0cyA+IDEwMCkgewogICAgICAgICAgICAgICAgICAgICAgICBjb25zb2xlLmxvZygiQm9rZWg6IEVSUk9SOiBVbmFibGUgdG8gcnVuIEJva2VoSlMgY29kZSBiZWNhdXNlIEJva2VoSlMgbGlicmFyeSBpcyBtaXNzaW5nIik7CiAgICAgICAgICAgICAgICAgICAgICAgIGNsZWFySW50ZXJ2YWwodGltZXIpOwogICAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgICAgIH0sIDEwLCByb290KQogICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICB9KSh3aW5kb3cpOwogICAgICAgICAgICAgIH0pOwogICAgICAgICAgICB9OwogICAgICAgICAgICBpZiAoZG9jdW1lbnQucmVhZHlTdGF0ZSAhPSAibG9hZGluZyIpIGZuKCk7CiAgICAgICAgICAgIGVsc2UgZG9jdW1lbnQuYWRkRXZlbnRMaXN0ZW5lcigiRE9NQ29udGVudExvYWRlZCIsIGZuKTsKICAgICAgICAgIH0pKCk7CiAgICAgICAgPC9zY3JpcHQ+CiAgICAKICA8L2JvZHk+CiAgCjwvaHRtbD4=&quot; width=&quot;790&quot; style=&quot;border:none !important;&quot; height=&quot;330&quot;></iframe>`)[0];
                popup_84bcb54ba8dc46168ee360997c8dcabc.setContent(i_frame_39617a0242f34d36b372cee78eb2df19);


            marker_6ee1646089ae40d492f2b17ae4657255.bindPopup(popup_84bcb54ba8dc46168ee360997c8dcabc)
            ;




        var marker_b3a94bbc34e043a4b8f3b7b86c957f22 = L.marker(
            [43.32, -70.5633],
            {
                icon: new L.Icon.Default()
                }
            ).addTo(map_54cb488c2c244a8daafaa65ad6b9b54b);



                var icon_c96b3294722a43c8ad712d1c1ab02ef0 = L.AwesomeMarkers.icon({
                    icon: 'stats',
                    iconColor: 'white',
                    markerColor: 'green',
                    prefix: 'glyphicon',
                    extraClasses: 'fa-rotate-0'
                    });
                marker_b3a94bbc34e043a4b8f3b7b86c957f22.setIcon(icon_c96b3294722a43c8ad712d1c1ab02ef0);


            var popup_44cd7c6a42e242ec959bb3829b3763c2 = L.popup({maxWidth: '2650'

            });


                var i_frame_9f68bbcbd3ec485da7caecc755187418 = $(`<iframe src=&quot;data:text/html;charset=utf-8;base64,CiAgICAKCgoKPCFET0NUWVBFIGh0bWw+CjxodG1sIGxhbmc9ImVuIj4KICAKICA8aGVhZD4KICAgIAogICAgICA8bWV0YSBjaGFyc2V0PSJ1dGYtOCI+CiAgICAgIDx0aXRsZT44NDE5MzE3PC90aXRsZT4KICAgICAgCiAgICAgIAogICAgICAgIAogICAgICAgICAgCiAgICAgICAgPGxpbmsgcmVsPSJzdHlsZXNoZWV0IiBocmVmPSJodHRwczovL2Nkbi5weWRhdGEub3JnL2Jva2VoL3JlbGVhc2UvYm9rZWgtMS4wLjEubWluLmNzcyIgdHlwZT0idGV4dC9jc3MiIC8+CiAgICAgICAgCiAgICAgICAgCiAgICAgICAgICAKICAgICAgICA8c2NyaXB0IHR5cGU9InRleHQvamF2YXNjcmlwdCIgc3JjPSJodHRwczovL2Nkbi5weWRhdGEub3JnL2Jva2VoL3JlbGVhc2UvYm9rZWgtMS4wLjEubWluLmpzIj48L3NjcmlwdD4KICAgICAgICA8c2NyaXB0IHR5cGU9InRleHQvamF2YXNjcmlwdCI+CiAgICAgICAgICAgIEJva2VoLnNldF9sb2dfbGV2ZWwoImluZm8iKTsKICAgICAgICA8L3NjcmlwdD4KICAgICAgICAKICAgICAgCiAgICAgIAogICAgCiAgPC9oZWFkPgogIAogIAogIDxib2R5PgogICAgCiAgICAgIAogICAgICAgIAogICAgICAgICAgCiAgICAgICAgICAKICAgICAgICAgICAgCiAgICAgICAgICAgICAgPGRpdiBjbGFzcz0iYmstcm9vdCIgaWQ9IjBlMGY2YjdiLWJlNGYtNGZiZi05YTQ1LTA0YmE3NWU2MGJjZSI+PC9kaXY+CiAgICAgICAgICAgIAogICAgICAgICAgCiAgICAgICAgCiAgICAgIAogICAgICAKICAgICAgICA8c2NyaXB0IHR5cGU9ImFwcGxpY2F0aW9uL2pzb24iIGlkPSIxNDQ4Ij4KICAgICAgICAgIHsiMTExZWNjNTgtOTkzNy00NjVkLWIxNWUtYzYzNzc3OThhMTg0Ijp7InJvb3RzIjp7InJlZmVyZW5jZXMiOlt7ImF0dHJpYnV0ZXMiOnsiYXhpc19sYWJlbCI6IkRhdGUvdGltZSIsImZvcm1hdHRlciI6eyJpZCI6IjEyNTgiLCJ0eXBlIjoiRGF0ZXRpbWVUaWNrRm9ybWF0dGVyIn0sInBsb3QiOnsiaWQiOiIxMjAyIiwic3VidHlwZSI6IkZpZ3VyZSIsInR5cGUiOiJQbG90In0sInRpY2tlciI6eyJpZCI6IjEyMTMiLCJ0eXBlIjoiRGF0ZXRpbWVUaWNrZXIifX0sImlkIjoiMTIxMiIsInR5cGUiOiJEYXRldGltZUF4aXMifSx7ImF0dHJpYnV0ZXMiOnsiY2FsbGJhY2siOm51bGwsInJlbmRlcmVycyI6W3siaWQiOiIxMjQ4IiwidHlwZSI6IkdseXBoUmVuZGVyZXIifV0sInRvb2x0aXBzIjpbWyJOYW1lIiwiZ2xvYmFsIl0sWyJCaWFzIiwiLTIuMjEiXSxbIlNraWxsIiwiMC44OSJdXX0sImlkIjoiMTI1MCIsInR5cGUiOiJIb3ZlclRvb2wifSx7ImF0dHJpYnV0ZXMiOnsiY2FsbGJhY2siOm51bGwsInJlbmRlcmVycyI6W3siaWQiOiIxMjM0IiwidHlwZSI6IkdseXBoUmVuZGVyZXIifV0sInRvb2x0aXBzIjpbWyJOYW1lIiwiT2JzZXJ2YXRpb25zIl0sWyJCaWFzIiwiTkEiXSxbIlNraWxsIiwiTkEiXV19LCJpZCI6IjEyMzYiLCJ0eXBlIjoiSG92ZXJUb29sIn0seyJhdHRyaWJ1dGVzIjp7Im1vbnRocyI6WzAsNl19LCJpZCI6IjEyNzEiLCJ0eXBlIjoiTW9udGhzVGlja2VyIn0seyJhdHRyaWJ1dGVzIjp7ImxpbmVfYWxwaGEiOjAuMSwibGluZV9jYXAiOiJyb3VuZCIsImxpbmVfY29sb3IiOiIjMWY3N2I0IiwibGluZV9qb2luIjoicm91bmQiLCJsaW5lX3dpZHRoIjo1LCJ4Ijp7ImZpZWxkIjoieCJ9LCJ5Ijp7ImZpZWxkIjoieSJ9fSwiaWQiOiIxMjMzIiwidHlwZSI6IkxpbmUifSx7ImF0dHJpYnV0ZXMiOnt9LCJpZCI6IjEyMjIiLCJ0eXBlIjoiUGFuVG9vbCJ9LHsiYXR0cmlidXRlcyI6e30sImlkIjoiMTI3NyIsInR5cGUiOiJVbmlvblJlbmRlcmVycyJ9LHsiYXR0cmlidXRlcyI6eyJjYWxsYmFjayI6bnVsbH0sImlkIjoiMTIwNiIsInR5cGUiOiJEYXRhUmFuZ2UxZCJ9LHsiYXR0cmlidXRlcyI6e30sImlkIjoiMTI3OCIsInR5cGUiOiJTZWxlY3Rpb24ifSx7ImF0dHJpYnV0ZXMiOnsiY2FsbGJhY2siOm51bGx9LCJpZCI6IjEyMDQiLCJ0eXBlIjoiRGF0YVJhbmdlMWQifSx7ImF0dHJpYnV0ZXMiOnt9LCJpZCI6IjEyMDgiLCJ0eXBlIjoiTGluZWFyU2NhbGUifSx7ImF0dHJpYnV0ZXMiOnsiZGF0YV9zb3VyY2UiOnsiaWQiOiIxMjM4IiwidHlwZSI6IkNvbHVtbkRhdGFTb3VyY2UifSwiZ2x5cGgiOnsiaWQiOiIxMjM5IiwidHlwZSI6IkxpbmUifSwiaG92ZXJfZ2x5cGgiOm51bGwsIm11dGVkX2dseXBoIjpudWxsLCJub25zZWxlY3Rpb25fZ2x5cGgiOnsiaWQiOiIxMjQwIiwidHlwZSI6IkxpbmUifSwic2VsZWN0aW9uX2dseXBoIjpudWxsLCJ2aWV3Ijp7ImlkIjoiMTI0MiIsInR5cGUiOiJDRFNWaWV3In19LCJpZCI6IjEyNDEiLCJ0eXBlIjoiR2x5cGhSZW5kZXJlciJ9LHsiYXR0cmlidXRlcyI6eyJsYWJlbCI6eyJ2YWx1ZSI6Ik9ic2VydmF0aW9ucyJ9LCJyZW5kZXJlcnMiOlt7ImlkIjoiMTIzNCIsInR5cGUiOiJHbHlwaFJlbmRlcmVyIn1dfSwiaWQiOiIxMjUzIiwidHlwZSI6IkxlZ2VuZEl0ZW0ifSx7ImF0dHJpYnV0ZXMiOnsic291cmNlIjp7ImlkIjoiMTIzMSIsInR5cGUiOiJDb2x1bW5EYXRhU291cmNlIn19LCJpZCI6IjEyMzUiLCJ0eXBlIjoiQ0RTVmlldyJ9LHsiYXR0cmlidXRlcyI6e30sImlkIjoiMTIxOCIsInR5cGUiOiJCYXNpY1RpY2tlciJ9LHsiYXR0cmlidXRlcyI6eyJkYXlzIjpbMSw4LDE1LDIyXX0sImlkIjoiMTI2NiIsInR5cGUiOiJEYXlzVGlja2VyIn0seyJhdHRyaWJ1dGVzIjp7ImJlbG93IjpbeyJpZCI6IjEyMTIiLCJ0eXBlIjoiRGF0ZXRpbWVBeGlzIn1dLCJsZWZ0IjpbeyJpZCI6IjEyMTciLCJ0eXBlIjoiTGluZWFyQXhpcyJ9XSwicGxvdF9oZWlnaHQiOjI1MCwicGxvdF93aWR0aCI6NzUwLCJyZW5kZXJlcnMiOlt7ImlkIjoiMTIxMiIsInR5cGUiOiJEYXRldGltZUF4aXMifSx7ImlkIjoiMTIxNiIsInR5cGUiOiJHcmlkIn0seyJpZCI6IjEyMTciLCJ0eXBlIjoiTGluZWFyQXhpcyJ9LHsiaWQiOiIxMjIxIiwidHlwZSI6IkdyaWQifSx7ImlkIjoiMTIyNiIsInR5cGUiOiJCb3hBbm5vdGF0aW9uIn0seyJpZCI6IjEyMzQiLCJ0eXBlIjoiR2x5cGhSZW5kZXJlciJ9LHsiaWQiOiIxMjQxIiwidHlwZSI6IkdseXBoUmVuZGVyZXIifSx7ImlkIjoiMTI0OCIsInR5cGUiOiJHbHlwaFJlbmRlcmVyIn0seyJpZCI6IjEyNTIiLCJ0eXBlIjoiTGVnZW5kIn1dLCJyaWdodCI6W3siaWQiOiIxMjUyIiwidHlwZSI6IkxlZ2VuZCJ9XSwidGl0bGUiOnsiaWQiOiIxMjAxIiwidHlwZSI6IlRpdGxlIn0sInRvb2xiYXIiOnsiaWQiOiIxMjI1IiwidHlwZSI6IlRvb2xiYXIifSwidG9vbGJhcl9sb2NhdGlvbiI6ImFib3ZlIiwieF9yYW5nZSI6eyJpZCI6IjEyMDQiLCJ0eXBlIjoiRGF0YVJhbmdlMWQifSwieF9zY2FsZSI6eyJpZCI6IjEyMDgiLCJ0eXBlIjoiTGluZWFyU2NhbGUifSwieV9yYW5nZSI6eyJpZCI6IjEyMDYiLCJ0eXBlIjoiRGF0YVJhbmdlMWQifSwieV9zY2FsZSI6eyJpZCI6IjEyMTAiLCJ0eXBlIjoiTGluZWFyU2NhbGUifX0sImlkIjoiMTIwMiIsInN1YnR5cGUiOiJGaWd1cmUiLCJ0eXBlIjoiUGxvdCJ9LHsiYXR0cmlidXRlcyI6e30sImlkIjoiMTI3MiIsInR5cGUiOiJZZWFyc1RpY2tlciJ9LHsiYXR0cmlidXRlcyI6eyJsaW5lX2FscGhhIjowLjY1LCJsaW5lX2NhcCI6InJvdW5kIiwibGluZV9jb2xvciI6IiMxZjc3YjQiLCJsaW5lX2pvaW4iOiJyb3VuZCIsImxpbmVfd2lkdGgiOjUsIngiOnsiZmllbGQiOiJ4In0sInkiOnsiZmllbGQiOiJ5In19LCJpZCI6IjEyMzkiLCJ0eXBlIjoiTGluZSJ9LHsiYXR0cmlidXRlcyI6eyJtYW50aXNzYXMiOlsxLDIsNV0sIm1heF9pbnRlcnZhbCI6NTAwLjAsIm51bV9taW5vcl90aWNrcyI6MH0sImlkIjoiMTI2MSIsInR5cGUiOiJBZGFwdGl2ZVRpY2tlciJ9LHsiYXR0cmlidXRlcyI6eyJiYXNlIjoyNCwibWFudGlzc2FzIjpbMSwyLDQsNiw4LDEyXSwibWF4X2ludGVydmFsIjo0MzIwMDAwMC4wLCJtaW5faW50ZXJ2YWwiOjM2MDAwMDAuMCwibnVtX21pbm9yX3RpY2tzIjowfSwiaWQiOiIxMjYzIiwidHlwZSI6IkFkYXB0aXZlVGlja2VyIn0seyJhdHRyaWJ1dGVzIjp7fSwiaWQiOiIxMjU4IiwidHlwZSI6IkRhdGV0aW1lVGlja0Zvcm1hdHRlciJ9LHsiYXR0cmlidXRlcyI6e30sImlkIjoiMTI2MCIsInR5cGUiOiJCYXNpY1RpY2tGb3JtYXR0ZXIifSx7ImF0dHJpYnV0ZXMiOnsiZGF0YV9zb3VyY2UiOnsiaWQiOiIxMjMxIiwidHlwZSI6IkNvbHVtbkRhdGFTb3VyY2UifSwiZ2x5cGgiOnsiaWQiOiIxMjMyIiwidHlwZSI6IkxpbmUifSwiaG92ZXJfZ2x5cGgiOm51bGwsIm11dGVkX2dseXBoIjpudWxsLCJub25zZWxlY3Rpb25fZ2x5cGgiOnsiaWQiOiIxMjMzIiwidHlwZSI6IkxpbmUifSwic2VsZWN0aW9uX2dseXBoIjpudWxsLCJ2aWV3Ijp7ImlkIjoiMTIzNSIsInR5cGUiOiJDRFNWaWV3In19LCJpZCI6IjEyMzQiLCJ0eXBlIjoiR2x5cGhSZW5kZXJlciJ9LHsiYXR0cmlidXRlcyI6eyJsYWJlbCI6eyJ2YWx1ZSI6IlNFQ09PUkEvQ05BUFMifSwicmVuZGVyZXJzIjpbeyJpZCI6IjEyNDEiLCJ0eXBlIjoiR2x5cGhSZW5kZXJlciJ9XX0sImlkIjoiMTI1NCIsInR5cGUiOiJMZWdlbmRJdGVtIn0seyJhdHRyaWJ1dGVzIjp7fSwiaWQiOiIxMjc1IiwidHlwZSI6IlVuaW9uUmVuZGVyZXJzIn0seyJhdHRyaWJ1dGVzIjp7fSwiaWQiOiIxMjI0IiwidHlwZSI6IlJlc2V0VG9vbCJ9LHsiYXR0cmlidXRlcyI6e30sImlkIjoiMTI3OSIsInR5cGUiOiJVbmlvblJlbmRlcmVycyJ9LHsiYXR0cmlidXRlcyI6eyJzb3VyY2UiOnsiaWQiOiIxMjQ1IiwidHlwZSI6IkNvbHVtbkRhdGFTb3VyY2UifX0sImlkIjoiMTI0OSIsInR5cGUiOiJDRFNWaWV3In0seyJhdHRyaWJ1dGVzIjp7ImRheXMiOlsxLDE1XX0sImlkIjoiMTI2NyIsInR5cGUiOiJEYXlzVGlja2VyIn0seyJhdHRyaWJ1dGVzIjp7ImF4aXNfbGFiZWwiOiJXYXRlciBIZWlnaHQgKG0pIiwiZm9ybWF0dGVyIjp7ImlkIjoiMTI2MCIsInR5cGUiOiJCYXNpY1RpY2tGb3JtYXR0ZXIifSwicGxvdCI6eyJpZCI6IjEyMDIiLCJzdWJ0eXBlIjoiRmlndXJlIiwidHlwZSI6IlBsb3QifSwidGlja2VyIjp7ImlkIjoiMTIxOCIsInR5cGUiOiJCYXNpY1RpY2tlciJ9fSwiaWQiOiIxMjE3IiwidHlwZSI6IkxpbmVhckF4aXMifSx7ImF0dHJpYnV0ZXMiOnsibGluZV9jYXAiOiJyb3VuZCIsImxpbmVfY29sb3IiOiJjcmltc29uIiwibGluZV9qb2luIjoicm91bmQiLCJsaW5lX3dpZHRoIjo1LCJ4Ijp7ImZpZWxkIjoieCJ9LCJ5Ijp7ImZpZWxkIjoieSJ9fSwiaWQiOiIxMjMyIiwidHlwZSI6IkxpbmUifSx7ImF0dHJpYnV0ZXMiOnsibGluZV9hbHBoYSI6MC4xLCJsaW5lX2NhcCI6InJvdW5kIiwibGluZV9jb2xvciI6IiMxZjc3YjQiLCJsaW5lX2pvaW4iOiJyb3VuZCIsImxpbmVfd2lkdGgiOjUsIngiOnsiZmllbGQiOiJ4In0sInkiOnsiZmllbGQiOiJ5In19LCJpZCI6IjEyNDciLCJ0eXBlIjoiTGluZSJ9LHsiYXR0cmlidXRlcyI6eyJib3R0b21fdW5pdHMiOiJzY3JlZW4iLCJmaWxsX2FscGhhIjp7InZhbHVlIjowLjV9LCJmaWxsX2NvbG9yIjp7InZhbHVlIjoibGlnaHRncmV5In0sImxlZnRfdW5pdHMiOiJzY3JlZW4iLCJsZXZlbCI6Im92ZXJsYXkiLCJsaW5lX2FscGhhIjp7InZhbHVlIjoxLjB9LCJsaW5lX2NvbG9yIjp7InZhbHVlIjoiYmxhY2sifSwibGluZV9kYXNoIjpbNCw0XSwibGluZV93aWR0aCI6eyJ2YWx1ZSI6Mn0sInBsb3QiOm51bGwsInJlbmRlcl9tb2RlIjoiY3NzIiwicmlnaHRfdW5pdHMiOiJzY3JlZW4iLCJ0b3BfdW5pdHMiOiJzY3JlZW4ifSwiaWQiOiIxMjI2IiwidHlwZSI6IkJveEFubm90YXRpb24ifSx7ImF0dHJpYnV0ZXMiOnt9LCJpZCI6IjEyNzQiLCJ0eXBlIjoiU2VsZWN0aW9uIn0seyJhdHRyaWJ1dGVzIjp7ImxpbmVfYWxwaGEiOjAuNjUsImxpbmVfY2FwIjoicm91bmQiLCJsaW5lX2NvbG9yIjoiI2FlYzdlOCIsImxpbmVfam9pbiI6InJvdW5kIiwibGluZV93aWR0aCI6NSwieCI6eyJmaWVsZCI6IngifSwieSI6eyJmaWVsZCI6InkifX0sImlkIjoiMTI0NiIsInR5cGUiOiJMaW5lIn0seyJhdHRyaWJ1dGVzIjp7ImRheXMiOlsxLDQsNywxMCwxMywxNiwxOSwyMiwyNSwyOF19LCJpZCI6IjEyNjUiLCJ0eXBlIjoiRGF5c1RpY2tlciJ9LHsiYXR0cmlidXRlcyI6eyJwbG90IjpudWxsLCJ0ZXh0IjoiODQxOTMxNyJ9LCJpZCI6IjEyMDEiLCJ0eXBlIjoiVGl0bGUifSx7ImF0dHJpYnV0ZXMiOnsibnVtX21pbm9yX3RpY2tzIjo1LCJ0aWNrZXJzIjpbeyJpZCI6IjEyNjEiLCJ0eXBlIjoiQWRhcHRpdmVUaWNrZXIifSx7ImlkIjoiMTI2MiIsInR5cGUiOiJBZGFwdGl2ZVRpY2tlciJ9LHsiaWQiOiIxMjYzIiwidHlwZSI6IkFkYXB0aXZlVGlja2VyIn0seyJpZCI6IjEyNjQiLCJ0eXBlIjoiRGF5c1RpY2tlciJ9LHsiaWQiOiIxMjY1IiwidHlwZSI6IkRheXNUaWNrZXIifSx7ImlkIjoiMTI2NiIsInR5cGUiOiJEYXlzVGlja2VyIn0seyJpZCI6IjEyNjciLCJ0eXBlIjoiRGF5c1RpY2tlciJ9LHsiaWQiOiIxMjY4IiwidHlwZSI6Ik1vbnRoc1RpY2tlciJ9LHsiaWQiOiIxMjY5IiwidHlwZSI6Ik1vbnRoc1RpY2tlciJ9LHsiaWQiOiIxMjcwIiwidHlwZSI6Ik1vbnRoc1RpY2tlciJ9LHsiaWQiOiIxMjcxIiwidHlwZSI6Ik1vbnRoc1RpY2tlciJ9LHsiaWQiOiIxMjcyIiwidHlwZSI6IlllYXJzVGlja2VyIn1dfSwiaWQiOiIxMjEzIiwidHlwZSI6IkRhdGV0aW1lVGlja2VyIn0seyJhdHRyaWJ1dGVzIjp7fSwiaWQiOiIxMjEwIiwidHlwZSI6IkxpbmVhclNjYWxlIn0seyJhdHRyaWJ1dGVzIjp7ImxhYmVsIjp7InZhbHVlIjoiZ2xvYmFsIn0sInJlbmRlcmVycyI6W3siaWQiOiIxMjQ4IiwidHlwZSI6IkdseXBoUmVuZGVyZXIifV19LCJpZCI6IjEyNTUiLCJ0eXBlIjoiTGVnZW5kSXRlbSJ9LHsiYXR0cmlidXRlcyI6eyJkYXlzIjpbMSwyLDMsNCw1LDYsNyw4LDksMTAsMTEsMTIsMTMsMTQsMTUsMTYsMTcsMTgsMTksMjAsMjEsMjIsMjMsMjQsMjUsMjYsMjcsMjgsMjksMzAsMzFdfSwiaWQiOiIxMjY0IiwidHlwZSI6IkRheXNUaWNrZXIifSx7ImF0dHJpYnV0ZXMiOnsiZGltZW5zaW9uIjoxLCJwbG90Ijp7ImlkIjoiMTIwMiIsInN1YnR5cGUiOiJGaWd1cmUiLCJ0eXBlIjoiUGxvdCJ9LCJ0aWNrZXIiOnsiaWQiOiIxMjE4IiwidHlwZSI6IkJhc2ljVGlja2VyIn19LCJpZCI6IjEyMjEiLCJ0eXBlIjoiR3JpZCJ9LHsiYXR0cmlidXRlcyI6eyJvdmVybGF5Ijp7ImlkIjoiMTIyNiIsInR5cGUiOiJCb3hBbm5vdGF0aW9uIn19LCJpZCI6IjEyMjMiLCJ0eXBlIjoiQm94Wm9vbVRvb2wifSx7ImF0dHJpYnV0ZXMiOnsibW9udGhzIjpbMCwxLDIsMyw0LDUsNiw3LDgsOSwxMCwxMV19LCJpZCI6IjEyNjgiLCJ0eXBlIjoiTW9udGhzVGlja2VyIn0seyJhdHRyaWJ1dGVzIjp7ImxpbmVfYWxwaGEiOjAuMSwibGluZV9jYXAiOiJyb3VuZCIsImxpbmVfY29sb3IiOiIjMWY3N2I0IiwibGluZV9qb2luIjoicm91bmQiLCJsaW5lX3dpZHRoIjo1LCJ4Ijp7ImZpZWxkIjoieCJ9LCJ5Ijp7ImZpZWxkIjoieSJ9fSwiaWQiOiIxMjQwIiwidHlwZSI6IkxpbmUifSx7ImF0dHJpYnV0ZXMiOnsiY2FsbGJhY2siOm51bGwsImRhdGEiOnsieCI6eyJfX25kYXJyYXlfXyI6IkFBQ0FWcHNkZGtJQUFHakZuaDEyUWdBQVVEU2lIWFpDQUFBNG82VWRka0lBQUNBU3FSMTJRZ0FBQ0lHc0hYWkNBQUR3NzY4ZGRrSUFBTmhlc3gxMlFnQUF3TTIySFhaQ0FBQ29QTG9kZGtJQUFKQ3J2UjEyUWdBQWVCckJIWFpDQUFCZ2ljUWRka0lBQUVqNHh4MTJRZ0FBTUdmTEhYWkNBQUFZMXM0ZGRrSUFBQUJGMGgxMlFnQUE2TFBWSFhaQ0FBRFFJdGtkZGtJQUFMaVIzQjEyUWdBQW9BRGdIWFpDQUFDSWIrTWRka0lBQUhEZTVoMTJRZ0FBV0UzcUhYWkNBQUFBSWtBZWRrSUFBT2lRUXg1MlFnQUEwUDlHSG5aQ0FBQzRia29lZGtJQUFLRGRUUjUyUWdBQWlFeFJIblpDQUFCd3UxUWVka0lBQUZncVdCNTJRZ0FBUUpsYkhuWkNBQUFvQ0Y4ZWRrSUFBQkIzWWg1MlFnQUErT1ZsSG5aQ0FBRGdWR2tlZGtJQUFNakRiQjUyUWdBQXNESndIblpDQUFDWW9YTWVka0lBQUlBUWR4NTJRZ0FBYUg5NkhuWkNBQUJRN24wZWRrSUFBRGhkZ1I1MlFnQUFJTXlFSG5aQ0FBQUlPNGdlZGtJQUFQQ3BpeDUyUWdBQTJCaVBIblpDQUFEQWg1SWVka0lBQUtqMmxSNTJRZ0FBa0dXWkhuWkNBQUI0MUp3ZWRrSUFBR0JEb0I1MlFnQUFTTEtqSG5aQ0FBQXdJYWNlZGtJQUFCaVFxaDUyUWdBQUFQK3RIblpDQUFEb2JiRWVka0lBQU5EY3RCNTJRZ0FBdUV1NEhuWkNBQUNndXJzZWRrSUFBSWdwdng1MlFnQUFjSmpDSG5aQ0FBQllCOFllZGtJQUFFQjJ5UjUyUWdBQUtPWE1IblpDQUFBUVZOQWVka0lBQVBqQzB4NTJRZ0FBNERIWEhuWkNBQURJb05vZWRrSUFBTEFQM2g1MlFnQUFtSDdoSG5aQ0FBQ0E3ZVFlZGtJQUFHaGM2QjUyUWdBQVVNdnJIblpDQUFBNE91OGVka0lBQUNDcDhoNTJRZ0FBQ0JqMkhuWkNBQUR3aHZrZWRrSUFBTmoxL0I1MlFnQUF3R1FBSDNaQ0FBQ28wd01mZGtJQUFKQkNCeDkyUWdBQWVMRUtIM1pDQUFCZ0lBNGZka0lBQUVpUEVSOTJRZ0FBTVA0VUgzWkNBQUFZYlJnZmRrSUFBQURjR3g5MlFnQUE2RW9mSDNaQ0FBRFF1U0lmZGtJQUFMZ29KaDkyUWdBQW9KY3BIM1pDQUFDSUJpMGZka0lBQUhCMU1COTJRZ0FBV09RekgzWkMiLCJkdHlwZSI6ImZsb2F0NjQiLCJzaGFwZSI6Wzk2XX0sInkiOnsiX19uZGFycmF5X18iOiJBQUFBQUVpWS96OEFBQURnVnZIM1B3QUFBTUJsU3ZBL0FBQUFZT2xHNFQ4QUFBQ2djQkRtdndBQUFFRGxzLzYvQUFBQUlNa3ZDY0FBQUFDZzZyTUZ3QUFBQUNBTU9BTEFBQUFBUUZ0NC9iOEFBQURnQkZ2anZ3QUFBS0NzT3VRL0FBQUFJQy9vL1Q4QUFBQ0FMTnY3UHdBQUFBQXF6dmsvQUFBQVlDZkI5ejhBQUFBQXdNaTBQd0FBQUdBUEtQVy9BQUFBWUZYT0JjQUFBQURBZ21zRndBQUFBQ0N3Q0FYQUFBQUFnTjJsQk1BQUFBRGdxNzBEd0FBQUFFQjYxUUxBQUFBQWdQaTYzajhBQUFCQXdLWHlQd0FBQUdEQ25QMC9BQUFBUU9KSkJFQUFBQUFnV3pmMlB3QUFBS0NOMTg0L0FBQUFZTzhDN2I4QUFBQmdDbVA4dndBQUFJQk9JZ1hBQUFBQTRCY1RETUFBQUFCZ01oc0N3QUFBQUtDWlJ2Qy9BQUFBQUl0SnpUOEFBQURnSGYzeVB3QUFBRUNGS0FGQUFBQUFnSHZTQ0VBQUFBQUFHbjBCUUFBQUFNQndUL1EvQUFBQWdMYVMxajhBQUFBQXFiYm92d0FBQUtCV1cvNi9BQUFBWUt3dENNQUFBQUJBZ29rQXdBQUFBQUN3eXZHL0FBQUFnTjBTeEw4QUFBQ2duVmJ0UHdBQUFFRDUyUDgvQUFBQTRGR0RDRUFBQUFBZ3Noa0NRQUFBQUtBa1lQYy9BQUFBSU1vWjVUOEFBQUJnTWVYaXZ3QUFBSUFXY3YyL0FBQUFJTXE0Q01BQUFBQ2dGdWtDd0FBQUFFREdNdnEvQUFBQVlMNG03YjhBQUFEZzFHSFRQd0FBQUtCSlJQZy9BQUFBQUEvWUJVQUFBQURnT09FQlFBQUFBR0RGMVBzL0FBQUFBQm5uOHo4QUFBQmc1NC9DdndBQUFPQVNpL2kvQUFBQVlCUmlCOEFBQUFBZ2thd0R3QUFBQU9BYjd2Ky9BQUFBZ0JXRCtMOEFBQUFBS0RIUXZ3QUFBSUNCYXZBL0FBQUFnS1p3QWtBQUFBQWc5M01BUUFBQUFHQ1A3dncvQUFBQWdERDErRDhBQUFBQWZNaThQd0FBQUFBaFhQVy9BQUFBNEdSQ0JzQUFBQURnaXc0RndBQUFBT0N5MmdQQUFBQUE0Tm1tQXNBQUFBQ2dyaGZ2dndBQUFLQVUyTmcvQUFBQW9PSDMrejhBQUFDQTNObjhQd0FBQUdEWHUvMC9BQUFBUU5LZC9qOEFBQUFBRDBQZVB3QUFBSUNWK082L0FBQUFvS3hFQThBQUFBQ2dyRVFEd0FBQUFLQ3NSQVBBIiwiZHR5cGUiOiJmbG9hdDY0Iiwic2hhcGUiOls5Nl19fSwic2VsZWN0ZWQiOnsiaWQiOiIxMjc2IiwidHlwZSI6IlNlbGVjdGlvbiJ9LCJzZWxlY3Rpb25fcG9saWN5Ijp7ImlkIjoiMTI3NyIsInR5cGUiOiJVbmlvblJlbmRlcmVycyJ9fSwiaWQiOiIxMjM4IiwidHlwZSI6IkNvbHVtbkRhdGFTb3VyY2UifSx7ImF0dHJpYnV0ZXMiOnsiYmFzZSI6NjAsIm1hbnRpc3NhcyI6WzEsMiw1LDEwLDE1LDIwLDMwXSwibWF4X2ludGVydmFsIjoxODAwMDAwLjAsIm1pbl9pbnRlcnZhbCI6MTAwMC4wLCJudW1fbWlub3JfdGlja3MiOjB9LCJpZCI6IjEyNjIiLCJ0eXBlIjoiQWRhcHRpdmVUaWNrZXIifSx7ImF0dHJpYnV0ZXMiOnsiY2FsbGJhY2siOm51bGwsInJlbmRlcmVycyI6W3siaWQiOiIxMjQxIiwidHlwZSI6IkdseXBoUmVuZGVyZXIifV0sInRvb2x0aXBzIjpbWyJOYW1lIiwiU0VDT09SQS9DTkFQUyJdLFsiQmlhcyIsIi0yLjE5Il0sWyJTa2lsbCIsIjEuMjEiXV19LCJpZCI6IjEyNDMiLCJ0eXBlIjoiSG92ZXJUb29sIn0seyJhdHRyaWJ1dGVzIjp7ImNhbGxiYWNrIjpudWxsLCJkYXRhIjp7IngiOnsiX19uZGFycmF5X18iOiJBQUNBVnBzZGRrSUFBR2pGbmgxMlFnQUFVRFNpSFhaQ0FBQTRvNlVkZGtJQUFDQVNxUjEyUWdBQUNJR3NIWFpDQUFEdzc2OGRka0lBQU5oZXN4MTJRZ0FBd00yMkhYWkNBQUNvUExvZGRrSUFBSkNydlIxMlFnQUFlQnJCSFhaQ0FBQmdpY1FkZGtJQUFFajR4eDEyUWdBQU1HZkxIWFpDQUFBWTFzNGRka0lBQUFCRjBoMTJRZ0FBNkxQVkhYWkNBQURRSXRrZGRrSUFBTGlSM0IxMlFnQUFvQURnSFhaQ0FBQ0liK01kZGtJQUFIRGU1aDEyUWdBQVdFM3FIWFpDQUFCQXZPMGRka0lBQUNncjhSMTJRZ0FBRUpyMEhYWkNBQUQ0Q1BnZGRrSUFBT0IzK3gxMlFnQUF5T2IrSFhaQ0FBQ3dWUUllZGtJQUFKakVCUjUyUWdBQWdETUpIblpDQUFCb29nd2Vka0lBQUZBUkVCNTJRZ0FBT0lBVEhuWkNBQUFnN3hZZWRrSUFBQWhlR2g1MlFnQUE4TXdkSG5aQ0FBRFlPeUVlZGtJQUFNQ3FKQjUyUWdBQXFCa29IblpDQUFDUWlDc2Vka0lBQUhqM0xoNTJRZ0FBWUdZeUhuWkNBQUJJMVRVZWRrSUFBREJFT1I1MlFnQUFHTE04SG5aQ0FBQUFJa0FlZGtJQUFPaVFReDUyUWdBQTBQOUdIblpDQUFDNGJrb2Vka0lBQUtEZFRSNTJRZ0FBaUV4UkhuWkNBQUJ3dTFRZWRrSUFBRmdxV0I1MlFnQUFRSmxiSG5aQ0FBQW9DRjhlZGtJQUFCQjNZaDUyUWdBQStPVmxIblpDQUFEZ1ZHa2Vka0lBQU1qRGJCNTJRZ0FBc0RKd0huWkNBQUNZb1hNZWRrSUFBSUFRZHg1MlFnQUFhSDk2SG5aQ0FBQlE3bjBlZGtJQUFEaGRnUjUyUWdBQUlNeUVIblpDQUFBSU80Z2Vka0lBQVBDcGl4NTJRZ0FBMkJpUEhuWkNBQURBaDVJZWRrSUFBS2oybFI1MlFnQUFrR1daSG5aQ0FBQjQxSndlZGtJQUFHQkRvQjUyUWdBQVNMS2pIblpDQUFBd0lhY2Vka0lBQUJpUXFoNTJRZ0FBQVArdEhuWkNBQURvYmJFZWRrSUFBTkRjdEI1MlFnQUF1RXU0SG5aQ0FBQ2d1cnNlZGtJQUFJZ3B2eDUyUWdBQWNKakNIblpDQUFCWUI4WWVka0lBQUVCMnlSNTJRZ0FBS09YTUhuWkNBQUFRVk5BZWRrSUFBUGpDMHg1MlFnQUE0REhYSG5aQ0FBRElvTm9lZGtJQUFMQVAzaDUyUWdBQW1IN2hIblpDQUFDQTdlUWVka0lBQUdoYzZCNTJRZ0FBVU12ckhuWkNBQUE0T3U4ZWRrSUFBQ0NwOGg1MlFnQUFDQmoySG5aQ0FBRHdodmtlZGtJQUFOajEvQjUyUWdBQXdHUUFIM1pDQUFDbzB3TWZka0lBQUpCQ0J4OTJRZ0FBZUxFS0gzWkNBQUJnSUE0ZmRrSUFBRWlQRVI5MlFnQUFNUDRVSDNaQ0FBQVliUmdmZGtJQUFBRGNHeDkyUWdBQTZFb2ZIM1pDQUFEUXVTSWZka0lBQUxnb0poOTJRZ0FBb0pjcEgzWkNBQUNJQmkwZmRrSUFBSEIxTUI5MlFnQUFXT1F6SDNaQ0FBQkFVemNmZGtJPSIsImR0eXBlIjoiZmxvYXQ2NCIsInNoYXBlIjpbMTIxXX0sInkiOnsiX19uZGFycmF5X18iOiJMYktkNzZmR0FFQ2dHaS9kSkFZRlFMZ2VoZXRSdUFaQVJiYnovZFI0QlVDdVIrRjZGSzRCUUwrZkdpL2RKUGcvdjU4YUw5MGs1ais3U1F3Q0s0ZTJQNHhzNS91cDhiSy9wSEE5Q3RlandEK2N4Q0N3Y21qbFAveXA4ZEpOWXZZL3hDQ3djbWlSQVVETnpNek16TXdHUU5FaTIvbCthZ2xBS1Z5UHd2VW9DVUJXRGkyeW5lOEZRRVNMYk9mN3FmOC9pVUZnNWRBaThUOGNXbVE3MzAvTlAxZzV0TWgydnMrL2h4Ylp6dmRUMDc5S0RBSXJoeGFwUHkyeW5lK254dWMvemN6TXpNek0rRCtQd3ZVb1hJOENRR3E4ZEpNWUJBZEFya2ZoZWhTdUNFQ0lGdG5POTFNSFFLQWFMOTBrQmdOQXhrczNpVUZnK1QvcEpqRUlyQnptUCtGNkZLNUg0Ym8vT3JUSWRyNmZ1cjhHZ1pWRGkyekhQL0xTVFdJUVdPay9XRG0weUhhKytUK2lSYmJ6L2RRRFFIU1RHQVJXRGdsQWwyNFNnOERLQzBEOHFmSFNUV0lMUUNQYitYNXF2QWRBNmlZeENLd2NBVUQ0VStPbG04VHlQNmpHU3plSlFkQS9JYkJ5YUpIdHpMOFlCRllPTGJMTnZ3clhvM0E5Q3NjL29VVzI4LzNVN0Qvc1ViZ2VoZXY3UDVxWm1abVptUVJBalpkdUVvUEFDRUF6TXpNek16TUtRTEZ5YUpIdGZBaEFSSXRzNS91cEEwQ2FtWm1abVpuNVA1ekVJTEJ5YU9VLzRYb1Vya2ZodWorN1NRd0NLNGVXdndyWG8zQTlDdGMvRGkyeW5lK244RCtObDI0U2c4RCtQNDJYYmhLRHdBWkF3L1VvWEkvQ0MwRDJLRnlQd3ZVT1FJeHM1L3VwOFE1QVhJL0M5U2hjQzBCZnVra01BaXNGUUthYnhDQ3djdm8vYzJpUjdYdy82VCtNYk9mN3FmSFNQeXlIRnRuTzk5cy9hWkh0ZkQ4MTdqOUk0WG9VcmtmNVB4MWFaRHZmVHdOQWJlZjdxZkhTQ0VCYVpEdmZUNDBNUUxUSWRyNmZHZzVBeFNDd2NtaVJERUNrY0QwSzE2TUhRSE5va2UxOFB3QkFJOXY1Zm1xODhqOFVya2ZoZWhUaVAyOFNnOERLb2VFL3Uwa01BaXVINmo5QllPWFFJdHYzUDkwa0JvR1ZRd0pBSmpFSXJCeGFDRUQ2Zm1xOGRKTU1RUFlvWEkvQzlRNUEzMCtObDI0U0RrQ3E4ZEpOWWhBS1FLNUg0WG9VcmdOQVptWm1abVptK0QvVWVPa21NUWpvUDBBMVhycEpETm8vMDAxaUVGZzU0RCt4Y21pUjdYenZQeCtGNjFHNEh2cy9nWlZEaTJ6bkEwQ1RHQVJXRGkwSlFERUlyQnhhWkF4QWFaSHRmRDgxRFVBMlhycEpEQUlMUUJmWnp2ZFQ0d1ZBQm9HVlE0dHMvVDlva2UxOFB6WHdQOGwydnA4YUwrRS9KUWFCbFVPTDREK0pRV0RsMENMclAyNFNnOERLb2ZjL040bEJZT1hRQVVCRWkyem4rNmtIUUZ5UHd2VW9YQXRBNlNZeENLd2NEVUFFVmc0dHNwMExRRVNMYk9mN3FRZEFBaXVIRnRuT0FFQTlDdGVqY0QzMFA0R1ZRNHRzNStNLy9kUjQ2U1l4MkQ4PSIsImR0eXBlIjoiZmxvYXQ2NCIsInNoYXBlIjpbMTIxXX19LCJzZWxlY3RlZCI6eyJpZCI6IjEyNzQiLCJ0eXBlIjoiU2VsZWN0aW9uIn0sInNlbGVjdGlvbl9wb2xpY3kiOnsiaWQiOiIxMjc1IiwidHlwZSI6IlVuaW9uUmVuZGVyZXJzIn19LCJpZCI6IjEyMzEiLCJ0eXBlIjoiQ29sdW1uRGF0YVNvdXJjZSJ9LHsiYXR0cmlidXRlcyI6eyJhY3RpdmVfZHJhZyI6ImF1dG8iLCJhY3RpdmVfaW5zcGVjdCI6ImF1dG8iLCJhY3RpdmVfbXVsdGkiOm51bGwsImFjdGl2ZV9zY3JvbGwiOiJhdXRvIiwiYWN0aXZlX3RhcCI6ImF1dG8iLCJ0b29scyI6W3siaWQiOiIxMjIyIiwidHlwZSI6IlBhblRvb2wifSx7ImlkIjoiMTIyMyIsInR5cGUiOiJCb3hab29tVG9vbCJ9LHsiaWQiOiIxMjI0IiwidHlwZSI6IlJlc2V0VG9vbCJ9LHsiaWQiOiIxMjM2IiwidHlwZSI6IkhvdmVyVG9vbCJ9LHsiaWQiOiIxMjQzIiwidHlwZSI6IkhvdmVyVG9vbCJ9LHsiaWQiOiIxMjUwIiwidHlwZSI6IkhvdmVyVG9vbCJ9XX0sImlkIjoiMTIyNSIsInR5cGUiOiJUb29sYmFyIn0seyJhdHRyaWJ1dGVzIjp7ImRhdGFfc291cmNlIjp7ImlkIjoiMTI0NSIsInR5cGUiOiJDb2x1bW5EYXRhU291cmNlIn0sImdseXBoIjp7ImlkIjoiMTI0NiIsInR5cGUiOiJMaW5lIn0sImhvdmVyX2dseXBoIjpudWxsLCJtdXRlZF9nbHlwaCI6bnVsbCwibm9uc2VsZWN0aW9uX2dseXBoIjp7ImlkIjoiMTI0NyIsInR5cGUiOiJMaW5lIn0sInNlbGVjdGlvbl9nbHlwaCI6bnVsbCwidmlldyI6eyJpZCI6IjEyNDkiLCJ0eXBlIjoiQ0RTVmlldyJ9fSwiaWQiOiIxMjQ4IiwidHlwZSI6IkdseXBoUmVuZGVyZXIifSx7ImF0dHJpYnV0ZXMiOnt9LCJpZCI6IjEyNzYiLCJ0eXBlIjoiU2VsZWN0aW9uIn0seyJhdHRyaWJ1dGVzIjp7ImNsaWNrX3BvbGljeSI6Im11dGUiLCJpdGVtcyI6W3siaWQiOiIxMjUzIiwidHlwZSI6IkxlZ2VuZEl0ZW0ifSx7ImlkIjoiMTI1NCIsInR5cGUiOiJMZWdlbmRJdGVtIn0seyJpZCI6IjEyNTUiLCJ0eXBlIjoiTGVnZW5kSXRlbSJ9XSwibG9jYXRpb24iOlswLDYwXSwicGxvdCI6eyJpZCI6IjEyMDIiLCJzdWJ0eXBlIjoiRmlndXJlIiwidHlwZSI6IlBsb3QifX0sImlkIjoiMTI1MiIsInR5cGUiOiJMZWdlbmQifSx7ImF0dHJpYnV0ZXMiOnsic291cmNlIjp7ImlkIjoiMTIzOCIsInR5cGUiOiJDb2x1bW5EYXRhU291cmNlIn19LCJpZCI6IjEyNDIiLCJ0eXBlIjoiQ0RTVmlldyJ9LHsiYXR0cmlidXRlcyI6eyJjYWxsYmFjayI6bnVsbCwiZGF0YSI6eyJ4Ijp7Il9fbmRhcnJheV9fIjoiQUFDQVZwc2Rka0lBQUdqRm5oMTJRZ0FBVURTaUhYWkNBQUJBdk8wZGRrSUFBQ2dyOFIxMlFnQUFFSnIwSFhaQ0FBQUFJa0FlZGtJQUFPaVFReDUyUWdBQTBQOUdIblpDQUFEQWg1SWVka0lBQUtqMmxSNTJRZ0FBa0dXWkhuWkNBQUNBN2VRZWRrSUFBR2hjNkI1MlFnQUFVTXZySG5aQyIsImR0eXBlIjoiZmxvYXQ2NCIsInNoYXBlIjpbMTVdfSwieSI6eyJfX25kYXJyYXlfXyI6IkFBQUE0QkxwNEw4QUFBRGdaQUhodndBQUFPQzJHZUcvQUFBQUlNTXc0NzhBQUFEZ1Nocmp2d0FBQU1EU0ErTy9BQUFBQUg0VjRiOEFBQUNnYlg3Z3Z3QUFBSUM2enQrL0FBQUFRS1ZmdDc4QUFBQWdQekczdndBQUFDRFpBcmUvQUFBQUlCTUdzNzhBQUFBZ0V3YXp2d0FBQUNBVEJyTy8iLCJkdHlwZSI6ImZsb2F0NjQiLCJzaGFwZSI6WzE1XX19LCJzZWxlY3RlZCI6eyJpZCI6IjEyNzgiLCJ0eXBlIjoiU2VsZWN0aW9uIn0sInNlbGVjdGlvbl9wb2xpY3kiOnsiaWQiOiIxMjc5IiwidHlwZSI6IlVuaW9uUmVuZGVyZXJzIn19LCJpZCI6IjEyNDUiLCJ0eXBlIjoiQ29sdW1uRGF0YVNvdXJjZSJ9LHsiYXR0cmlidXRlcyI6eyJtb250aHMiOlswLDQsOF19LCJpZCI6IjEyNzAiLCJ0eXBlIjoiTW9udGhzVGlja2VyIn0seyJhdHRyaWJ1dGVzIjp7Im1vbnRocyI6WzAsMiw0LDYsOCwxMF19LCJpZCI6IjEyNjkiLCJ0eXBlIjoiTW9udGhzVGlja2VyIn0seyJhdHRyaWJ1dGVzIjp7InBsb3QiOnsiaWQiOiIxMjAyIiwic3VidHlwZSI6IkZpZ3VyZSIsInR5cGUiOiJQbG90In0sInRpY2tlciI6eyJpZCI6IjEyMTMiLCJ0eXBlIjoiRGF0ZXRpbWVUaWNrZXIifX0sImlkIjoiMTIxNiIsInR5cGUiOiJHcmlkIn1dLCJyb290X2lkcyI6WyIxMjAyIl19LCJ0aXRsZSI6IkJva2VoIEFwcGxpY2F0aW9uIiwidmVyc2lvbiI6IjEuMC4xIn19CiAgICAgICAgPC9zY3JpcHQ+CiAgICAgICAgPHNjcmlwdCB0eXBlPSJ0ZXh0L2phdmFzY3JpcHQiPgogICAgICAgICAgKGZ1bmN0aW9uKCkgewogICAgICAgICAgICB2YXIgZm4gPSBmdW5jdGlvbigpIHsKICAgICAgICAgICAgICBCb2tlaC5zYWZlbHkoZnVuY3Rpb24oKSB7CiAgICAgICAgICAgICAgICAoZnVuY3Rpb24ocm9vdCkgewogICAgICAgICAgICAgICAgICBmdW5jdGlvbiBlbWJlZF9kb2N1bWVudChyb290KSB7CiAgICAgICAgICAgICAgICAgICAgCiAgICAgICAgICAgICAgICAgIHZhciBkb2NzX2pzb24gPSBkb2N1bWVudC5nZXRFbGVtZW50QnlJZCgnMTQ0OCcpLnRleHRDb250ZW50OwogICAgICAgICAgICAgICAgICB2YXIgcmVuZGVyX2l0ZW1zID0gW3siZG9jaWQiOiIxMTFlY2M1OC05OTM3LTQ2NWQtYjE1ZS1jNjM3Nzc5OGExODQiLCJyb290cyI6eyIxMjAyIjoiMGUwZjZiN2ItYmU0Zi00ZmJmLTlhNDUtMDRiYTc1ZTYwYmNlIn19XTsKICAgICAgICAgICAgICAgICAgcm9vdC5Cb2tlaC5lbWJlZC5lbWJlZF9pdGVtcyhkb2NzX2pzb24sIHJlbmRlcl9pdGVtcyk7CiAgICAgICAgICAgICAgICAKICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgICBpZiAocm9vdC5Cb2tlaCAhPT0gdW5kZWZpbmVkKSB7CiAgICAgICAgICAgICAgICAgICAgZW1iZWRfZG9jdW1lbnQocm9vdCk7CiAgICAgICAgICAgICAgICAgIH0gZWxzZSB7CiAgICAgICAgICAgICAgICAgICAgdmFyIGF0dGVtcHRzID0gMDsKICAgICAgICAgICAgICAgICAgICB2YXIgdGltZXIgPSBzZXRJbnRlcnZhbChmdW5jdGlvbihyb290KSB7CiAgICAgICAgICAgICAgICAgICAgICBpZiAocm9vdC5Cb2tlaCAhPT0gdW5kZWZpbmVkKSB7CiAgICAgICAgICAgICAgICAgICAgICAgIGVtYmVkX2RvY3VtZW50KHJvb3QpOwogICAgICAgICAgICAgICAgICAgICAgICBjbGVhckludGVydmFsKHRpbWVyKTsKICAgICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICAgICAgIGF0dGVtcHRzKys7CiAgICAgICAgICAgICAgICAgICAgICBpZiAoYXR0ZW1wdHMgPiAxMDApIHsKICAgICAgICAgICAgICAgICAgICAgICAgY29uc29sZS5sb2coIkJva2VoOiBFUlJPUjogVW5hYmxlIHRvIHJ1biBCb2tlaEpTIGNvZGUgYmVjYXVzZSBCb2tlaEpTIGxpYnJhcnkgaXMgbWlzc2luZyIpOwogICAgICAgICAgICAgICAgICAgICAgICBjbGVhckludGVydmFsKHRpbWVyKTsKICAgICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICAgICB9LCAxMCwgcm9vdCkKICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgfSkod2luZG93KTsKICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgfTsKICAgICAgICAgICAgaWYgKGRvY3VtZW50LnJlYWR5U3RhdGUgIT0gImxvYWRpbmciKSBmbigpOwogICAgICAgICAgICBlbHNlIGRvY3VtZW50LmFkZEV2ZW50TGlzdGVuZXIoIkRPTUNvbnRlbnRMb2FkZWQiLCBmbik7CiAgICAgICAgICB9KSgpOwogICAgICAgIDwvc2NyaXB0PgogICAgCiAgPC9ib2R5PgogIAo8L2h0bWw+&quot; width=&quot;790&quot; style=&quot;border:none !important;&quot; height=&quot;330&quot;></iframe>`)[0];
                popup_44cd7c6a42e242ec959bb3829b3763c2.setContent(i_frame_9f68bbcbd3ec485da7caecc755187418);


            marker_b3a94bbc34e043a4b8f3b7b86c957f22.bindPopup(popup_44cd7c6a42e242ec959bb3829b3763c2)
            ;




        var marker_7796e1a5ae3648308d75fde74420f82d = L.marker(
            [42.3539, -71.0503],
            {
                icon: new L.Icon.Default()
                }
            ).addTo(map_54cb488c2c244a8daafaa65ad6b9b54b);



                var icon_06f0a8071f5d4de6a229c756fa146f25 = L.AwesomeMarkers.icon({
                    icon: 'stats',
                    iconColor: 'white',
                    markerColor: 'green',
                    prefix: 'glyphicon',
                    extraClasses: 'fa-rotate-0'
                    });
                marker_7796e1a5ae3648308d75fde74420f82d.setIcon(icon_06f0a8071f5d4de6a229c756fa146f25);


            var popup_820c7aaa26f74cf388719d06cad72c8c = L.popup({maxWidth: '2650'

            });


                var i_frame_2e07c1aa64734d4c8b1fc5a1d0196e6c = $(`<iframe src=&quot;data:text/html;charset=utf-8;base64,CiAgICAKCgoKPCFET0NUWVBFIGh0bWw+CjxodG1sIGxhbmc9ImVuIj4KICAKICA8aGVhZD4KICAgIAogICAgICA8bWV0YSBjaGFyc2V0PSJ1dGYtOCI+CiAgICAgIDx0aXRsZT44NDQzOTcwPC90aXRsZT4KICAgICAgCiAgICAgIAogICAgICAgIAogICAgICAgICAgCiAgICAgICAgPGxpbmsgcmVsPSJzdHlsZXNoZWV0IiBocmVmPSJodHRwczovL2Nkbi5weWRhdGEub3JnL2Jva2VoL3JlbGVhc2UvYm9rZWgtMS4wLjEubWluLmNzcyIgdHlwZT0idGV4dC9jc3MiIC8+CiAgICAgICAgCiAgICAgICAgCiAgICAgICAgICAKICAgICAgICA8c2NyaXB0IHR5cGU9InRleHQvamF2YXNjcmlwdCIgc3JjPSJodHRwczovL2Nkbi5weWRhdGEub3JnL2Jva2VoL3JlbGVhc2UvYm9rZWgtMS4wLjEubWluLmpzIj48L3NjcmlwdD4KICAgICAgICA8c2NyaXB0IHR5cGU9InRleHQvamF2YXNjcmlwdCI+CiAgICAgICAgICAgIEJva2VoLnNldF9sb2dfbGV2ZWwoImluZm8iKTsKICAgICAgICA8L3NjcmlwdD4KICAgICAgICAKICAgICAgCiAgICAgIAogICAgCiAgPC9oZWFkPgogIAogIAogIDxib2R5PgogICAgCiAgICAgIAogICAgICAgIAogICAgICAgICAgCiAgICAgICAgICAKICAgICAgICAgICAgCiAgICAgICAgICAgICAgPGRpdiBjbGFzcz0iYmstcm9vdCIgaWQ9IjE5YjRkOWEwLTE5ZDUtNGUyOC1hNTk1LTIxNzIxMTgwYzQxOCI+PC9kaXY+CiAgICAgICAgICAgIAogICAgICAgICAgCiAgICAgICAgCiAgICAgIAogICAgICAKICAgICAgICA8c2NyaXB0IHR5cGU9ImFwcGxpY2F0aW9uL2pzb24iIGlkPSIxNjcyIj4KICAgICAgICAgIHsiNTZlYTI4ODYtZTM4ZS00NjA0LWE2ZTgtYjIwODc4MTYzZWUwIjp7InJvb3RzIjp7InJlZmVyZW5jZXMiOlt7ImF0dHJpYnV0ZXMiOnsiYm90dG9tX3VuaXRzIjoic2NyZWVuIiwiZmlsbF9hbHBoYSI6eyJ2YWx1ZSI6MC41fSwiZmlsbF9jb2xvciI6eyJ2YWx1ZSI6ImxpZ2h0Z3JleSJ9LCJsZWZ0X3VuaXRzIjoic2NyZWVuIiwibGV2ZWwiOiJvdmVybGF5IiwibGluZV9hbHBoYSI6eyJ2YWx1ZSI6MS4wfSwibGluZV9jb2xvciI6eyJ2YWx1ZSI6ImJsYWNrIn0sImxpbmVfZGFzaCI6WzQsNF0sImxpbmVfd2lkdGgiOnsidmFsdWUiOjJ9LCJwbG90IjpudWxsLCJyZW5kZXJfbW9kZSI6ImNzcyIsInJpZ2h0X3VuaXRzIjoic2NyZWVuIiwidG9wX3VuaXRzIjoic2NyZWVuIn0sImlkIjoiMTQ3NCIsInR5cGUiOiJCb3hBbm5vdGF0aW9uIn0seyJhdHRyaWJ1dGVzIjp7ImNhbGxiYWNrIjpudWxsLCJkYXRhIjp7IngiOnsiX19uZGFycmF5X18iOiJBQUNBVnBzZGRrSUFBR2pGbmgxMlFnQUFVRFNpSFhaQ0FBQTRvNlVkZGtJQUFDQVNxUjEyUWdBQUNJR3NIWFpDQUFEdzc2OGRka0lBQU5oZXN4MTJRZ0FBd00yMkhYWkNBQUNvUExvZGRrSUFBSkNydlIxMlFnQUFlQnJCSFhaQ0FBQmdpY1FkZGtJQUFFajR4eDEyUWdBQU1HZkxIWFpDQUFBWTFzNGRka0lBQUFCRjBoMTJRZ0FBNkxQVkhYWkNBQURRSXRrZGRrSUFBTGlSM0IxMlFnQUFvQURnSFhaQ0FBQ0liK01kZGtJQUFIRGU1aDEyUWdBQVdFM3FIWFpDQUFCQXZPMGRka0lBQUNncjhSMTJRZ0FBRUpyMEhYWkNBQUQ0Q1BnZGRrSUFBT0IzK3gxMlFnQUF5T2IrSFhaQ0FBQ3dWUUllZGtJQUFKakVCUjUyUWdBQWdETUpIblpDQUFCb29nd2Vka0lBQUZBUkVCNTJRZ0FBT0lBVEhuWkNBQUFnN3hZZWRrSUFBQWhlR2g1MlFnQUE4TXdkSG5aQ0FBRFlPeUVlZGtJQUFNQ3FKQjUyUWdBQXFCa29IblpDQUFDUWlDc2Vka0lBQUhqM0xoNTJRZ0FBWUdZeUhuWkNBQUJJMVRVZWRrSUFBREJFT1I1MlFnQUFHTE04SG5aQ0FBQUFJa0FlZGtJQUFPaVFReDUyUWdBQTBQOUdIblpDQUFDNGJrb2Vka0lBQUtEZFRSNTJRZ0FBaUV4UkhuWkNBQUJ3dTFRZWRrSUFBRmdxV0I1MlFnQUFRSmxiSG5aQ0FBQW9DRjhlZGtJQUFCQjNZaDUyUWdBQStPVmxIblpDQUFEZ1ZHa2Vka0lBQU1qRGJCNTJRZ0FBc0RKd0huWkNBQUNZb1hNZWRrSUFBSUFRZHg1MlFnQUFhSDk2SG5aQ0FBQlE3bjBlZGtJQUFEaGRnUjUyUWdBQUlNeUVIblpDQUFBSU80Z2Vka0lBQVBDcGl4NTJRZ0FBMkJpUEhuWkNBQURBaDVJZWRrSUFBS2oybFI1MlFnQUFrR1daSG5aQ0FBQjQxSndlZGtJQUFHQkRvQjUyUWdBQVNMS2pIblpDQUFBd0lhY2Vka0lBQUJpUXFoNTJRZ0FBQVArdEhuWkNBQURvYmJFZWRrSUFBTkRjdEI1MlFnQUF1RXU0SG5aQ0FBQ2d1cnNlZGtJQUFJZ3B2eDUyUWdBQWNKakNIblpDQUFCWUI4WWVka0lBQUVCMnlSNTJRZ0FBS09YTUhuWkNBQUFRVk5BZWRrSUFBUGpDMHg1MlFnQUE0REhYSG5aQ0FBRElvTm9lZGtJQUFMQVAzaDUyUWdBQW1IN2hIblpDQUFDQTdlUWVka0lBQUdoYzZCNTJRZ0FBVU12ckhuWkNBQUE0T3U4ZWRrSUFBQ0NwOGg1MlFnQUFDQmoySG5aQ0FBRHdodmtlZGtJQUFOajEvQjUyUWdBQXdHUUFIM1pDQUFDbzB3TWZka0lBQUpCQ0J4OTJRZ0FBZUxFS0gzWkNBQUJnSUE0ZmRrSUFBRWlQRVI5MlFnQUFNUDRVSDNaQ0FBQVliUmdmZGtJQUFBRGNHeDkyUWdBQTZFb2ZIM1pDQUFEUXVTSWZka0lBQUxnb0poOTJRZ0FBb0pjcEgzWkNBQUNJQmkwZmRrSUFBSEIxTUI5MlFnQUFXT1F6SDNaQ0FBQkFVemNmZGtJPSIsImR0eXBlIjoiZmxvYXQ2NCIsInNoYXBlIjpbMTIxXX0sInkiOnsiX19uZGFycmF5X18iOiJmVDgxWHJwSkFVQkk0WG9VcmtjR1FHbVI3WHcvTlFoQTZQdXA4ZEpOQjBDRjYxRzRIb1VEUUdNUVdEbTB5UHcvMDAxaUVGZzU4RC8wL2RSNDZTYlJQOHFoUmJiei9iUy9tcG1abVptWnVUK1dRNHRzNS92bFA2YWJ4Q0N3Y3ZZL2hldFJ1QjZGQVVDTmwyNFNnOEFIUUlnVzJjNzNVd3RBakd6bis2bnhDa0RhenZkVDQ2VUhRQXJYbzNBOUNnSkF1MGtNQWl1SDlqL0J5cUZGdHZQaFAzc1Vya2ZoZXNTL3BwdkVJTEJ5MkwrTWJPZjdxZkdpUHowSzE2TndQZVkvRW9QQXlxRkYrRDlNTjRsQllPVUNRQ1l4Q0t3Y1dnaEFleFN1UitGNkNrQldEaTJ5bmU4SVFEWmV1a2tNQWdWQWt1MThQelZlL2o5M3ZwOGFMOTN3UDRYclViZ2VoZE0vUE45UGpaZHVzci9EOVNoY2o4TEZQMjNuKzZueDB1ay9VcmdlaGV0UitqL1RUV0lRV0RrRVFBNHRzcDN2cHdwQXlxRkZ0dlA5RFVDUzdYdy9OVjROUUpMdGZEODFYZ2xBVXJnZWhldFJBMEJBTlY2NlNRejRQNFhyVWJnZWhlTS93TXFoUmJienZiL3kwazFpRUZqUnYzU1RHQVJXRHMwL2o4TDFLRnlQN2o5OVB6VmV1a244UDlFaTIvbCthZ1ZBcXZIU1RXSVFDMEJBTlY2NlNRd05RQU1yaHhiWnpncEFYSS9DOVNoY0JrREdTemVKUVdBQVFBclhvM0E5Q3ZNLy90UjQ2U1l4NEQ4WDJjNzNVK1BGUDd4MGt4Z0VWdUkveGtzM2lVRmc5VCsyOC8zVWVPa0FRQ1VHZ1pWRGl3aEF3TXFoUmJiekQwQUlyQnhhWkxzUlFBd0NLNGNXV1JGQVBRclhvM0E5RDBCM3ZwOGFMOTBKUU1RZ3NISm9rUUpBSmpFSXJCeGE5ajlXRGkyeW5lL25QMFNMYk9mN3FlVS9xTVpMTjRsQjlEOHpNek16TXpNQVFFb01BaXVIRmdWQVl4QllPYlRJQzBBM2lVRmc1VkFRUUIxYVpEdmZ6eEJBNUtXYnhDQ3dEa0RQOTFQanBac0pRR3E4ZEpNWUJBUkFBeXVIRnRuTzl6K1M3WHcvTlY3cVA1M3ZwOFpMTitFL1hJL0M5U2hjNnovNFUrT2xtOFQ0UDQyWGJoS0R3QUpBTGJLZDc2ZkdDa0QyS0Z5UHd2VVBRTXVoUmJiei9SQkFCRllPTGJJZEVFQ05sMjRTZzhBTFFOMGtCb0dWUXdaQXNYSm9rZTE4L1Q4djNTUUdnWlh2UDlFaTIvbCthdHcvT3JUSWRyNmYyajlPWWhCWU9iVHdQd0FBQUFBQUFQdy9PclRJZHI2ZkJFQ21tOFFnc0hJTFFLcngwazFpRUE5QWt1MThQelZlRDBEUDkxUGpwWnNNUUhzVXJrZmhlZ2RBakd6bis2bnhBRUNQd3ZVb1hJLzBQOXY1Zm1xOGRPTS9KakVJckJ4YTNEODNpVUZnNWREcVAybVI3WHcvTmZnLzh0Sk5ZaEJZQWtDa2NEMEsxNk1JUU81OFB6VmV1ZzFBU09GNkZLNUhEMERKZHI2ZkdpOE5RT1NsbThRZ3NBaEFCRllPTGJLZEFrQUVWZzR0c3AzM1A5ck85MVBqcGVjL0hGcGtPOTlQMVQ4PSIsImR0eXBlIjoiZmxvYXQ2NCIsInNoYXBlIjpbMTIxXX19LCJzZWxlY3RlZCI6eyJpZCI6IjE1MTYiLCJ0eXBlIjoiU2VsZWN0aW9uIn0sInNlbGVjdGlvbl9wb2xpY3kiOnsiaWQiOiIxNTE3IiwidHlwZSI6IlVuaW9uUmVuZGVyZXJzIn19LCJpZCI6IjE0ODYiLCJ0eXBlIjoiQ29sdW1uRGF0YVNvdXJjZSJ9LHsiYXR0cmlidXRlcyI6e30sImlkIjoiMTQ1OCIsInR5cGUiOiJMaW5lYXJTY2FsZSJ9LHsiYXR0cmlidXRlcyI6eyJiYXNlIjo2MCwibWFudGlzc2FzIjpbMSwyLDUsMTAsMTUsMjAsMzBdLCJtYXhfaW50ZXJ2YWwiOjE4MDAwMDAuMCwibWluX2ludGVydmFsIjoxMDAwLjAsIm51bV9taW5vcl90aWNrcyI6MH0sImlkIjoiMTUwMiIsInR5cGUiOiJBZGFwdGl2ZVRpY2tlciJ9LHsiYXR0cmlidXRlcyI6eyJiZWxvdyI6W3siaWQiOiIxNDYwIiwidHlwZSI6IkRhdGV0aW1lQXhpcyJ9XSwibGVmdCI6W3siaWQiOiIxNDY1IiwidHlwZSI6IkxpbmVhckF4aXMifV0sInBsb3RfaGVpZ2h0IjoyNTAsInBsb3Rfd2lkdGgiOjc1MCwicmVuZGVyZXJzIjpbeyJpZCI6IjE0NjAiLCJ0eXBlIjoiRGF0ZXRpbWVBeGlzIn0seyJpZCI6IjE0NjQiLCJ0eXBlIjoiR3JpZCJ9LHsiaWQiOiIxNDY1IiwidHlwZSI6IkxpbmVhckF4aXMifSx7ImlkIjoiMTQ2OSIsInR5cGUiOiJHcmlkIn0seyJpZCI6IjE0NzQiLCJ0eXBlIjoiQm94QW5ub3RhdGlvbiJ9LHsiaWQiOiIxNDgyIiwidHlwZSI6IkdseXBoUmVuZGVyZXIifSx7ImlkIjoiMTQ4OSIsInR5cGUiOiJHbHlwaFJlbmRlcmVyIn0seyJpZCI6IjE0OTMiLCJ0eXBlIjoiTGVnZW5kIn1dLCJyaWdodCI6W3siaWQiOiIxNDkzIiwidHlwZSI6IkxlZ2VuZCJ9XSwidGl0bGUiOnsiaWQiOiIxNDQ5IiwidHlwZSI6IlRpdGxlIn0sInRvb2xiYXIiOnsiaWQiOiIxNDczIiwidHlwZSI6IlRvb2xiYXIifSwidG9vbGJhcl9sb2NhdGlvbiI6ImFib3ZlIiwieF9yYW5nZSI6eyJpZCI6IjE0NTIiLCJ0eXBlIjoiRGF0YVJhbmdlMWQifSwieF9zY2FsZSI6eyJpZCI6IjE0NTYiLCJ0eXBlIjoiTGluZWFyU2NhbGUifSwieV9yYW5nZSI6eyJpZCI6IjE0NTQiLCJ0eXBlIjoiRGF0YVJhbmdlMWQifSwieV9zY2FsZSI6eyJpZCI6IjE0NTgiLCJ0eXBlIjoiTGluZWFyU2NhbGUifX0sImlkIjoiMTQ1MCIsInN1YnR5cGUiOiJGaWd1cmUiLCJ0eXBlIjoiUGxvdCJ9LHsiYXR0cmlidXRlcyI6eyJudW1fbWlub3JfdGlja3MiOjUsInRpY2tlcnMiOlt7ImlkIjoiMTUwMSIsInR5cGUiOiJBZGFwdGl2ZVRpY2tlciJ9LHsiaWQiOiIxNTAyIiwidHlwZSI6IkFkYXB0aXZlVGlja2VyIn0seyJpZCI6IjE1MDMiLCJ0eXBlIjoiQWRhcHRpdmVUaWNrZXIifSx7ImlkIjoiMTUwNCIsInR5cGUiOiJEYXlzVGlja2VyIn0seyJpZCI6IjE1MDUiLCJ0eXBlIjoiRGF5c1RpY2tlciJ9LHsiaWQiOiIxNTA2IiwidHlwZSI6IkRheXNUaWNrZXIifSx7ImlkIjoiMTUwNyIsInR5cGUiOiJEYXlzVGlja2VyIn0seyJpZCI6IjE1MDgiLCJ0eXBlIjoiTW9udGhzVGlja2VyIn0seyJpZCI6IjE1MDkiLCJ0eXBlIjoiTW9udGhzVGlja2VyIn0seyJpZCI6IjE1MTAiLCJ0eXBlIjoiTW9udGhzVGlja2VyIn0seyJpZCI6IjE1MTEiLCJ0eXBlIjoiTW9udGhzVGlja2VyIn0seyJpZCI6IjE1MTIiLCJ0eXBlIjoiWWVhcnNUaWNrZXIifV19LCJpZCI6IjE0NjEiLCJ0eXBlIjoiRGF0ZXRpbWVUaWNrZXIifSx7ImF0dHJpYnV0ZXMiOnsibGluZV9hbHBoYSI6MC4xLCJsaW5lX2NhcCI6InJvdW5kIiwibGluZV9jb2xvciI6IiMxZjc3YjQiLCJsaW5lX2pvaW4iOiJyb3VuZCIsImxpbmVfd2lkdGgiOjUsIngiOnsiZmllbGQiOiJ4In0sInkiOnsiZmllbGQiOiJ5In19LCJpZCI6IjE0ODgiLCJ0eXBlIjoiTGluZSJ9LHsiYXR0cmlidXRlcyI6eyJsaW5lX2NhcCI6InJvdW5kIiwibGluZV9jb2xvciI6ImNyaW1zb24iLCJsaW5lX2pvaW4iOiJyb3VuZCIsImxpbmVfd2lkdGgiOjUsIngiOnsiZmllbGQiOiJ4In0sInkiOnsiZmllbGQiOiJ5In19LCJpZCI6IjE0ODciLCJ0eXBlIjoiTGluZSJ9LHsiYXR0cmlidXRlcyI6eyJjYWxsYmFjayI6bnVsbCwicmVuZGVyZXJzIjpbeyJpZCI6IjE0ODkiLCJ0eXBlIjoiR2x5cGhSZW5kZXJlciJ9XSwidG9vbHRpcHMiOltbIk5hbWUiLCJPYnNlcnZhdGlvbnMiXSxbIkJpYXMiLCJOQSJdLFsiU2tpbGwiLCJOQSJdXX0sImlkIjoiMTQ5MSIsInR5cGUiOiJIb3ZlclRvb2wifSx7ImF0dHJpYnV0ZXMiOnsibW9udGhzIjpbMCw2XX0sImlkIjoiMTUxMSIsInR5cGUiOiJNb250aHNUaWNrZXIifSx7ImF0dHJpYnV0ZXMiOnsic291cmNlIjp7ImlkIjoiMTQ4NiIsInR5cGUiOiJDb2x1bW5EYXRhU291cmNlIn19LCJpZCI6IjE0OTAiLCJ0eXBlIjoiQ0RTVmlldyJ9LHsiYXR0cmlidXRlcyI6eyJsaW5lX2FscGhhIjowLjY1LCJsaW5lX2NhcCI6InJvdW5kIiwibGluZV9jb2xvciI6IiNmZjdmMGUiLCJsaW5lX2pvaW4iOiJyb3VuZCIsImxpbmVfd2lkdGgiOjUsIngiOnsiZmllbGQiOiJ4In0sInkiOnsiZmllbGQiOiJ5In19LCJpZCI6IjE0ODAiLCJ0eXBlIjoiTGluZSJ9LHsiYXR0cmlidXRlcyI6eyJjYWxsYmFjayI6bnVsbCwicmVuZGVyZXJzIjpbeyJpZCI6IjE0ODIiLCJ0eXBlIjoiR2x5cGhSZW5kZXJlciJ9XSwidG9vbHRpcHMiOltbIk5hbWUiLCJORUNPRlNfQm9zdG9uIl0sWyJCaWFzIiwiLTEuNjAiXSxbIlNraWxsIiwiMC4zNiJdXX0sImlkIjoiMTQ4NCIsInR5cGUiOiJIb3ZlclRvb2wifSx7ImF0dHJpYnV0ZXMiOnsiYWN0aXZlX2RyYWciOiJhdXRvIiwiYWN0aXZlX2luc3BlY3QiOiJhdXRvIiwiYWN0aXZlX211bHRpIjpudWxsLCJhY3RpdmVfc2Nyb2xsIjoiYXV0byIsImFjdGl2ZV90YXAiOiJhdXRvIiwidG9vbHMiOlt7ImlkIjoiMTQ3MCIsInR5cGUiOiJQYW5Ub29sIn0seyJpZCI6IjE0NzEiLCJ0eXBlIjoiQm94Wm9vbVRvb2wifSx7ImlkIjoiMTQ3MiIsInR5cGUiOiJSZXNldFRvb2wifSx7ImlkIjoiMTQ4NCIsInR5cGUiOiJIb3ZlclRvb2wifSx7ImlkIjoiMTQ5MSIsInR5cGUiOiJIb3ZlclRvb2wifV19LCJpZCI6IjE0NzMiLCJ0eXBlIjoiVG9vbGJhciJ9LHsiYXR0cmlidXRlcyI6eyJkaW1lbnNpb24iOjEsInBsb3QiOnsiaWQiOiIxNDUwIiwic3VidHlwZSI6IkZpZ3VyZSIsInR5cGUiOiJQbG90In0sInRpY2tlciI6eyJpZCI6IjE0NjYiLCJ0eXBlIjoiQmFzaWNUaWNrZXIifX0sImlkIjoiMTQ2OSIsInR5cGUiOiJHcmlkIn0seyJhdHRyaWJ1dGVzIjp7ImxpbmVfYWxwaGEiOjAuMSwibGluZV9jYXAiOiJyb3VuZCIsImxpbmVfY29sb3IiOiIjMWY3N2I0IiwibGluZV9qb2luIjoicm91bmQiLCJsaW5lX3dpZHRoIjo1LCJ4Ijp7ImZpZWxkIjoieCJ9LCJ5Ijp7ImZpZWxkIjoieSJ9fSwiaWQiOiIxNDgxIiwidHlwZSI6IkxpbmUifSx7ImF0dHJpYnV0ZXMiOnt9LCJpZCI6IjE1MTUiLCJ0eXBlIjoiVW5pb25SZW5kZXJlcnMifSx7ImF0dHJpYnV0ZXMiOnt9LCJpZCI6IjE1MTQiLCJ0eXBlIjoiU2VsZWN0aW9uIn0seyJhdHRyaWJ1dGVzIjp7InBsb3QiOm51bGwsInRleHQiOiI4NDQzOTcwIn0sImlkIjoiMTQ0OSIsInR5cGUiOiJUaXRsZSJ9LHsiYXR0cmlidXRlcyI6eyJjYWxsYmFjayI6bnVsbH0sImlkIjoiMTQ1NCIsInR5cGUiOiJEYXRhUmFuZ2UxZCJ9LHsiYXR0cmlidXRlcyI6e30sImlkIjoiMTUxNyIsInR5cGUiOiJVbmlvblJlbmRlcmVycyJ9LHsiYXR0cmlidXRlcyI6eyJsYWJlbCI6eyJ2YWx1ZSI6Ik5FQ09GU19Cb3N0b24ifSwicmVuZGVyZXJzIjpbeyJpZCI6IjE0ODIiLCJ0eXBlIjoiR2x5cGhSZW5kZXJlciJ9XX0sImlkIjoiMTQ5NCIsInR5cGUiOiJMZWdlbmRJdGVtIn0seyJhdHRyaWJ1dGVzIjp7ImNsaWNrX3BvbGljeSI6Im11dGUiLCJpdGVtcyI6W3siaWQiOiIxNDk0IiwidHlwZSI6IkxlZ2VuZEl0ZW0ifSx7ImlkIjoiMTQ5NSIsInR5cGUiOiJMZWdlbmRJdGVtIn1dLCJsb2NhdGlvbiI6WzAsNjBdLCJwbG90Ijp7ImlkIjoiMTQ1MCIsInN1YnR5cGUiOiJGaWd1cmUiLCJ0eXBlIjoiUGxvdCJ9fSwiaWQiOiIxNDkzIiwidHlwZSI6IkxlZ2VuZCJ9LHsiYXR0cmlidXRlcyI6eyJiYXNlIjoyNCwibWFudGlzc2FzIjpbMSwyLDQsNiw4LDEyXSwibWF4X2ludGVydmFsIjo0MzIwMDAwMC4wLCJtaW5faW50ZXJ2YWwiOjM2MDAwMDAuMCwibnVtX21pbm9yX3RpY2tzIjowfSwiaWQiOiIxNTAzIiwidHlwZSI6IkFkYXB0aXZlVGlja2VyIn0seyJhdHRyaWJ1dGVzIjp7ImF4aXNfbGFiZWwiOiJEYXRlL3RpbWUiLCJmb3JtYXR0ZXIiOnsiaWQiOiIxNDk4IiwidHlwZSI6IkRhdGV0aW1lVGlja0Zvcm1hdHRlciJ9LCJwbG90Ijp7ImlkIjoiMTQ1MCIsInN1YnR5cGUiOiJGaWd1cmUiLCJ0eXBlIjoiUGxvdCJ9LCJ0aWNrZXIiOnsiaWQiOiIxNDYxIiwidHlwZSI6IkRhdGV0aW1lVGlja2VyIn19LCJpZCI6IjE0NjAiLCJ0eXBlIjoiRGF0ZXRpbWVBeGlzIn0seyJhdHRyaWJ1dGVzIjp7Im1hbnRpc3NhcyI6WzEsMiw1XSwibWF4X2ludGVydmFsIjo1MDAuMCwibnVtX21pbm9yX3RpY2tzIjowfSwiaWQiOiIxNTAxIiwidHlwZSI6IkFkYXB0aXZlVGlja2VyIn0seyJhdHRyaWJ1dGVzIjp7Im1vbnRocyI6WzAsMSwyLDMsNCw1LDYsNyw4LDksMTAsMTFdfSwiaWQiOiIxNTA4IiwidHlwZSI6Ik1vbnRoc1RpY2tlciJ9LHsiYXR0cmlidXRlcyI6eyJtb250aHMiOlswLDQsOF19LCJpZCI6IjE1MTAiLCJ0eXBlIjoiTW9udGhzVGlja2VyIn0seyJhdHRyaWJ1dGVzIjp7fSwiaWQiOiIxNTAwIiwidHlwZSI6IkJhc2ljVGlja0Zvcm1hdHRlciJ9LHsiYXR0cmlidXRlcyI6eyJjYWxsYmFjayI6bnVsbCwiZGF0YSI6eyJ4Ijp7Il9fbmRhcnJheV9fIjoiQUFDQVZwc2Rka0lBQUdqRm5oMTJRZ0FBVURTaUhYWkNBQUE0bzZVZGRrSUFBQ0FTcVIxMlFnQUFDSUdzSFhaQ0FBRHc3NjhkZGtJQUFOaGVzeDEyUWdBQXdNMjJIWFpDQUFDb1BMb2Rka0lBQUpDcnZSMTJRZ0FBZUJyQkhYWkNBQUJnaWNRZGRrSUFBRWo0eHgxMlFnQUFNR2ZMSFhaQ0FBQVkxczRkZGtJQUFBQkYwaDEyUWdBQTZMUFZIWFpDQUFEUUl0a2Rka0lBQUxpUjNCMTJRZ0FBb0FEZ0hYWkNBQUNJYitNZGRrSUFBSERlNWgxMlFnQUFXRTNxSFhaQ0FBQkF2TzBkZGtJQUFDZ3I4UjEyUWdBQUVKcjBIWFpDQUFENENQZ2Rka0lBQU9CMyt4MTJRZ0FBeU9iK0hYWkNBQUN3VlFJZWRrSUFBSmpFQlI1MlFnQUFnRE1KSG5aQ0FBQm9vZ3dlZGtJQUFGQVJFQjUyUWdBQU9JQVRIblpDQUFBZzd4WWVka0lBQUFoZUdoNTJRZ0FBOE13ZEhuWkNBQURZT3lFZWRrSUFBTUNxSkI1MlFnQUFxQmtvSG5aQ0FBQ1FpQ3NlZGtJQUFIajNMaDUyUWdBQVlHWXlIblpDQUFCSTFUVWVka0lBQURCRU9SNTJRZ0FBR0xNOEhuWkNBQUFBSWtBZWRrSUFBT2lRUXg1MlFnQUEwUDlHSG5aQ0FBQzRia29lZGtJQUFLRGRUUjUyUWdBQWlFeFJIblpDQUFCd3UxUWVka0lBQUZncVdCNTJRZ0FBUUpsYkhuWkNBQUFvQ0Y4ZWRrSUFBQkIzWWg1MlFnQUErT1ZsSG5aQ0FBRGdWR2tlZGtJQUFNakRiQjUyUWdBQXNESndIblpDQUFDWW9YTWVka0lBQUlBUWR4NTJRZ0FBYUg5NkhuWkNBQUJRN24wZWRrSUFBRGhkZ1I1MlFnQUFJTXlFSG5aQ0FBQUlPNGdlZGtJQUFQQ3BpeDUyUWdBQTJCaVBIblpDQUFEQWg1SWVka0lBQUtqMmxSNTJRZ0FBa0dXWkhuWkNBQUI0MUp3ZWRrSUFBR0JEb0I1MlFnQUFTTEtqSG5aQ0FBQXdJYWNlZGtJQUFCaVFxaDUyUWdBQUFQK3RIblpDQUFEb2JiRWVka0lBQU5EY3RCNTJRZ0FBdUV1NEhuWkNBQUNndXJzZWRrSUFBSWdwdng1MlFnQUFjSmpDSG5aQ0FBQllCOFllZGtJQUFFQjJ5UjUyUWdBQUtPWE1IblpDQUFBUVZOQWVka0lBQVBqQzB4NTJRZ0FBNERIWEhuWkNBQURJb05vZWRrSUFBTEFQM2g1MlFnQUFtSDdoSG5aQ0FBQ0E3ZVFlZGtJQUFHaGM2QjUyUWdBQVVNdnJIblpDQUFBNE91OGVka0lBQUNDcDhoNTJRZ0FBQ0JqMkhuWkNBQUR3aHZrZWRrSUFBTmoxL0I1MlFnQUF3R1FBSDNaQ0FBQ28wd01mZGtJQUFKQkNCeDkyUWdBQWVMRUtIM1pDQUFCZ0lBNGZka0lBQUVpUEVSOTJRZ0FBTVA0VUgzWkNBQUFZYlJnZmRrSUFBQURjR3g5MlFnQUE2RW9mSDNaQ0FBRFF1U0lmZGtJQUFMZ29KaDkyUWdBQW9KY3BIM1pDQUFDSUJpMGZka0lBQUhCMU1COTJRZ0FBV09RekgzWkMiLCJkdHlwZSI6ImZsb2F0NjQiLCJzaGFwZSI6WzEyMF19LCJ5Ijp7Il9fbmRhcnJheV9fIjoiQUFBQUFJL2E2VDhBQUFDQUQzSHhQd0FBQUlEWDlQVS9BQUFBZ0o5NCtqOEFBQUFBYzYvdVB3QUFBRUJPMjlBL0FBQUFvRW1vMjc4QUFBQkFZMFBvdndBQUFNQlFXZkcvQUFBQUFQQ1E5cjhBQUFBQVRuZmx2d0FBQUtBZ21yRS9BQUFBSU5iZDZUOEFBQURBVy9qeVB3QUFBR0RNQWZrL0FBQUFBRDBML3o4QUFBQUE1TW56UHdBQUFNQVZFZUUvQUFBQUlISEd4YjhBQUFDQW5sbm52d0FBQUVEUW9QUy9BQUFBWU5HVS9iOEFBQUFnTTZMenZ3QUFBTUFwWCtPL0FBQUFnRm5Da0Q4QUFBRGdCY3pqUHdBQUFJRDhpUE0vQUFBQUFQWXIvVDhBQUFEZzBwYjBQd0FBQUlCZkErZy9BQUFBb0dSa3l6OEFBQUFBVnozVnZ3QUFBQ0J3RnV5L0FBQUFZQnJIOXI4QUFBQkFsZ250dndBQUFNRHZDZG0vQUFBQXdEVDl2ejhBQUFEZ1JUTG9Qd0FBQUtCeU12WS9BQUFBSU9FbEFFQUFBQUJBQ243NFB3QUFBR0JTc1BBL0FBQUF3RFRGNFQ4QUFBQUE3MjdCdndBQUFFQ3NmT3EvQUFBQVlNNU8rTDhBQUFEQWJwYnl2d0FBQUFBZXZPbS9BQUFBWUwyVzNMOEFBQURnTGhuU1B3QUFBTUJHTXZBL0FBQUE0RUhlK3o4QUFBREE4T1gzUHdBQUFJQ2Y3Zk0vQUFBQXdKenE3ejhBQUFCZzRWYlJQd0FBQU9CMko5Mi9BQUFBd0hQcDhyOEFBQUJnc3lqdHZ3QUFBQ0IvZnVTL0FBQUFBSmFvMTc4QUFBQ2d4QnpiUHdBQUFPQ0hlUE0vQUFBQVFPOFVBRUFBQUFEZzlSdi9Qd0FBQUVBTkR2NC9BQUFBb0NRQS9UOEFBQURBTVhueFB3QUFBT0Q3eU5jL0FBQUFnTTlTMXI4QUFBREFHd3JVdndBQUFBQm93ZEcvQUFBQW9Hanh6cjhBQUFBZzZ3M2VQd0FBQUtBaTVmSS9BQUFBZ01wRy9qOEFBQURndXpUL1B3QUFBS0JXRVFCQUFBQUFZRStJQUVBQUFBREFncVQwUHdBQUFFRE5jT0EvQUFBQW9OWE8wTDhBQUFCZ1llWFh2d0FBQUVEdCs5Ni9BQUFBZ0R3SjQ3OEFBQUJnb3ZURFB3QUFBS0NOQSswL0FBQUFZUG1FK2o4QUFBRGdRcHY5UHdBQUFFREdXQUJBQUFBQUFPdmpBVUFBQUFEZ09yUDNQd0FBQUtBL1BlYy9BQUFBNE5KK25iOEFBQUFBSDJuVXZ3QUFBSUFvZmVPL0FBQUFZTUhGN0w4QUFBQmd5bkRJdndBQUFFQmNqZUEvQUFBQWdIV2I4ejhBQUFEZ0tnejVQd0FBQUdEZ2ZQNC9BQUFBNE1yMkFVQUFBQUJnc0FYNVB3QUFBTUNWTyt3L0FBQUFvQ3V2eVQ4QUFBQ2dieUhMdndBQUFNQ0MvT08vQUFBQXdGU1k4TDhBQUFDZ1NjSFp2d0FBQUdCL3ZjMC9BQUFBZ0dTLzZ6OEFBQUFnenE3MFB3QUFBT0RwZmZzL0FBQUE0SUltQVVBQUFBQ0EvNy80UHdBQUFFRHlaZTQvQUFBQVlNdVgxajhBQUFCZ3k1ZldQd0FBQUdETGw5WS8iLCJkdHlwZSI6ImZsb2F0NjQiLCJzaGFwZSI6WzEyMF19fSwic2VsZWN0ZWQiOnsiaWQiOiIxNTE0IiwidHlwZSI6IlNlbGVjdGlvbiJ9LCJzZWxlY3Rpb25fcG9saWN5Ijp7ImlkIjoiMTUxNSIsInR5cGUiOiJVbmlvblJlbmRlcmVycyJ9fSwiaWQiOiIxNDc5IiwidHlwZSI6IkNvbHVtbkRhdGFTb3VyY2UifSx7ImF0dHJpYnV0ZXMiOnsiZGF0YV9zb3VyY2UiOnsiaWQiOiIxNDg2IiwidHlwZSI6IkNvbHVtbkRhdGFTb3VyY2UifSwiZ2x5cGgiOnsiaWQiOiIxNDg3IiwidHlwZSI6IkxpbmUifSwiaG92ZXJfZ2x5cGgiOm51bGwsIm11dGVkX2dseXBoIjpudWxsLCJub25zZWxlY3Rpb25fZ2x5cGgiOnsiaWQiOiIxNDg4IiwidHlwZSI6IkxpbmUifSwic2VsZWN0aW9uX2dseXBoIjpudWxsLCJ2aWV3Ijp7ImlkIjoiMTQ5MCIsInR5cGUiOiJDRFNWaWV3In19LCJpZCI6IjE0ODkiLCJ0eXBlIjoiR2x5cGhSZW5kZXJlciJ9LHsiYXR0cmlidXRlcyI6eyJkYXlzIjpbMSwxNV19LCJpZCI6IjE1MDciLCJ0eXBlIjoiRGF5c1RpY2tlciJ9LHsiYXR0cmlidXRlcyI6eyJkYXlzIjpbMSw4LDE1LDIyXX0sImlkIjoiMTUwNiIsInR5cGUiOiJEYXlzVGlja2VyIn0seyJhdHRyaWJ1dGVzIjp7fSwiaWQiOiIxNTEyIiwidHlwZSI6IlllYXJzVGlja2VyIn0seyJhdHRyaWJ1dGVzIjp7fSwiaWQiOiIxNDk4IiwidHlwZSI6IkRhdGV0aW1lVGlja0Zvcm1hdHRlciJ9LHsiYXR0cmlidXRlcyI6e30sImlkIjoiMTQ2NiIsInR5cGUiOiJCYXNpY1RpY2tlciJ9LHsiYXR0cmlidXRlcyI6eyJtb250aHMiOlswLDIsNCw2LDgsMTBdfSwiaWQiOiIxNTA5IiwidHlwZSI6Ik1vbnRoc1RpY2tlciJ9LHsiYXR0cmlidXRlcyI6eyJjYWxsYmFjayI6bnVsbH0sImlkIjoiMTQ1MiIsInR5cGUiOiJEYXRhUmFuZ2UxZCJ9LHsiYXR0cmlidXRlcyI6e30sImlkIjoiMTQ3MCIsInR5cGUiOiJQYW5Ub29sIn0seyJhdHRyaWJ1dGVzIjp7InNvdXJjZSI6eyJpZCI6IjE0NzkiLCJ0eXBlIjoiQ29sdW1uRGF0YVNvdXJjZSJ9fSwiaWQiOiIxNDgzIiwidHlwZSI6IkNEU1ZpZXcifSx7ImF0dHJpYnV0ZXMiOnsib3ZlcmxheSI6eyJpZCI6IjE0NzQiLCJ0eXBlIjoiQm94QW5ub3RhdGlvbiJ9fSwiaWQiOiIxNDcxIiwidHlwZSI6IkJveFpvb21Ub29sIn0seyJhdHRyaWJ1dGVzIjp7ImRhdGFfc291cmNlIjp7ImlkIjoiMTQ3OSIsInR5cGUiOiJDb2x1bW5EYXRhU291cmNlIn0sImdseXBoIjp7ImlkIjoiMTQ4MCIsInR5cGUiOiJMaW5lIn0sImhvdmVyX2dseXBoIjpudWxsLCJtdXRlZF9nbHlwaCI6bnVsbCwibm9uc2VsZWN0aW9uX2dseXBoIjp7ImlkIjoiMTQ4MSIsInR5cGUiOiJMaW5lIn0sInNlbGVjdGlvbl9nbHlwaCI6bnVsbCwidmlldyI6eyJpZCI6IjE0ODMiLCJ0eXBlIjoiQ0RTVmlldyJ9fSwiaWQiOiIxNDgyIiwidHlwZSI6IkdseXBoUmVuZGVyZXIifSx7ImF0dHJpYnV0ZXMiOnsiZGF5cyI6WzEsMiwzLDQsNSw2LDcsOCw5LDEwLDExLDEyLDEzLDE0LDE1LDE2LDE3LDE4LDE5LDIwLDIxLDIyLDIzLDI0LDI1LDI2LDI3LDI4LDI5LDMwLDMxXX0sImlkIjoiMTUwNCIsInR5cGUiOiJEYXlzVGlja2VyIn0seyJhdHRyaWJ1dGVzIjp7fSwiaWQiOiIxNTE2IiwidHlwZSI6IlNlbGVjdGlvbiJ9LHsiYXR0cmlidXRlcyI6eyJsYWJlbCI6eyJ2YWx1ZSI6Ik9ic2VydmF0aW9ucyJ9LCJyZW5kZXJlcnMiOlt7ImlkIjoiMTQ4OSIsInR5cGUiOiJHbHlwaFJlbmRlcmVyIn1dfSwiaWQiOiIxNDk1IiwidHlwZSI6IkxlZ2VuZEl0ZW0ifSx7ImF0dHJpYnV0ZXMiOnt9LCJpZCI6IjE0NTYiLCJ0eXBlIjoiTGluZWFyU2NhbGUifSx7ImF0dHJpYnV0ZXMiOnsicGxvdCI6eyJpZCI6IjE0NTAiLCJzdWJ0eXBlIjoiRmlndXJlIiwidHlwZSI6IlBsb3QifSwidGlja2VyIjp7ImlkIjoiMTQ2MSIsInR5cGUiOiJEYXRldGltZVRpY2tlciJ9fSwiaWQiOiIxNDY0IiwidHlwZSI6IkdyaWQifSx7ImF0dHJpYnV0ZXMiOnt9LCJpZCI6IjE0NzIiLCJ0eXBlIjoiUmVzZXRUb29sIn0seyJhdHRyaWJ1dGVzIjp7ImRheXMiOlsxLDQsNywxMCwxMywxNiwxOSwyMiwyNSwyOF19LCJpZCI6IjE1MDUiLCJ0eXBlIjoiRGF5c1RpY2tlciJ9LHsiYXR0cmlidXRlcyI6eyJheGlzX2xhYmVsIjoiV2F0ZXIgSGVpZ2h0IChtKSIsImZvcm1hdHRlciI6eyJpZCI6IjE1MDAiLCJ0eXBlIjoiQmFzaWNUaWNrRm9ybWF0dGVyIn0sInBsb3QiOnsiaWQiOiIxNDUwIiwic3VidHlwZSI6IkZpZ3VyZSIsInR5cGUiOiJQbG90In0sInRpY2tlciI6eyJpZCI6IjE0NjYiLCJ0eXBlIjoiQmFzaWNUaWNrZXIifX0sImlkIjoiMTQ2NSIsInR5cGUiOiJMaW5lYXJBeGlzIn1dLCJyb290X2lkcyI6WyIxNDUwIl19LCJ0aXRsZSI6IkJva2VoIEFwcGxpY2F0aW9uIiwidmVyc2lvbiI6IjEuMC4xIn19CiAgICAgICAgPC9zY3JpcHQ+CiAgICAgICAgPHNjcmlwdCB0eXBlPSJ0ZXh0L2phdmFzY3JpcHQiPgogICAgICAgICAgKGZ1bmN0aW9uKCkgewogICAgICAgICAgICB2YXIgZm4gPSBmdW5jdGlvbigpIHsKICAgICAgICAgICAgICBCb2tlaC5zYWZlbHkoZnVuY3Rpb24oKSB7CiAgICAgICAgICAgICAgICAoZnVuY3Rpb24ocm9vdCkgewogICAgICAgICAgICAgICAgICBmdW5jdGlvbiBlbWJlZF9kb2N1bWVudChyb290KSB7CiAgICAgICAgICAgICAgICAgICAgCiAgICAgICAgICAgICAgICAgIHZhciBkb2NzX2pzb24gPSBkb2N1bWVudC5nZXRFbGVtZW50QnlJZCgnMTY3MicpLnRleHRDb250ZW50OwogICAgICAgICAgICAgICAgICB2YXIgcmVuZGVyX2l0ZW1zID0gW3siZG9jaWQiOiI1NmVhMjg4Ni1lMzhlLTQ2MDQtYTZlOC1iMjA4NzgxNjNlZTAiLCJyb290cyI6eyIxNDUwIjoiMTliNGQ5YTAtMTlkNS00ZTI4LWE1OTUtMjE3MjExODBjNDE4In19XTsKICAgICAgICAgICAgICAgICAgcm9vdC5Cb2tlaC5lbWJlZC5lbWJlZF9pdGVtcyhkb2NzX2pzb24sIHJlbmRlcl9pdGVtcyk7CiAgICAgICAgICAgICAgICAKICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgICBpZiAocm9vdC5Cb2tlaCAhPT0gdW5kZWZpbmVkKSB7CiAgICAgICAgICAgICAgICAgICAgZW1iZWRfZG9jdW1lbnQocm9vdCk7CiAgICAgICAgICAgICAgICAgIH0gZWxzZSB7CiAgICAgICAgICAgICAgICAgICAgdmFyIGF0dGVtcHRzID0gMDsKICAgICAgICAgICAgICAgICAgICB2YXIgdGltZXIgPSBzZXRJbnRlcnZhbChmdW5jdGlvbihyb290KSB7CiAgICAgICAgICAgICAgICAgICAgICBpZiAocm9vdC5Cb2tlaCAhPT0gdW5kZWZpbmVkKSB7CiAgICAgICAgICAgICAgICAgICAgICAgIGVtYmVkX2RvY3VtZW50KHJvb3QpOwogICAgICAgICAgICAgICAgICAgICAgICBjbGVhckludGVydmFsKHRpbWVyKTsKICAgICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICAgICAgIGF0dGVtcHRzKys7CiAgICAgICAgICAgICAgICAgICAgICBpZiAoYXR0ZW1wdHMgPiAxMDApIHsKICAgICAgICAgICAgICAgICAgICAgICAgY29uc29sZS5sb2coIkJva2VoOiBFUlJPUjogVW5hYmxlIHRvIHJ1biBCb2tlaEpTIGNvZGUgYmVjYXVzZSBCb2tlaEpTIGxpYnJhcnkgaXMgbWlzc2luZyIpOwogICAgICAgICAgICAgICAgICAgICAgICBjbGVhckludGVydmFsKHRpbWVyKTsKICAgICAgICAgICAgICAgICAgICAgIH0KICAgICAgICAgICAgICAgICAgICB9LCAxMCwgcm9vdCkKICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgfSkod2luZG93KTsKICAgICAgICAgICAgICB9KTsKICAgICAgICAgICAgfTsKICAgICAgICAgICAgaWYgKGRvY3VtZW50LnJlYWR5U3RhdGUgIT0gImxvYWRpbmciKSBmbigpOwogICAgICAgICAgICBlbHNlIGRvY3VtZW50LmFkZEV2ZW50TGlzdGVuZXIoIkRPTUNvbnRlbnRMb2FkZWQiLCBmbik7CiAgICAgICAgICB9KSgpOwogICAgICAgIDwvc2NyaXB0PgogICAgCiAgPC9ib2R5PgogIAo8L2h0bWw+&quot; width=&quot;790&quot; style=&quot;border:none !important;&quot; height=&quot;330&quot;></iframe>`)[0];
                popup_820c7aaa26f74cf388719d06cad72c8c.setContent(i_frame_2e07c1aa64734d4c8b1fc5a1d0196e6c);


            marker_7796e1a5ae3648308d75fde74420f82d.bindPopup(popup_820c7aaa26f74cf388719d06cad72c8c)
            ;




        var marker_e2a117ec22924bef9b4ef7842e0dead0 = L.marker(
            [41.7043, -71.1641],
            {
                icon: new L.Icon.Default()
                }
            ).addTo(map_54cb488c2c244a8daafaa65ad6b9b54b);



                var icon_37b9c695b73a4dd888cd38940424bf6b = L.AwesomeMarkers.icon({
                    icon: 'stats',
                    iconColor: 'white',
                    markerColor: 'green',
                    prefix: 'glyphicon',
                    extraClasses: 'fa-rotate-0'
                    });
                marker_e2a117ec22924bef9b4ef7842e0dead0.setIcon(icon_37b9c695b73a4dd888cd38940424bf6b);


            var popup_65c7f72788754a0ba8fe1570f9117af0 = L.popup({maxWidth: '2650'

            });


                var i_frame_3e0f8909d1b7489e86f7fe897ea6a3b8 = $(`<iframe src=&quot;data:text/html;charset=utf-8;base64,CiAgICAKCgoKPCFET0NUWVBFIGh0bWw+CjxodG1sIGxhbmc9ImVuIj4KICAKICA8aGVhZD4KICAgIAogICAgICA8bWV0YSBjaGFyc2V0PSJ1dGYtOCI+CiAgICAgIDx0aXRsZT44NDQ3Mzg2PC90aXRsZT4KICAgICAgCiAgICAgIAogICAgICAgIAogICAgICAgICAgCiAgICAgICAgPGxpbmsgcmVsPSJzdHlsZXNoZWV0IiBocmVmPSJodHRwczovL2Nkbi5weWRhdGEub3JnL2Jva2VoL3JlbGVhc2UvYm9rZWgtMS4wLjEubWluLmNzcyIgdHlwZT0idGV4dC9jc3MiIC8+CiAgICAgICAgCiAgICAgICAgCiAgICAgICAgICAKICAgICAgICA8c2NyaXB0IHR5cGU9InRleHQvamF2YXNjcmlwdCIgc3JjPSJodHRwczovL2Nkbi5weWRhdGEub3JnL2Jva2VoL3JlbGVhc2UvYm9rZWgtMS4wLjEubWluLmpzIj48L3NjcmlwdD4KICAgICAgICA8c2NyaXB0IHR5cGU9InRleHQvamF2YXNjcmlwdCI+CiAgICAgICAgICAgIEJva2VoLnNldF9sb2dfbGV2ZWwoImluZm8iKTsKICAgICAgICA8L3NjcmlwdD4KICAgICAgICAKICAgICAgCiAgICAgIAogICAgCiAgPC9oZWFkPgogIAogIAogIDxib2R5PgogICAgCiAgICAgIAogICAgICAgIAogICAgICAgICAgCiAgICAgICAgICAKICAgICAgICAgICAgCiAgICAgICAgICAgICAgPGRpdiBjbGFzcz0iYmstcm9vdCIgaWQ9ImYzNjZiNDExLTVmODctNDI5Yy05OWY5LWQ0MjVmYWUxNzE0NSI+PC9kaXY+CiAgICAgICAgICAgIAogICAgICAgICAgCiAgICAgICAgCiAgICAgIAogICAgICAKICAgICAgICA8c2NyaXB0IHR5cGU9ImFwcGxpY2F0aW9uL2pzb24iIGlkPSIxODcyIj4KICAgICAgICAgIHsiZTUxYTg5MDgtZWFkYS00NjM0LWExOGItNzk1NzczMjcyNWU5Ijp7InJvb3RzIjp7InJlZmVyZW5jZXMiOlt7ImF0dHJpYnV0ZXMiOnsiYmFzZSI6NjAsIm1hbnRpc3NhcyI6WzEsMiw1LDEwLDE1LDIwLDMwXSwibWF4X2ludGVydmFsIjoxODAwMDAwLjAsIm1pbl9pbnRlcnZhbCI6MTAwMC4wLCJudW1fbWlub3JfdGlja3MiOjB9LCJpZCI6IjE3MTgiLCJ0eXBlIjoiQWRhcHRpdmVUaWNrZXIifSx7ImF0dHJpYnV0ZXMiOnt9LCJpZCI6IjE3MTQiLCJ0eXBlIjoiRGF0ZXRpbWVUaWNrRm9ybWF0dGVyIn0seyJhdHRyaWJ1dGVzIjp7ImNsaWNrX3BvbGljeSI6Im11dGUiLCJpdGVtcyI6W3siaWQiOiIxNzExIiwidHlwZSI6IkxlZ2VuZEl0ZW0ifV0sImxvY2F0aW9uIjpbMCw2MF0sInBsb3QiOnsiaWQiOiIxNjc0Iiwic3VidHlwZSI6IkZpZ3VyZSIsInR5cGUiOiJQbG90In19LCJpZCI6IjE3MTAiLCJ0eXBlIjoiTGVnZW5kIn0seyJhdHRyaWJ1dGVzIjp7ImRheXMiOlsxLDQsNywxMCwxMywxNiwxOSwyMiwyNSwyOF19LCJpZCI6IjE3MjEiLCJ0eXBlIjoiRGF5c1RpY2tlciJ9LHsiYXR0cmlidXRlcyI6eyJvdmVybGF5Ijp7ImlkIjoiMTY5OCIsInR5cGUiOiJCb3hBbm5vdGF0aW9uIn19LCJpZCI6IjE2OTUiLCJ0eXBlIjoiQm94Wm9vbVRvb2wifSx7ImF0dHJpYnV0ZXMiOnsic291cmNlIjp7ImlkIjoiMTcwMyIsInR5cGUiOiJDb2x1bW5EYXRhU291cmNlIn19LCJpZCI6IjE3MDciLCJ0eXBlIjoiQ0RTVmlldyJ9LHsiYXR0cmlidXRlcyI6eyJiZWxvdyI6W3siaWQiOiIxNjg0IiwidHlwZSI6IkRhdGV0aW1lQXhpcyJ9XSwibGVmdCI6W3siaWQiOiIxNjg5IiwidHlwZSI6IkxpbmVhckF4aXMifV0sInBsb3RfaGVpZ2h0IjoyNTAsInBsb3Rfd2lkdGgiOjc1MCwicmVuZGVyZXJzIjpbeyJpZCI6IjE2ODQiLCJ0eXBlIjoiRGF0ZXRpbWVBeGlzIn0seyJpZCI6IjE2ODgiLCJ0eXBlIjoiR3JpZCJ9LHsiaWQiOiIxNjg5IiwidHlwZSI6IkxpbmVhckF4aXMifSx7ImlkIjoiMTY5MyIsInR5cGUiOiJHcmlkIn0seyJpZCI6IjE2OTgiLCJ0eXBlIjoiQm94QW5ub3RhdGlvbiJ9LHsiaWQiOiIxNzA2IiwidHlwZSI6IkdseXBoUmVuZGVyZXIifSx7ImlkIjoiMTcxMCIsInR5cGUiOiJMZWdlbmQifV0sInJpZ2h0IjpbeyJpZCI6IjE3MTAiLCJ0eXBlIjoiTGVnZW5kIn1dLCJ0aXRsZSI6eyJpZCI6IjE2NzMiLCJ0eXBlIjoiVGl0bGUifSwidG9vbGJhciI6eyJpZCI6IjE2OTciLCJ0eXBlIjoiVG9vbGJhciJ9LCJ0b29sYmFyX2xvY2F0aW9uIjoiYWJvdmUiLCJ4X3JhbmdlIjp7ImlkIjoiMTY3NiIsInR5cGUiOiJEYXRhUmFuZ2UxZCJ9LCJ4X3NjYWxlIjp7ImlkIjoiMTY4MCIsInR5cGUiOiJMaW5lYXJTY2FsZSJ9LCJ5X3JhbmdlIjp7ImlkIjoiMTY3OCIsInR5cGUiOiJEYXRhUmFuZ2UxZCJ9LCJ5X3NjYWxlIjp7ImlkIjoiMTY4MiIsInR5cGUiOiJMaW5lYXJTY2FsZSJ9fSwiaWQiOiIxNjc0Iiwic3VidHlwZSI6IkZpZ3VyZSIsInR5cGUiOiJQbG90In0seyJhdHRyaWJ1dGVzIjp7fSwiaWQiOiIxNjgyIiwidHlwZSI6IkxpbmVhclNjYWxlIn0seyJhdHRyaWJ1dGVzIjp7fSwiaWQiOiIxNzMwIiwidHlwZSI6IlNlbGVjdGlvbiJ9LHsiYXR0cmlidXRlcyI6eyJheGlzX2xhYmVsIjoiRGF0ZS90aW1lIiwiZm9ybWF0dGVyIjp7ImlkIjoiMTcxNCIsInR5cGUiOiJEYXRldGltZVRpY2tGb3JtYXR0ZXIifSwicGxvdCI6eyJpZCI6IjE2NzQiLCJzdWJ0eXBlIjoiRmlndXJlIiwidHlwZSI6IlBsb3QifSwidGlja2VyIjp7ImlkIjoiMTY4NSIsInR5cGUiOiJEYXRldGltZVRpY2tlciJ9fSwiaWQiOiIxNjg0IiwidHlwZSI6IkRhdGV0aW1lQXhpcyJ9LHsiYXR0cmlidXRlcyI6e30sImlkIjoiMTY5NiIsInR5cGUiOiJSZXNldFRvb2wifSx7ImF0dHJpYnV0ZXMiOnsiZGltZW5zaW9uIjoxLCJwbG90Ijp7ImlkIjoiMTY3NCIsInN1YnR5cGUiOiJGaWd1cmUiLCJ0eXBlIjoiUGxvdCJ9LCJ0aWNrZXIiOnsiaWQiOiIxNjkwIiwidHlwZSI6IkJhc2ljVGlja2VyIn19LCJpZCI6IjE2OTMiLCJ0eXBlIjoiR3JpZCJ9LHsiYXR0cmlidXRlcyI6eyJsaW5lX2NhcCI6InJvdW5kIiwibGluZV9jb2xvciI6ImNyaW1zb24iLCJsaW5lX2pvaW4iOiJyb3VuZCIsImxpbmVfd2lkdGgiOjUsIngiOnsiZmllbGQiOiJ4In0sInkiOnsiZmllbGQiOiJ5In19LCJpZCI6IjE3MDQiLCJ0eXBlIjoiTGluZSJ9LHsiYXR0cmlidXRlcyI6eyJtb250aHMiOlswLDEsMiwzLDQsNSw2LDcsOCw5LDEwLDExXX0sImlkIjoiMTcyNCIsInR5cGUiOiJNb250aHNUaWNrZXIifSx7ImF0dHJpYnV0ZXMiOnsibW9udGhzIjpbMCw2XX0sImlkIjoiMTcyNyIsInR5cGUiOiJNb250aHNUaWNrZXIifSx7ImF0dHJpYnV0ZXMiOnsibnVtX21pbm9yX3RpY2tzIjo1LCJ0aWNrZXJzIjpbeyJpZCI6IjE3MTciLCJ0eXBlIjoiQWRhcHRpdmVUaWNrZXIifSx7ImlkIjoiMTcxOCIsInR5cGUiOiJBZGFwdGl2ZVRpY2tlciJ9LHsiaWQiOiIxNzE5IiwidHlwZSI6IkFkYXB0aXZlVGlja2VyIn0seyJpZCI6IjE3MjAiLCJ0eXBlIjoiRGF5c1RpY2tlciJ9LHsiaWQiOiIxNzIxIiwidHlwZSI6IkRheXNUaWNrZXIifSx7ImlkIjoiMTcyMiIsInR5cGUiOiJEYXlzVGlja2VyIn0seyJpZCI6IjE3MjMiLCJ0eXBlIjoiRGF5c1RpY2tlciJ9LHsiaWQiOiIxNzI0IiwidHlwZSI6Ik1vbnRoc1RpY2tlciJ9LHsiaWQiOiIxNzI1IiwidHlwZSI6Ik1vbnRoc1RpY2tlciJ9LHsiaWQiOiIxNzI2IiwidHlwZSI6Ik1vbnRoc1RpY2tlciJ9LHsiaWQiOiIxNzI3IiwidHlwZSI6Ik1vbnRoc1RpY2tlciJ9LHsiaWQiOiIxNzI4IiwidHlwZSI6IlllYXJzVGlja2VyIn1dfSwiaWQiOiIxNjg1IiwidHlwZSI6IkRhdGV0aW1lVGlja2VyIn0seyJhdHRyaWJ1dGVzIjp7ImRheXMiOlsxLDIsMyw0LDUsNiw3LDgsOSwxMCwxMSwxMiwxMywxNCwxNSwxNiwxNywxOCwxOSwyMCwyMSwyMiwyMywyNCwyNSwyNiwyNywyOCwyOSwzMCwzMV19LCJpZCI6IjE3MjAiLCJ0eXBlIjoiRGF5c1RpY2tlciJ9LHsiYXR0cmlidXRlcyI6eyJsYWJlbCI6eyJ2YWx1ZSI6Ik9ic2VydmF0aW9ucyJ9LCJyZW5kZXJlcnMiOlt7ImlkIjoiMTcwNiIsInR5cGUiOiJHbHlwaFJlbmRlcmVyIn1dfSwiaWQiOiIxNzExIiwidHlwZSI6IkxlZ2VuZEl0ZW0ifSx7ImF0dHJpYnV0ZXMiOnt9LCJpZCI6IjE2OTQiLCJ0eXBlIjoiUGFuVG9vbCJ9LHsiYXR0cmlidXRlcyI6eyJwbG90Ijp7ImlkIjoiMTY3NCIsInN1YnR5cGUiOiJGaWd1cmUiLCJ0eXBlIjoiUGxvdCJ9LCJ0aWNrZXIiOnsiaWQiOiIxNjg1IiwidHlwZSI6IkRhdGV0aW1lVGlja2VyIn19LCJpZCI6IjE2ODgiLCJ0eXBlIjoiR3JpZCJ9LHsiYXR0cmlidXRlcyI6eyJtYW50aXNzYXMiOlsxLDIsNV0sIm1heF9pbnRlcnZhbCI6NTAwLjAsIm51bV9taW5vcl90aWNrcyI6MH0sImlkIjoiMTcxNyIsInR5cGUiOiJBZGFwdGl2ZVRpY2tlciJ9LHsiYXR0cmlidXRlcyI6eyJtb250aHMiOlswLDIsNCw2LDgsMTBdfSwiaWQiOiIxNzI1IiwidHlwZSI6Ik1vbnRoc1RpY2tlciJ9LHsiYXR0cmlidXRlcyI6e30sImlkIjoiMTcxNiIsInR5cGUiOiJCYXNpY1RpY2tGb3JtYXR0ZXIifSx7ImF0dHJpYnV0ZXMiOnsiY2FsbGJhY2siOm51bGx9LCJpZCI6IjE2NzYiLCJ0eXBlIjoiRGF0YVJhbmdlMWQifSx7ImF0dHJpYnV0ZXMiOnsibW9udGhzIjpbMCw0LDhdfSwiaWQiOiIxNzI2IiwidHlwZSI6Ik1vbnRoc1RpY2tlciJ9LHsiYXR0cmlidXRlcyI6e30sImlkIjoiMTY5MCIsInR5cGUiOiJCYXNpY1RpY2tlciJ9LHsiYXR0cmlidXRlcyI6e30sImlkIjoiMTY4MCIsInR5cGUiOiJMaW5lYXJTY2FsZSJ9LHsiYXR0cmlidXRlcyI6eyJsaW5lX2FscGhhIjowLjEsImxpbmVfY2FwIjoicm91bmQiLCJsaW5lX2NvbG9yIjoiIzFmNzdiNCIsImxpbmVfam9pbiI6InJvdW5kIiwibGluZV93aWR0aCI6NSwieCI6eyJmaWVsZCI6IngifSwieSI6eyJmaWVsZCI6InkifX0sImlkIjoiMTcwNSIsInR5cGUiOiJMaW5lIn0seyJhdHRyaWJ1dGVzIjp7ImRheXMiOlsxLDE1XX0sImlkIjoiMTcyMyIsInR5cGUiOiJEYXlzVGlja2VyIn0seyJhdHRyaWJ1dGVzIjp7fSwiaWQiOiIxNzMxIiwidHlwZSI6IlVuaW9uUmVuZGVyZXJzIn0seyJhdHRyaWJ1dGVzIjp7ImNhbGxiYWNrIjpudWxsLCJyZW5kZXJlcnMiOlt7ImlkIjoiMTcwNiIsInR5cGUiOiJHbHlwaFJlbmRlcmVyIn1dLCJ0b29sdGlwcyI6W1siTmFtZSIsIk9ic2VydmF0aW9ucyJdLFsiQmlhcyIsIk5BIl0sWyJTa2lsbCIsIk5BIl1dfSwiaWQiOiIxNzA4IiwidHlwZSI6IkhvdmVyVG9vbCJ9LHsiYXR0cmlidXRlcyI6eyJhY3RpdmVfZHJhZyI6ImF1dG8iLCJhY3RpdmVfaW5zcGVjdCI6ImF1dG8iLCJhY3RpdmVfbXVsdGkiOm51bGwsImFjdGl2ZV9zY3JvbGwiOiJhdXRvIiwiYWN0aXZlX3RhcCI6ImF1dG8iLCJ0b29scyI6W3siaWQiOiIxNjk0IiwidHlwZSI6IlBhblRvb2wifSx7ImlkIjoiMTY5NSIsInR5cGUiOiJCb3hab29tVG9vbCJ9LHsiaWQiOiIxNjk2IiwidHlwZSI6IlJlc2V0VG9vbCJ9LHsiaWQiOiIxNzA4IiwidHlwZSI6IkhvdmVyVG9vbCJ9XX0sImlkIjoiMTY5NyIsInR5cGUiOiJUb29sYmFyIn0seyJhdHRyaWJ1dGVzIjp7ImJhc2UiOjI0LCJtYW50aXNzYXMiOlsxLDIsNCw2LDgsMTJdLCJtYXhfaW50ZXJ2YWwiOjQzMjAwMDAwLjAsIm1pbl9pbnRlcnZhbCI6MzYwMDAwMC4wLCJudW1fbWlub3JfdGlja3MiOjB9LCJpZCI6IjE3MTkiLCJ0eXBlIjoiQWRhcHRpdmVUaWNrZXIifSx7ImF0dHJpYnV0ZXMiOnt9LCJpZCI6IjE3MjgiLCJ0eXBlIjoiWWVhcnNUaWNrZXIifSx7ImF0dHJpYnV0ZXMiOnsiY2FsbGJhY2siOm51bGx9LCJpZCI6IjE2NzgiLCJ0eXBlIjoiRGF0YVJhbmdlMWQifSx7ImF0dHJpYnV0ZXMiOnsiYXhpc19sYWJlbCI6IldhdGVyIEhlaWdodCAobSkiLCJmb3JtYXR0ZXIiOnsiaWQiOiIxNzE2IiwidHlwZSI6IkJhc2ljVGlja0Zvcm1hdHRlciJ9LCJwbG90Ijp7ImlkIjoiMTY3NCIsInN1YnR5cGUiOiJGaWd1cmUiLCJ0eXBlIjoiUGxvdCJ9LCJ0aWNrZXIiOnsiaWQiOiIxNjkwIiwidHlwZSI6IkJhc2ljVGlja2VyIn19LCJpZCI6IjE2ODkiLCJ0eXBlIjoiTGluZWFyQXhpcyJ9LHsiYXR0cmlidXRlcyI6eyJwbG90IjpudWxsLCJ0ZXh0IjoiODQ0NzM4NiJ9LCJpZCI6IjE2NzMiLCJ0eXBlIjoiVGl0bGUifSx7ImF0dHJpYnV0ZXMiOnsiYm90dG9tX3VuaXRzIjoic2NyZWVuIiwiZmlsbF9hbHBoYSI6eyJ2YWx1ZSI6MC41fSwiZmlsbF9jb2xvciI6eyJ2YWx1ZSI6ImxpZ2h0Z3JleSJ9LCJsZWZ0X3VuaXRzIjoic2NyZWVuIiwibGV2ZWwiOiJvdmVybGF5IiwibGluZV9hbHBoYSI6eyJ2YWx1ZSI6MS4wfSwibGluZV9jb2xvciI6eyJ2YWx1ZSI6ImJsYWNrIn0sImxpbmVfZGFzaCI6WzQsNF0sImxpbmVfd2lkdGgiOnsidmFsdWUiOjJ9LCJwbG90IjpudWxsLCJyZW5kZXJfbW9kZSI6ImNzcyIsInJpZ2h0X3VuaXRzIjoic2NyZWVuIiwidG9wX3VuaXRzIjoic2NyZWVuIn0sImlkIjoiMTY5OCIsInR5cGUiOiJCb3hBbm5vdGF0aW9uIn0seyJhdHRyaWJ1dGVzIjp7ImRhdGFfc291cmNlIjp7ImlkIjoiMTcwMyIsInR5cGUiOiJDb2x1bW5EYXRhU291cmNlIn0sImdseXBoIjp7ImlkIjoiMTcwNCIsInR5cGUiOiJMaW5lIn0sImhvdmVyX2dseXBoIjpudWxsLCJtdXRlZF9nbHlwaCI6bnVsbCwibm9uc2VsZWN0aW9uX2dseXBoIjp7ImlkIjoiMTcwNSIsInR5cGUiOiJMaW5lIn0sInNlbGVjdGlvbl9nbHlwaCI6bnVsbCwidmlldyI6eyJpZCI6IjE3MDciLCJ0eXBlIjoiQ0RTVmlldyJ9fSwiaWQiOiIxNzA2IiwidHlwZSI6IkdseXBoUmVuZGVyZXIifSx7ImF0dHJpYnV0ZXMiOnsiY2FsbGJhY2siOm51bGwsImRhdGEiOnsieCI6eyJfX25kYXJyYXlfXyI6IkFBQ0FWcHNkZGtJQUFHakZuaDEyUWdBQVVEU2lIWFpDQUFBNG82VWRka0lBQUNBU3FSMTJRZ0FBQ0lHc0hYWkNBQUR3NzY4ZGRrSUFBTmhlc3gxMlFnQUF3TTIySFhaQ0FBQ29QTG9kZGtJQUFKQ3J2UjEyUWdBQWVCckJIWFpDQUFCZ2ljUWRka0lBQUVqNHh4MTJRZ0FBTUdmTEhYWkNBQUFZMXM0ZGRrSUFBQUJGMGgxMlFnQUE2TFBWSFhaQ0FBRFFJdGtkZGtJQUFMaVIzQjEyUWdBQW9BRGdIWFpDQUFDSWIrTWRka0lBQUhEZTVoMTJRZ0FBV0UzcUhYWkNBQUJBdk8wZGRrSUFBQ2dyOFIxMlFnQUFFSnIwSFhaQ0FBRDRDUGdkZGtJQUFPQjMreDEyUWdBQXlPYitIWFpDQUFDd1ZRSWVka0lBQUpqRUJSNTJRZ0FBZ0RNSkhuWkNBQUJvb2d3ZWRrSUFBRkFSRUI1MlFnQUFPSUFUSG5aQ0FBQWc3eFllZGtJQUFBaGVHaDUyUWdBQThNd2RIblpDQUFEWU95RWVka0lBQU1DcUpCNTJRZ0FBcUJrb0huWkNBQUNRaUNzZWRrSUFBSGozTGg1MlFnQUFZR1l5SG5aQ0FBQkkxVFVlZGtJQUFEQkVPUjUyUWdBQUdMTThIblpDQUFBQUlrQWVka0lBQU9pUVF4NTJRZ0FBMFA5R0huWkNBQUM0YmtvZWRrSUFBS0RkVFI1MlFnQUFpRXhSSG5aQ0FBQnd1MVFlZGtJQUFGZ3FXQjUyUWdBQVFKbGJIblpDQUFBb0NGOGVka0lBQUJCM1loNTJRZ0FBK09WbEhuWkNBQURnVkdrZWRrSUFBTWpEYkI1MlFnQUFzREp3SG5aQ0FBQ1lvWE1lZGtJQUFJQVFkeDUyUWdBQWFIOTZIblpDQUFCUTduMGVka0lBQURoZGdSNTJRZ0FBSU15RUhuWkNBQUFJTzRnZWRrSUFBUENwaXg1MlFnQUEyQmlQSG5aQ0FBREFoNUllZGtJQUFLajJsUjUyUWdBQWtHV1pIblpDQUFCNDFKd2Vka0lBQUdCRG9CNTJRZ0FBU0xLakhuWkNBQUF3SWFjZWRrSUFBQmlRcWg1MlFnQUFBUCt0SG5aQ0FBRG9iYkVlZGtJQUFORGN0QjUyUWdBQXVFdTRIblpDQUFDZ3Vyc2Vka0lBQUlncHZ4NTJRZ0FBY0pqQ0huWkNBQUJZQjhZZWRrSUFBRUIyeVI1MlFnQUFLT1hNSG5aQ0FBQVFWTkFlZGtJQUFQakMweDUyUWdBQTRESFhIblpDQUFESW9Ob2Vka0lBQUxBUDNoNTJRZ0FBbUg3aEhuWkNBQUNBN2VRZWRrSUFBR2hjNkI1MlFnQUFVTXZySG5aQ0FBQTRPdThlZGtJQUFDQ3A4aDUyUWdBQUNCajJIblpDQUFEd2h2a2Vka0lBQU5qMS9CNTJRZ0FBd0dRQUgzWkNBQUNvMHdNZmRrSUFBSkJDQng5MlFnQUFlTEVLSDNaQ0FBQmdJQTRmZGtJQUFFaVBFUjkyUWdBQU1QNFVIM1pDQUFBWWJSZ2Zka0lBQUFEY0d4OTJRZ0FBNkVvZkgzWkNBQURRdVNJZmRrSUFBTGdvSmg5MlFnQUFvSmNwSDNaQ0FBQ0lCaTBmZGtJQUFIQjFNQjkyUWdBQVdPUXpIM1pDQUFCQVV6Y2Zka0k9IiwiZHR5cGUiOiJmbG9hdDY0Iiwic2hhcGUiOlsxMjFdfSwieSI6eyJfX25kYXJyYXlfXyI6InZIU1RHQVJXOUQ5dDUvdXA4ZEx0UDZKRnR2UDkxT0EvYVpIdGZEODF2ai84cWZIU1RXTEF2NFBBeXFGRnRyTy9WZzR0c3Azdnh6OVBqWmR1RW9QWVA4bDJ2cDhhTCtVL0pRYUJsVU9MOEQ4ajIvbCthcnoyUHkyeW5lK254dmsvcEhBOUN0ZWorRC82Zm1xOGRKUDBQMEZnNWRBaTIrMC8vdFI0NlNZeDREL2IrWDVxdkhURFA0eHM1L3VwOGFLL1BOOVBqWmR1Z2o4MlhycEpEQUxMUDRHVlE0dHM1OXMvSEZwa085OVA2VCt1UitGNkZLN3pQMXBrTzk5UGpmay9ONGxCWU9YUStqL1A5MVBqcFp2MlA0WHJVYmdlaGU4L1RtSVFXRG0wNEQ4UldEbTB5SGErUC9wK2FyeDBrN2kvK241cXZIU1RtTCtTN1h3L05WN0tQK0Y2Rks1SDRkby90dlA5MUhqcDVqOFVya2ZoZWhUeVAvaFQ0NldieFBnL2VPa21NUWlzL0Qvbys2bngwazM4UDhaTE40bEJZUGMvajhMMUtGeVA3ai9SSXR2NWZtcmNQK2ttTVFpc0hLby8vS254MGsxaVVMOHBYSS9DOVNqTVAyRGwwQ0xiK2RZL2FyeDBreGdFM2ovTnpNek16TXpvUHdhQmxVT0xiUFUvSWJCeWFKSHQvRDhZQkZZT0xiTDlQN1RJZHI2Zkd2ay9Cb0dWUTR0czhUOVlPYlRJZHI3alB4U3VSK0Y2Rk00LzIvbCthcngwa3o5VTQ2V2J4Q0N3UDRsQllPWFFJc3MvM1NRR2daVkQyei9EOVNoY2o4THBQNC9DOVNoY2ovUS9QUXJYbzNBOS9EL0F5cUZGdHZQL1B3QUFBQUFBQVA0L1ZPT2xtOFFnK0QvTnpNek16TXp3UDI4U2c4REtvZUUvc1hKb2tlMTgxejl0NS91cDhkTE5QMUNObDI0U2c4QS9LVnlQd3ZVb3ZEOGxCb0dWUTR2VVAzRTlDdGVqY09rLzgvM1VlT2ttOVQvZlQ0MlhiaEw3UDJxOGRKTVlCUHcvajhMMUtGeVArRDhwWEkvQzlTajBQNTN2cDhaTE4rMC9xTVpMTjRsQjVEKzdTUXdDSzRmV1AwVzI4LzNVZU1rLzBTTGIrWDVxekQ4cFhJL0M5U2pnUDlSNDZTWXhDT3cvbmUrbnhrczM5ejhSV0RtMHlIWUFRUExTVFdJUVdBSkFXRG0weUhhK0FFQ3luZStueGt2NVA4M016TXpNelBBL0FpdUhGdG5PNHo4UldEbTB5SGJlUHdyWG8zQTlDdDgvc3AzdnA4Wkw0eitGNjFHNEhvWG5QK09sbThRZ3NQQS9qWmR1RW9QQTlqODJYcnBKREFML1B6VmV1a2tNQWdKQXdjcWhSYmJ6QVVCdDUvdXA4ZEw5UDFUanBadkVJUFkvZEpNWUJGWU83VC9FSUxCeWFKSGhQeVVHZ1pWRGk5dy9EaTJ5bmUrbjNqOS9hcngwa3hqa1A0R1ZRNHRzNStzL1B6VmV1a2tNOUQ5RWkyem4rNm43UDhRZ3NISm9rUUJBQ0t3Y1dtUTdBVURsMENMYitYNytQOGwydnA4YUwvYy9nOERLb1VXMjd6L3ZwOFpMTjRubFB6MEsxNk53UGVJL29Cb3YzU1FHNVQ5N0ZLNUg0WHJvUDhaTE40bEJZTzAvdWtrTUFpdUg4ajg9IiwiZHR5cGUiOiJmbG9hdDY0Iiwic2hhcGUiOlsxMjFdfX0sInNlbGVjdGVkIjp7ImlkIjoiMTczMCIsInR5cGUiOiJTZWxlY3Rpb24ifSwic2VsZWN0aW9uX3BvbGljeSI6eyJpZCI6IjE3MzEiLCJ0eXBlIjoiVW5pb25SZW5kZXJlcnMifX0sImlkIjoiMTcwMyIsInR5cGUiOiJDb2x1bW5EYXRhU291cmNlIn0seyJhdHRyaWJ1dGVzIjp7ImRheXMiOlsxLDgsMTUsMjJdfSwiaWQiOiIxNzIyIiwidHlwZSI6IkRheXNUaWNrZXIifV0sInJvb3RfaWRzIjpbIjE2NzQiXX0sInRpdGxlIjoiQm9rZWggQXBwbGljYXRpb24iLCJ2ZXJzaW9uIjoiMS4wLjEifX0KICAgICAgICA8L3NjcmlwdD4KICAgICAgICA8c2NyaXB0IHR5cGU9InRleHQvamF2YXNjcmlwdCI+CiAgICAgICAgICAoZnVuY3Rpb24oKSB7CiAgICAgICAgICAgIHZhciBmbiA9IGZ1bmN0aW9uKCkgewogICAgICAgICAgICAgIEJva2VoLnNhZmVseShmdW5jdGlvbigpIHsKICAgICAgICAgICAgICAgIChmdW5jdGlvbihyb290KSB7CiAgICAgICAgICAgICAgICAgIGZ1bmN0aW9uIGVtYmVkX2RvY3VtZW50KHJvb3QpIHsKICAgICAgICAgICAgICAgICAgICAKICAgICAgICAgICAgICAgICAgdmFyIGRvY3NfanNvbiA9IGRvY3VtZW50LmdldEVsZW1lbnRCeUlkKCcxODcyJykudGV4dENvbnRlbnQ7CiAgICAgICAgICAgICAgICAgIHZhciByZW5kZXJfaXRlbXMgPSBbeyJkb2NpZCI6ImU1MWE4OTA4LWVhZGEtNDYzNC1hMThiLTc5NTc3MzI3MjVlOSIsInJvb3RzIjp7IjE2NzQiOiJmMzY2YjQxMS01Zjg3LTQyOWMtOTlmOS1kNDI1ZmFlMTcxNDUifX1dOwogICAgICAgICAgICAgICAgICByb290LkJva2VoLmVtYmVkLmVtYmVkX2l0ZW1zKGRvY3NfanNvbiwgcmVuZGVyX2l0ZW1zKTsKICAgICAgICAgICAgICAgIAogICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICAgIGlmIChyb290LkJva2VoICE9PSB1bmRlZmluZWQpIHsKICAgICAgICAgICAgICAgICAgICBlbWJlZF9kb2N1bWVudChyb290KTsKICAgICAgICAgICAgICAgICAgfSBlbHNlIHsKICAgICAgICAgICAgICAgICAgICB2YXIgYXR0ZW1wdHMgPSAwOwogICAgICAgICAgICAgICAgICAgIHZhciB0aW1lciA9IHNldEludGVydmFsKGZ1bmN0aW9uKHJvb3QpIHsKICAgICAgICAgICAgICAgICAgICAgIGlmIChyb290LkJva2VoICE9PSB1bmRlZmluZWQpIHsKICAgICAgICAgICAgICAgICAgICAgICAgZW1iZWRfZG9jdW1lbnQocm9vdCk7CiAgICAgICAgICAgICAgICAgICAgICAgIGNsZWFySW50ZXJ2YWwodGltZXIpOwogICAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgICAgICAgYXR0ZW1wdHMrKzsKICAgICAgICAgICAgICAgICAgICAgIGlmIChhdHRlbXB0cyA+IDEwMCkgewogICAgICAgICAgICAgICAgICAgICAgICBjb25zb2xlLmxvZygiQm9rZWg6IEVSUk9SOiBVbmFibGUgdG8gcnVuIEJva2VoSlMgY29kZSBiZWNhdXNlIEJva2VoSlMgbGlicmFyeSBpcyBtaXNzaW5nIik7CiAgICAgICAgICAgICAgICAgICAgICAgIGNsZWFySW50ZXJ2YWwodGltZXIpOwogICAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgICAgIH0sIDEwLCByb290KQogICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICB9KSh3aW5kb3cpOwogICAgICAgICAgICAgIH0pOwogICAgICAgICAgICB9OwogICAgICAgICAgICBpZiAoZG9jdW1lbnQucmVhZHlTdGF0ZSAhPSAibG9hZGluZyIpIGZuKCk7CiAgICAgICAgICAgIGVsc2UgZG9jdW1lbnQuYWRkRXZlbnRMaXN0ZW5lcigiRE9NQ29udGVudExvYWRlZCIsIGZuKTsKICAgICAgICAgIH0pKCk7CiAgICAgICAgPC9zY3JpcHQ+CiAgICAKICA8L2JvZHk+CiAgCjwvaHRtbD4=&quot; width=&quot;790&quot; style=&quot;border:none !important;&quot; height=&quot;330&quot;></iframe>`)[0];
                popup_65c7f72788754a0ba8fe1570f9117af0.setContent(i_frame_3e0f8909d1b7489e86f7fe897ea6a3b8);


            marker_e2a117ec22924bef9b4ef7842e0dead0.bindPopup(popup_65c7f72788754a0ba8fe1570f9117af0)
            ;




        var marker_af4e9250c52146579f066384e525743a = L.marker(
            [41.6885, -69.951],
            {
                icon: new L.Icon.Default()
                }
            ).addTo(map_54cb488c2c244a8daafaa65ad6b9b54b);



                var icon_cc16b5cf07b7493d9d61db87afe725e5 = L.AwesomeMarkers.icon({
                    icon: 'stats',
                    iconColor: 'white',
                    markerColor: 'green',
                    prefix: 'glyphicon',
                    extraClasses: 'fa-rotate-0'
                    });
                marker_af4e9250c52146579f066384e525743a.setIcon(icon_cc16b5cf07b7493d9d61db87afe725e5);


            var popup_e194341ac9f2433c82b71f9482595645 = L.popup({maxWidth: '2650'

            });


                var i_frame_5c12e4dbdf28402ca44a364c57c903ce = $(`<iframe src=&quot;data:text/html;charset=utf-8;base64,CiAgICAKCgoKPCFET0NUWVBFIGh0bWw+CjxodG1sIGxhbmc9ImVuIj4KICAKICA8aGVhZD4KICAgIAogICAgICA8bWV0YSBjaGFyc2V0PSJ1dGYtOCI+CiAgICAgIDx0aXRsZT44NDQ3NDM1PC90aXRsZT4KICAgICAgCiAgICAgIAogICAgICAgIAogICAgICAgICAgCiAgICAgICAgPGxpbmsgcmVsPSJzdHlsZXNoZWV0IiBocmVmPSJodHRwczovL2Nkbi5weWRhdGEub3JnL2Jva2VoL3JlbGVhc2UvYm9rZWgtMS4wLjEubWluLmNzcyIgdHlwZT0idGV4dC9jc3MiIC8+CiAgICAgICAgCiAgICAgICAgCiAgICAgICAgICAKICAgICAgICA8c2NyaXB0IHR5cGU9InRleHQvamF2YXNjcmlwdCIgc3JjPSJodHRwczovL2Nkbi5weWRhdGEub3JnL2Jva2VoL3JlbGVhc2UvYm9rZWgtMS4wLjEubWluLmpzIj48L3NjcmlwdD4KICAgICAgICA8c2NyaXB0IHR5cGU9InRleHQvamF2YXNjcmlwdCI+CiAgICAgICAgICAgIEJva2VoLnNldF9sb2dfbGV2ZWwoImluZm8iKTsKICAgICAgICA8L3NjcmlwdD4KICAgICAgICAKICAgICAgCiAgICAgIAogICAgCiAgPC9oZWFkPgogIAogIAogIDxib2R5PgogICAgCiAgICAgIAogICAgICAgIAogICAgICAgICAgCiAgICAgICAgICAKICAgICAgICAgICAgCiAgICAgICAgICAgICAgPGRpdiBjbGFzcz0iYmstcm9vdCIgaWQ9Ijc5YmY4NzBjLWM2MjAtNGQzMS04OTdhLTRhZmQxOWJiZGM1NCI+PC9kaXY+CiAgICAgICAgICAgIAogICAgICAgICAgCiAgICAgICAgCiAgICAgIAogICAgICAKICAgICAgICA8c2NyaXB0IHR5cGU9ImFwcGxpY2F0aW9uL2pzb24iIGlkPSIyMTQ0Ij4KICAgICAgICAgIHsiNDYyOTY2N2ItYzlkOC00MzJjLTkxMDktYmEyNzFlYWQ0ZGMyIjp7InJvb3RzIjp7InJlZmVyZW5jZXMiOlt7ImF0dHJpYnV0ZXMiOnsic291cmNlIjp7ImlkIjoiMTkxNyIsInR5cGUiOiJDb2x1bW5EYXRhU291cmNlIn19LCJpZCI6IjE5MjEiLCJ0eXBlIjoiQ0RTVmlldyJ9LHsiYXR0cmlidXRlcyI6e30sImlkIjoiMTg4MiIsInR5cGUiOiJMaW5lYXJTY2FsZSJ9LHsiYXR0cmlidXRlcyI6eyJkaW1lbnNpb24iOjEsInBsb3QiOnsiaWQiOiIxODc0Iiwic3VidHlwZSI6IkZpZ3VyZSIsInR5cGUiOiJQbG90In0sInRpY2tlciI6eyJpZCI6IjE4OTAiLCJ0eXBlIjoiQmFzaWNUaWNrZXIifX0sImlkIjoiMTg5MyIsInR5cGUiOiJHcmlkIn0seyJhdHRyaWJ1dGVzIjp7ImF4aXNfbGFiZWwiOiJEYXRlL3RpbWUiLCJmb3JtYXR0ZXIiOnsiaWQiOiIxOTM4IiwidHlwZSI6IkRhdGV0aW1lVGlja0Zvcm1hdHRlciJ9LCJwbG90Ijp7ImlkIjoiMTg3NCIsInN1YnR5cGUiOiJGaWd1cmUiLCJ0eXBlIjoiUGxvdCJ9LCJ0aWNrZXIiOnsiaWQiOiIxODg1IiwidHlwZSI6IkRhdGV0aW1lVGlja2VyIn19LCJpZCI6IjE4ODQiLCJ0eXBlIjoiRGF0ZXRpbWVBeGlzIn0seyJhdHRyaWJ1dGVzIjp7Im1vbnRocyI6WzAsMiw0LDYsOCwxMF19LCJpZCI6IjE5NDkiLCJ0eXBlIjoiTW9udGhzVGlja2VyIn0seyJhdHRyaWJ1dGVzIjp7fSwiaWQiOiIxOTUyIiwidHlwZSI6IlllYXJzVGlja2VyIn0seyJhdHRyaWJ1dGVzIjp7ImxpbmVfYWxwaGEiOjAuNjUsImxpbmVfY2FwIjoicm91bmQiLCJsaW5lX2NvbG9yIjoiIzJjYTAyYyIsImxpbmVfam9pbiI6InJvdW5kIiwibGluZV93aWR0aCI6NSwieCI6eyJmaWVsZCI6IngifSwieSI6eyJmaWVsZCI6InkifX0sImlkIjoiMTkxOCIsInR5cGUiOiJMaW5lIn0seyJhdHRyaWJ1dGVzIjp7ImRheXMiOlsxLDgsMTUsMjJdfSwiaWQiOiIxOTQ2IiwidHlwZSI6IkRheXNUaWNrZXIifSx7ImF0dHJpYnV0ZXMiOnt9LCJpZCI6IjE5NTQiLCJ0eXBlIjoiU2VsZWN0aW9uIn0seyJhdHRyaWJ1dGVzIjp7ImxpbmVfYWxwaGEiOjAuMSwibGluZV9jYXAiOiJyb3VuZCIsImxpbmVfY29sb3IiOiIjMWY3N2I0IiwibGluZV9qb2luIjoicm91bmQiLCJsaW5lX3dpZHRoIjo1LCJ4Ijp7ImZpZWxkIjoieCJ9LCJ5Ijp7ImZpZWxkIjoieSJ9fSwiaWQiOiIxOTA1IiwidHlwZSI6IkxpbmUifSx7ImF0dHJpYnV0ZXMiOnsibW9udGhzIjpbMCw2XX0sImlkIjoiMTk1MSIsInR5cGUiOiJNb250aHNUaWNrZXIifSx7ImF0dHJpYnV0ZXMiOnsiYWN0aXZlX2RyYWciOiJhdXRvIiwiYWN0aXZlX2luc3BlY3QiOiJhdXRvIiwiYWN0aXZlX211bHRpIjpudWxsLCJhY3RpdmVfc2Nyb2xsIjoiYXV0byIsImFjdGl2ZV90YXAiOiJhdXRvIiwidG9vbHMiOlt7ImlkIjoiMTg5NCIsInR5cGUiOiJQYW5Ub29sIn0seyJpZCI6IjE4OTUiLCJ0eXBlIjoiQm94Wm9vbVRvb2wifSx7ImlkIjoiMTg5NiIsInR5cGUiOiJSZXNldFRvb2wifSx7ImlkIjoiMTkwOCIsInR5cGUiOiJIb3ZlclRvb2wifSx7ImlkIjoiMTkxNSIsInR5cGUiOiJIb3ZlclRvb2wifSx7ImlkIjoiMTkyMiIsInR5cGUiOiJIb3ZlclRvb2wifSx7ImlkIjoiMTkyOSIsInR5cGUiOiJIb3ZlclRvb2wifV19LCJpZCI6IjE4OTciLCJ0eXBlIjoiVG9vbGJhciJ9LHsiYXR0cmlidXRlcyI6e30sImlkIjoiMTkzOCIsInR5cGUiOiJEYXRldGltZVRpY2tGb3JtYXR0ZXIifSx7ImF0dHJpYnV0ZXMiOnsiYmFzZSI6NjAsIm1hbnRpc3NhcyI6WzEsMiw1LDEwLDE1LDIwLDMwXSwibWF4X2ludGVydmFsIjoxODAwMDAwLjAsIm1pbl9pbnRlcnZhbCI6MTAwMC4wLCJudW1fbWlub3JfdGlja3MiOjB9LCJpZCI6IjE5NDIiLCJ0eXBlIjoiQWRhcHRpdmVUaWNrZXIifSx7ImF0dHJpYnV0ZXMiOnsibW9udGhzIjpbMCw0LDhdfSwiaWQiOiIxOTUwIiwidHlwZSI6Ik1vbnRoc1RpY2tlciJ9LHsiYXR0cmlidXRlcyI6eyJjYWxsYmFjayI6bnVsbCwicmVuZGVyZXJzIjpbeyJpZCI6IjE5MjAiLCJ0eXBlIjoiR2x5cGhSZW5kZXJlciJ9XSwidG9vbHRpcHMiOltbIk5hbWUiLCJUaW1lX3YyX0hpc3RvcnlfQmVzdCJdLFsiQmlhcyIsIi0xLjM4Il0sWyJTa2lsbCIsIjAuNDMiXV19LCJpZCI6IjE5MjIiLCJ0eXBlIjoiSG92ZXJUb29sIn0seyJhdHRyaWJ1dGVzIjp7ImRhdGFfc291cmNlIjp7ImlkIjoiMTkxNyIsInR5cGUiOiJDb2x1bW5EYXRhU291cmNlIn0sImdseXBoIjp7ImlkIjoiMTkxOCIsInR5cGUiOiJMaW5lIn0sImhvdmVyX2dseXBoIjpudWxsLCJtdXRlZF9nbHlwaCI6bnVsbCwibm9uc2VsZWN0aW9uX2dseXBoIjp7ImlkIjoiMTkxOSIsInR5cGUiOiJMaW5lIn0sInNlbGVjdGlvbl9nbHlwaCI6bnVsbCwidmlldyI6eyJpZCI6IjE5MjEiLCJ0eXBlIjoiQ0RTVmlldyJ9fSwiaWQiOiIxOTIwIiwidHlwZSI6IkdseXBoUmVuZGVyZXIifSx7ImF0dHJpYnV0ZXMiOnsiY2xpY2tfcG9saWN5IjoibXV0ZSIsIml0ZW1zIjpbeyJpZCI6IjE5MzIiLCJ0eXBlIjoiTGVnZW5kSXRlbSJ9LHsiaWQiOiIxOTMzIiwidHlwZSI6IkxlZ2VuZEl0ZW0ifSx7ImlkIjoiMTkzNCIsInR5cGUiOiJMZWdlbmRJdGVtIn0seyJpZCI6IjE5MzUiLCJ0eXBlIjoiTGVnZW5kSXRlbSJ9XSwibG9jYXRpb24iOlswLDYwXSwicGxvdCI6eyJpZCI6IjE4NzQiLCJzdWJ0eXBlIjoiRmlndXJlIiwidHlwZSI6IlBsb3QifX0sImlkIjoiMTkzMSIsInR5cGUiOiJMZWdlbmQifSx7ImF0dHJpYnV0ZXMiOnsiYm90dG9tX3VuaXRzIjoic2NyZWVuIiwiZmlsbF9hbHBoYSI6eyJ2YWx1ZSI6MC41fSwiZmlsbF9jb2xvciI6eyJ2YWx1ZSI6ImxpZ2h0Z3JleSJ9LCJsZWZ0X3VuaXRzIjoic2NyZWVuIiwibGV2ZWwiOiJvdmVybGF5IiwibGluZV9hbHBoYSI6eyJ2YWx1ZSI6MS4wfSwibGluZV9jb2xvciI6eyJ2YWx1ZSI6ImJsYWNrIn0sImxpbmVfZGFzaCI6WzQsNF0sImxpbmVfd2lkdGgiOnsidmFsdWUiOjJ9LCJwbG90IjpudWxsLCJyZW5kZXJfbW9kZSI6ImNzcyIsInJpZ2h0X3VuaXRzIjoic2NyZWVuIiwidG9wX3VuaXRzIjoic2NyZWVuIn0sImlkIjoiMTg5OCIsInR5cGUiOiJCb3hBbm5vdGF0aW9uIn0seyJhdHRyaWJ1dGVzIjp7ImNhbGxiYWNrIjpudWxsLCJkYXRhIjp7IngiOnsiX19uZGFycmF5X18iOiJBQUNBVnBzZGRrSUFBR2pGbmgxMlFnQUFVRFNpSFhaQ0FBQkF2TzBkZGtJQUFDZ3I4UjEyUWdBQUVKcjBIWFpDQUFBQUlrQWVka0lBQU9pUVF4NTJRZ0FBMFA5R0huWkNBQURBaDVJZWRrSUFBS2oybFI1MlFnQUFrR1daSG5aQ0FBQ0E3ZVFlZGtJQUFHaGM2QjUyUWdBQVVNdnJIblpDIiwiZHR5cGUiOiJmbG9hdDY0Iiwic2hhcGUiOlsxNV19LCJ5Ijp7Il9fbmRhcnJheV9fIjoiQUFBQUFOWVA0YjhBQUFDQXppN2h2d0FBQUFESFRlRy9BQUFBUUNIMzQ3OEFBQURncE5QanZ3QUFBSUFvc09PL0FBQUFJSGlqNEw4QUFBQWdrYTNmdndBQUFDQXlGTjYvQUFBQXdPZG50RDhBQUFBZ09xbXpQd0FBQUlDTTZySS9BQUFBZ0FjOWhEOEFBQUNBQnoyRVB3QUFBSUFIUFlRLyIsImR0eXBlIjoiZmxvYXQ2NCIsInNoYXBlIjpbMTVdfX0sInNlbGVjdGVkIjp7ImlkIjoiMTk2MCIsInR5cGUiOiJTZWxlY3Rpb24ifSwic2VsZWN0aW9uX3BvbGljeSI6eyJpZCI6IjE5NjEiLCJ0eXBlIjoiVW5pb25SZW5kZXJlcnMifX0sImlkIjoiMTkyNCIsInR5cGUiOiJDb2x1bW5EYXRhU291cmNlIn0seyJhdHRyaWJ1dGVzIjp7ImRheXMiOlsxLDQsNywxMCwxMywxNiwxOSwyMiwyNSwyOF19LCJpZCI6IjE5NDUiLCJ0eXBlIjoiRGF5c1RpY2tlciJ9LHsiYXR0cmlidXRlcyI6eyJsaW5lX2FscGhhIjowLjEsImxpbmVfY2FwIjoicm91bmQiLCJsaW5lX2NvbG9yIjoiIzFmNzdiNCIsImxpbmVfam9pbiI6InJvdW5kIiwibGluZV93aWR0aCI6NSwieCI6eyJmaWVsZCI6IngifSwieSI6eyJmaWVsZCI6InkifX0sImlkIjoiMTkxOSIsInR5cGUiOiJMaW5lIn0seyJhdHRyaWJ1dGVzIjp7Im92ZXJsYXkiOnsiaWQiOiIxODk4IiwidHlwZSI6IkJveEFubm90YXRpb24ifX0sImlkIjoiMTg5NSIsInR5cGUiOiJCb3hab29tVG9vbCJ9LHsiYXR0cmlidXRlcyI6eyJwbG90IjpudWxsLCJ0ZXh0IjoiODQ0NzQzNSJ9LCJpZCI6IjE4NzMiLCJ0eXBlIjoiVGl0bGUifSx7ImF0dHJpYnV0ZXMiOnsiZGF0YV9zb3VyY2UiOnsiaWQiOiIxOTAzIiwidHlwZSI6IkNvbHVtbkRhdGFTb3VyY2UifSwiZ2x5cGgiOnsiaWQiOiIxOTA0IiwidHlwZSI6IkxpbmUifSwiaG92ZXJfZ2x5cGgiOm51bGwsIm11dGVkX2dseXBoIjpudWxsLCJub25zZWxlY3Rpb25fZ2x5cGgiOnsiaWQiOiIxOTA1IiwidHlwZSI6IkxpbmUifSwic2VsZWN0aW9uX2dseXBoIjpudWxsLCJ2aWV3Ijp7ImlkIjoiMTkwNyIsInR5cGUiOiJDRFNWaWV3In19LCJpZCI6IjE5MDYiLCJ0eXBlIjoiR2x5cGhSZW5kZXJlciJ9LHsiYXR0cmlidXRlcyI6e30sImlkIjoiMTg5NCIsInR5cGUiOiJQYW5Ub29sIn0seyJhdHRyaWJ1dGVzIjp7ImxhYmVsIjp7InZhbHVlIjoiVGltZV92Ml9IaXN0b3J5X0Jlc3QifSwicmVuZGVyZXJzIjpbeyJpZCI6IjE5MjAiLCJ0eXBlIjoiR2x5cGhSZW5kZXJlciJ9XX0sImlkIjoiMTkzNCIsInR5cGUiOiJMZWdlbmRJdGVtIn0seyJhdHRyaWJ1dGVzIjp7ImJhc2UiOjI0LCJtYW50aXNzYXMiOlsxLDIsNCw2LDgsMTJdLCJtYXhfaW50ZXJ2YWwiOjQzMjAwMDAwLjAsIm1pbl9pbnRlcnZhbCI6MzYwMDAwMC4wLCJudW1fbWlub3JfdGlja3MiOjB9LCJpZCI6IjE5NDMiLCJ0eXBlIjoiQWRhcHRpdmVUaWNrZXIifSx7ImF0dHJpYnV0ZXMiOnt9LCJpZCI6IjE5NTciLCJ0eXBlIjoiVW5pb25SZW5kZXJlcnMifSx7ImF0dHJpYnV0ZXMiOnsibGluZV9hbHBoYSI6MC4xLCJsaW5lX2NhcCI6InJvdW5kIiwibGluZV9jb2xvciI6IiMxZjc3YjQiLCJsaW5lX2pvaW4iOiJyb3VuZCIsImxpbmVfd2lkdGgiOjUsIngiOnsiZmllbGQiOiJ4In0sInkiOnsiZmllbGQiOiJ5In19LCJpZCI6IjE5MTIiLCJ0eXBlIjoiTGluZSJ9LHsiYXR0cmlidXRlcyI6e30sImlkIjoiMTk2MCIsInR5cGUiOiJTZWxlY3Rpb24ifSx7ImF0dHJpYnV0ZXMiOnt9LCJpZCI6IjE5NDAiLCJ0eXBlIjoiQmFzaWNUaWNrRm9ybWF0dGVyIn0seyJhdHRyaWJ1dGVzIjp7ImxhYmVsIjp7InZhbHVlIjoiT2JzZXJ2YXRpb25zIn0sInJlbmRlcmVycyI6W3siaWQiOiIxOTA2IiwidHlwZSI6IkdseXBoUmVuZGVyZXIifV19LCJpZCI6IjE5MzIiLCJ0eXBlIjoiTGVnZW5kSXRlbSJ9LHsiYXR0cmlidXRlcyI6e30sImlkIjoiMTk1NSIsInR5cGUiOiJVbmlvblJlbmRlcmVycyJ9LHsiYXR0cmlidXRlcyI6eyJzb3VyY2UiOnsiaWQiOiIxOTEwIiwidHlwZSI6IkNvbHVtbkRhdGFTb3VyY2UifX0sImlkIjoiMTkxNCIsInR5cGUiOiJDRFNWaWV3In0seyJhdHRyaWJ1dGVzIjp7fSwiaWQiOiIxOTU4IiwidHlwZSI6IlNlbGVjdGlvbiJ9LHsiYXR0cmlidXRlcyI6eyJzb3VyY2UiOnsiaWQiOiIxOTI0IiwidHlwZSI6IkNvbHVtbkRhdGFTb3VyY2UifX0sImlkIjoiMTkyOCIsInR5cGUiOiJDRFNWaWV3In0seyJhdHRyaWJ1dGVzIjp7fSwiaWQiOiIxODkwIiwidHlwZSI6IkJhc2ljVGlja2VyIn0seyJhdHRyaWJ1dGVzIjp7ImRheXMiOlsxLDIsMyw0LDUsNiw3LDgsOSwxMCwxMSwxMiwxMywxNCwxNSwxNiwxNywxOCwxOSwyMCwyMSwyMiwyMywyNCwyNSwyNiwyNywyOCwyOSwzMCwzMV19LCJpZCI6IjE5NDQiLCJ0eXBlIjoiRGF5c1RpY2tlciJ9LHsiYXR0cmlidXRlcyI6eyJzb3VyY2UiOnsiaWQiOiIxOTAzIiwidHlwZSI6IkNvbHVtbkRhdGFTb3VyY2UifX0sImlkIjoiMTkwNyIsInR5cGUiOiJDRFNWaWV3In0seyJhdHRyaWJ1dGVzIjp7ImNhbGxiYWNrIjpudWxsLCJkYXRhIjp7IngiOnsiX19uZGFycmF5X18iOiJBQUJnaWNRZGRrSUFBRWo0eHgxMlFnQUFNR2ZMSFhaQ0FBQWc3eFllZGtJQUFBaGVHaDUyUWdBQThNd2RIblpDQUFEZ1ZHa2Vka0lBQU1qRGJCNTJRZ0FBc0RKd0huWkNBQUNndXJzZWRrSUFBSWdwdng1MlFnQUFjSmpDSG5aQyIsImR0eXBlIjoiZmxvYXQ2NCIsInNoYXBlIjpbMTJdfSwieSI6eyJfX25kYXJyYXlfXyI6IkFBQUFRTXJwMWI4QUFBQkFoK0RWdndBQUFFQkUxOVcvQUFBQTRJRUwxYjhBQUFDQVRJclR2d0FBQUNBWENkSy9BQUFBWVAwaXpqOEFBQUJBTDBEUFB3QUFBS0N3THRBL0FBQUFvTlZ2M0Q4QUFBQ2cxVy9jUHdBQUFLRFZiOXcvIiwiZHR5cGUiOiJmbG9hdDY0Iiwic2hhcGUiOlsxMl19fSwic2VsZWN0ZWQiOnsiaWQiOiIxOTU2IiwidHlwZSI6IlNlbGVjdGlvbiJ9LCJzZWxlY3Rpb25fcG9saWN5Ijp7ImlkIjoiMTk1NyIsInR5cGUiOiJVbmlvblJlbmRlcmVycyJ9fSwiaWQiOiIxOTEwIiwidHlwZSI6IkNvbHVtbkRhdGFTb3VyY2UifSx7ImF0dHJpYnV0ZXMiOnsiY2FsbGJhY2siOm51bGwsInJlbmRlcmVycyI6W3siaWQiOiIxOTI3IiwidHlwZSI6IkdseXBoUmVuZGVyZXIifV0sInRvb2x0aXBzIjpbWyJOYW1lIiwiZ2xvYmFsIl0sWyJCaWFzIiwiLTEuMzUiXSxbIlNraWxsIiwiMC40MyJdXX0sImlkIjoiMTkyOSIsInR5cGUiOiJIb3ZlclRvb2wifSx7ImF0dHJpYnV0ZXMiOnsiY2FsbGJhY2siOm51bGwsImRhdGEiOnsieCI6eyJfX25kYXJyYXlfXyI6IkFBQ0FWcHNkZGtJQUFHakZuaDEyUWdBQVVEU2lIWFpDQUFBNG82VWRka0lBQUNBU3FSMTJRZ0FBQ0lHc0hYWkNBQUR3NzY4ZGRrSUFBTmhlc3gxMlFnQUF3TTIySFhaQ0FBQ29QTG9kZGtJQUFKQ3J2UjEyUWdBQWVCckJIWFpDQUFCZ2ljUWRka0lBQUVqNHh4MTJRZ0FBTUdmTEhYWkNBQUFZMXM0ZGRrSUFBQUJGMGgxMlFnQUE2TFBWSFhaQ0FBRFFJdGtkZGtJQUFMaVIzQjEyUWdBQW9BRGdIWFpDQUFDSWIrTWRka0lBQUhEZTVoMTJRZ0FBV0UzcUhYWkNBQUJBdk8wZGRrSUFBQ2dyOFIxMlFnQUFFSnIwSFhaQ0FBRDRDUGdkZGtJQUFPQjMreDEyUWdBQXlPYitIWFpDQUFDd1ZRSWVka0lBQUpqRUJSNTJRZ0FBZ0RNSkhuWkNBQUJvb2d3ZWRrSUFBRkFSRUI1MlFnQUFPSUFUSG5aQ0FBQWc3eFllZGtJQUFBaGVHaDUyUWdBQThNd2RIblpDQUFEWU95RWVka0lBQU1DcUpCNTJRZ0FBcUJrb0huWkNBQUNRaUNzZWRrSUFBSGozTGg1MlFnQUFZR1l5SG5aQ0FBQkkxVFVlZGtJQUFEQkVPUjUyUWdBQUdMTThIblpDQUFBQUlrQWVka0lBQU9pUVF4NTJRZ0FBMFA5R0huWkNBQUM0YmtvZWRrSUFBS0RkVFI1MlFnQUFpRXhSSG5aQ0FBQnd1MVFlZGtJQUFGZ3FXQjUyUWdBQVFKbGJIblpDQUFBb0NGOGVka0lBQUJCM1loNTJRZ0FBK09WbEhuWkNBQURnVkdrZWRrSUFBTWpEYkI1MlFnQUFzREp3SG5aQ0FBQ1lvWE1lZGtJQUFJQVFkeDUyUWdBQWFIOTZIblpDQUFCUTduMGVka0lBQURoZGdSNTJRZ0FBSU15RUhuWkNBQUFJTzRnZWRrSUFBUENwaXg1MlFnQUEyQmlQSG5aQ0FBREFoNUllZGtJQUFLajJsUjUyUWdBQWtHV1pIblpDQUFCNDFKd2Vka0lBQUdCRG9CNTJRZ0FBU0xLakhuWkNBQUF3SWFjZWRrSUFBQUQvclI1MlFnQUE2RzJ4SG5aQ0FBRFEzTFFlZGtJQUFMaEx1QjUyUWdBQW9McTdIblpDQUFDSUtiOGVka0lBQUhDWXdoNTJRZ0FBV0FmR0huWkNBQUJBZHNrZWRrSUFBQ2psekI1MlFnQUFFRlRRSG5aQ0FBRElvTm9lZGtJQUFMQVAzaDUyUWdBQW1IN2hIblpDQUFDQTdlUWVka0lBQUdoYzZCNTJRZ0FBVU12ckhuWkNBQUE0T3U4ZWRrSUFBQ0NwOGg1MlFnQUFDQmoySG5aQ0FBRHdodmtlZGtJQUFOajEvQjUyUWdBQXdHUUFIM1pDQUFDbzB3TWZka0lBQUpCQ0J4OTJRZ0FBZUxFS0gzWkNBQUJnSUE0ZmRrSUFBRWlQRVI5MlFnQUFNUDRVSDNaQ0FBQVliUmdmZGtJQUFBRGNHeDkyUWdBQTZFb2ZIM1pDQUFEUXVTSWZka0lBQUxnb0poOTJRZ0FBb0pjcEgzWkNBQUNJQmkwZmRrSUFBSEIxTUI5MlFnQUFXT1F6SDNaQ0FBQkFVemNmZGtJPSIsImR0eXBlIjoiZmxvYXQ2NCIsInNoYXBlIjpbMTE4XX0sInkiOnsiX19uZGFycmF5X18iOiJmVDgxWHJwSjZEL3NVYmdlaGV2elA3YnovZFI0NmZnL3dNcWhSYmJ6K1QrRHdNcWhSYmIzUHplSlFXRGwwUEkvYVpIdGZEODE2ajlPWWhCWU9iVGdQNFhyVWJnZWhkTS9LNGNXMmM3M3d6L2pwWnZFSUxDeVA3YnovZFI0NmRZLzBTTGIrWDVxN0QrUzdYdy9OVjcyUDA1aUVGZzV0UHcvU09GNkZLNUgveit4Y21pUjdYejlQMCtObDI0U2cvZy9GdG5POTFQajhUK294a3MzaVVIb1A3Z2VoZXRSdU40L2pHem4rNm54MGovWG8zQTlDdGZEUC9UOTFIanBKckUvWU9YUUl0djUxai9GSUxCeWFKSHRQM2pwSmpFSXJQWS9WT09sbThRZy9EOEsxNk53UFFyOVA1THRmRDgxWHZvL1g3cEpEQUlyOVQrV1E0dHM1L3Z0UDY1SDRYb1VydU0vOWloY2o4TDEyRCs4ZEpNWUJGYk9QMk1RV0RtMHlNWS91a2tNQWl1SDRqOHRzcDN2cDhieFA2RkZ0dlA5MVBvL2JlZjdxZkhTQUVCM3ZwOGFMOTBCUUQ4MVhycEpEQUZBajhMMUtGeVAvRDhLMTZOd1BRcjFQeHN2M1NRR2dlMC9LNGNXMmM3MzR6K0pRV0RsMENMYlAwamhlaFN1UjlFL25lK254a3MzeVQ5eFBRclhvM0RsUDVxWm1abVptZk0vSEZwa085OVArejhUZzhES29VVUFRTHBKREFJcmh3QkFSSXRzNS91cC9ULzkxSGpwSmpINFB5L2RKQWFCbGZFL2ZUODFYcnBKNkQ4aHNISm9rZTNnUDZqR1N6ZUpRZGcvd01xaFJiYnozVDlGdHZQOTFIanRQeS9kSkFhQmxmay9qR3puKzZueEFVQTlDdGVqY0QwRlFETXpNek16TXdWQWFaSHRmRDgxQlVCemFKSHRmRDhGUUZnNXRNaDJ2Z0ZBbGtPTGJPZjcrei9vKzZueDBrMzJQeWxjajhMMUtQUS9TZ3dDSzRjVzh6OE9MYktkNzZmMlA4VWdzSEpva2YwL2c4REtvVVcyQWtBY1dtUTczMDhGUUhOb2tlMThQd1ZBYzJpUjdYdy9CVUFFVmc0dHNwMy9QMUs0SG9YclVmdy81ZEFpMi9sKzlqOXhQUXJYbzNEeFA1WkRpMnpuKyswL3ZIU1RHQVJXOGo4aHNISm9rZTM0UHd3Q0s0Y1cyUUJBQ3RlamNEMEtCVUJ6YUpIdGZEOEZRUDNVZU9rbU1RVkFYSS9DOVNoYytUOElyQnhhWkR2M1AycThkSk1ZQlBJL0JvR1ZRNHRzNnorWGJoS0R3TXJwUCsrbnhrczNpZkUvWU9YUUl0djUrRDlPWWhCWU9iUUFRRE16TXpNek13UkFzcDN2cDhaTEJVQWNXbVE3MzA4RlFNUDFLRnlQd2dOQTMwK05sMjRTQUVCa085OVBqWmY0UDgzTXpNek16UEkvZDc2ZkdpL2Q3RDlPWWhCWU9iVG9QenEweUhhK24rNC9VcmdlaGV0UjlEK3NIRnBrTzkvOVB6emZUNDJYYmdKQWVPa21NUWlzQkVCTU40bEJZT1VFUUx4MGt4Z0VWZ05BQ0t3Y1dtUTdBRUFoc0hKb2tlMzRQN3gwa3hnRVZ2SS9BeXVIRnRuTzZ6OD0iLCJkdHlwZSI6ImZsb2F0NjQiLCJzaGFwZSI6WzExOF19fSwic2VsZWN0ZWQiOnsiaWQiOiIxOTU0IiwidHlwZSI6IlNlbGVjdGlvbiJ9LCJzZWxlY3Rpb25fcG9saWN5Ijp7ImlkIjoiMTk1NSIsInR5cGUiOiJVbmlvblJlbmRlcmVycyJ9fSwiaWQiOiIxOTAzIiwidHlwZSI6IkNvbHVtbkRhdGFTb3VyY2UifSx7ImF0dHJpYnV0ZXMiOnt9LCJpZCI6IjE4ODAiLCJ0eXBlIjoiTGluZWFyU2NhbGUifSx7ImF0dHJpYnV0ZXMiOnsiY2FsbGJhY2siOm51bGwsInJlbmRlcmVycyI6W3siaWQiOiIxOTA2IiwidHlwZSI6IkdseXBoUmVuZGVyZXIifV0sInRvb2x0aXBzIjpbWyJOYW1lIiwiT2JzZXJ2YXRpb25zIl0sWyJCaWFzIiwiTkEiXSxbIlNraWxsIiwiTkEiXV19LCJpZCI6IjE5MDgiLCJ0eXBlIjoiSG92ZXJUb29sIn0seyJhdHRyaWJ1dGVzIjp7ImRhdGFfc291cmNlIjp7ImlkIjoiMTkyNCIsInR5cGUiOiJDb2x1bW5EYXRhU291cmNlIn0sImdseXBoIjp7ImlkIjoiMTkyNSIsInR5cGUiOiJMaW5lIn0sImhvdmVyX2dseXBoIjpudWxsLCJtdXRlZF9nbHlwaCI6bnVsbCwibm9uc2VsZWN0aW9uX2dseXBoIjp7ImlkIjoiMTkyNiIsInR5cGUiOiJMaW5lIn0sInNlbGVjdGlvbl9nbHlwaCI6bnVsbCwidmlldyI6eyJpZCI6IjE5MjgiLCJ0eXBlIjoiQ0RTVmlldyJ9fSwiaWQiOiIxOTI3IiwidHlwZSI6IkdseXBoUmVuZGVyZXIifSx7ImF0dHJpYnV0ZXMiOnsiY2FsbGJhY2siOm51bGwsInJlbmRlcmVycyI6W3siaWQiOiIxOTEzIiwidHlwZSI6IkdseXBoUmVuZGVyZXIifV0sInRvb2x0aXBzIjpbWyJOYW1lIiwiVGltZV92Ml9BdmVyYWdlc19CZXN0Il0sWyJCaWFzIiwiLTEuMTYiXSxbIlNraWxsIiwiMC41NyJdXX0sImlkIjoiMTkxNSIsInR5cGUiOiJIb3ZlclRvb2wifSx7ImF0dHJpYnV0ZXMiOnsicGxvdCI6eyJpZCI6IjE4NzQiLCJzdWJ0eXBlIjoiRmlndXJlIiwidHlwZSI6IlBsb3QifSwidGlja2VyIjp7ImlkIjoiMTg4NSIsInR5cGUiOiJEYXRldGltZVRpY2tlciJ9fSwiaWQiOiIxODg4IiwidHlwZSI6IkdyaWQifSx7ImF0dHJpYnV0ZXMiOnsibGluZV9hbHBoYSI6MC42NSwibGluZV9jYXAiOiJyb3VuZCIsImxpbmVfY29sb3IiOiIjZmZiYjc4IiwibGluZV9qb2luIjoicm91bmQiLCJsaW5lX3dpZHRoIjo1LCJ4Ijp7ImZpZWxkIjoieCJ9LCJ5Ijp7ImZpZWxkIjoieSJ9fSwiaWQiOiIxOTExIiwidHlwZSI6IkxpbmUifSx7ImF0dHJpYnV0ZXMiOnsibWFudGlzc2FzIjpbMSwyLDVdLCJtYXhfaW50ZXJ2YWwiOjUwMC4wLCJudW1fbWlub3JfdGlja3MiOjB9LCJpZCI6IjE5NDEiLCJ0eXBlIjoiQWRhcHRpdmVUaWNrZXIifSx7ImF0dHJpYnV0ZXMiOnsiYXhpc19sYWJlbCI6IldhdGVyIEhlaWdodCAobSkiLCJmb3JtYXR0ZXIiOnsiaWQiOiIxOTQwIiwidHlwZSI6IkJhc2ljVGlja0Zvcm1hdHRlciJ9LCJwbG90Ijp7ImlkIjoiMTg3NCIsInN1YnR5cGUiOiJGaWd1cmUiLCJ0eXBlIjoiUGxvdCJ9LCJ0aWNrZXIiOnsiaWQiOiIxODkwIiwidHlwZSI6IkJhc2ljVGlja2VyIn19LCJpZCI6IjE4ODkiLCJ0eXBlIjoiTGluZWFyQXhpcyJ9LHsiYXR0cmlidXRlcyI6eyJsYWJlbCI6eyJ2YWx1ZSI6Imdsb2JhbCJ9LCJyZW5kZXJlcnMiOlt7ImlkIjoiMTkyNyIsInR5cGUiOiJHbHlwaFJlbmRlcmVyIn1dfSwiaWQiOiIxOTM1IiwidHlwZSI6IkxlZ2VuZEl0ZW0ifSx7ImF0dHJpYnV0ZXMiOnt9LCJpZCI6IjE5NTYiLCJ0eXBlIjoiU2VsZWN0aW9uIn0seyJhdHRyaWJ1dGVzIjp7fSwiaWQiOiIxODk2IiwidHlwZSI6IlJlc2V0VG9vbCJ9LHsiYXR0cmlidXRlcyI6eyJjYWxsYmFjayI6bnVsbH0sImlkIjoiMTg3NiIsInR5cGUiOiJEYXRhUmFuZ2UxZCJ9LHsiYXR0cmlidXRlcyI6eyJiZWxvdyI6W3siaWQiOiIxODg0IiwidHlwZSI6IkRhdGV0aW1lQXhpcyJ9XSwibGVmdCI6W3siaWQiOiIxODg5IiwidHlwZSI6IkxpbmVhckF4aXMifV0sInBsb3RfaGVpZ2h0IjoyNTAsInBsb3Rfd2lkdGgiOjc1MCwicmVuZGVyZXJzIjpbeyJpZCI6IjE4ODQiLCJ0eXBlIjoiRGF0ZXRpbWVBeGlzIn0seyJpZCI6IjE4ODgiLCJ0eXBlIjoiR3JpZCJ9LHsiaWQiOiIxODg5IiwidHlwZSI6IkxpbmVhckF4aXMifSx7ImlkIjoiMTg5MyIsInR5cGUiOiJHcmlkIn0seyJpZCI6IjE4OTgiLCJ0eXBlIjoiQm94QW5ub3RhdGlvbiJ9LHsiaWQiOiIxOTA2IiwidHlwZSI6IkdseXBoUmVuZGVyZXIifSx7ImlkIjoiMTkxMyIsInR5cGUiOiJHbHlwaFJlbmRlcmVyIn0seyJpZCI6IjE5MjAiLCJ0eXBlIjoiR2x5cGhSZW5kZXJlciJ9LHsiaWQiOiIxOTI3IiwidHlwZSI6IkdseXBoUmVuZGVyZXIifSx7ImlkIjoiMTkzMSIsInR5cGUiOiJMZWdlbmQifV0sInJpZ2h0IjpbeyJpZCI6IjE5MzEiLCJ0eXBlIjoiTGVnZW5kIn1dLCJ0aXRsZSI6eyJpZCI6IjE4NzMiLCJ0eXBlIjoiVGl0bGUifSwidG9vbGJhciI6eyJpZCI6IjE4OTciLCJ0eXBlIjoiVG9vbGJhciJ9LCJ0b29sYmFyX2xvY2F0aW9uIjoiYWJvdmUiLCJ4X3JhbmdlIjp7ImlkIjoiMTg3NiIsInR5cGUiOiJEYXRhUmFuZ2UxZCJ9LCJ4X3NjYWxlIjp7ImlkIjoiMTg4MCIsInR5cGUiOiJMaW5lYXJTY2FsZSJ9LCJ5X3JhbmdlIjp7ImlkIjoiMTg3OCIsInR5cGUiOiJEYXRhUmFuZ2UxZCJ9LCJ5X3NjYWxlIjp7ImlkIjoiMTg4MiIsInR5cGUiOiJMaW5lYXJTY2FsZSJ9fSwiaWQiOiIxODc0Iiwic3VidHlwZSI6IkZpZ3VyZSIsInR5cGUiOiJQbG90In0seyJhdHRyaWJ1dGVzIjp7ImNhbGxiYWNrIjpudWxsLCJkYXRhIjp7IngiOnsiX19uZGFycmF5X18iOiJBQUNBVnBzZGRrSUFBR2pGbmgxMlFnQUFVRFNpSFhaQ0FBQTRvNlVkZGtJQUFDQVNxUjEyUWdBQUNJR3NIWFpDQUFEdzc2OGRka0lBQU5oZXN4MTJRZ0FBd00yMkhYWkNBQUNvUExvZGRrSUFBSkNydlIxMlFnQUFlQnJCSFhaQ0FBQmdpY1FkZGtJQUFFajR4eDEyUWdBQU1HZkxIWFpDQUFBWTFzNGRka0lBQUFCRjBoMTJRZ0FBNkxQVkhYWkNBQURRSXRrZGRrSUFBTGlSM0IxMlFnQUFvQURnSFhaQ0FBQ0liK01kZGtJQUFIRGU1aDEyUWdBQVdFM3FIWFpDQUFCQXZPMGRka0lBQUNncjhSMTJRZ0FBRUpyMEhYWkNBQUQ0Q1BnZGRrSUFBT0IzK3gxMlFnQUF5T2IrSFhaQ0FBQ3dWUUllZGtJQUFKakVCUjUyUWdBQWdETUpIblpDQUFCb29nd2Vka0lBQUZBUkVCNTJRZ0FBT0lBVEhuWkNBQUFnN3hZZWRrSUFBQWhlR2g1MlFnQUE4TXdkSG5aQ0FBRFlPeUVlZGtJQUFNQ3FKQjUyUWdBQXFCa29IblpDQUFDUWlDc2Vka0lBQUhqM0xoNTJRZ0FBWUdZeUhuWkNBQUJJMVRVZWRrSUFBREJFT1I1MlFnQUFHTE04SG5aQ0FBQUFJa0FlZGtJQUFPaVFReDUyUWdBQTBQOUdIblpDQUFDNGJrb2Vka0lBQUtEZFRSNTJRZ0FBaUV4UkhuWkNBQUJ3dTFRZWRrSUFBRmdxV0I1MlFnQUFRSmxiSG5aQ0FBQW9DRjhlZGtJQUFCQjNZaDUyUWdBQStPVmxIblpDQUFEZ1ZHa2Vka0lBQU1qRGJCNTJRZ0FBc0RKd0huWkNBQUNZb1hNZWRrSUFBSUFRZHg1MlFnQUFhSDk2SG5aQ0FBQlE3bjBlZGtJQUFEaGRnUjUyUWdBQUlNeUVIblpDQUFBSU80Z2Vka0lBQVBDcGl4NTJRZ0FBMkJpUEhuWkNBQURBaDVJZWRrSUFBS2oybFI1MlFnQUFrR1daSG5aQ0FBQjQxSndlZGtJQUFHQkRvQjUyUWdBQVNMS2pIblpDQUFBd0lhY2Vka0lBQUJpUXFoNTJRZ0FBQVArdEhuWkNBQURvYmJFZWRrSUFBTkRjdEI1MlFnQUF1RXU0SG5aQ0FBQ2d1cnNlZGtJQUFJZ3B2eDUyUWdBQWNKakNIblpDQUFCWUI4WWVka0lBQUVCMnlSNTJRZ0FBS09YTUhuWkNBQUFRVk5BZWRrSUFBUGpDMHg1MlFnQUE0REhYSG5aQ0FBRElvTm9lZGtJQUFMQVAzaDUyUWdBQW1IN2hIblpDQUFDQTdlUWVka0lBQUdoYzZCNTJRZ0FBVU12ckhuWkNBQUE0T3U4ZWRrSUFBQ0NwOGg1MlFnQUFDQmoySG5aQ0FBRHdodmtlZGtJQUFOajEvQjUyUWdBQXdHUUFIM1pDQUFDbzB3TWZka0lBQUpCQ0J4OTJRZ0FBZUxFS0gzWkNBQUJnSUE0ZmRrSUFBRWlQRVI5MlFnQUFNUDRVSDNaQ0FBQVliUmdmZGtJQUFBRGNHeDkyUWdBQTZFb2ZIM1pDQUFEUXVTSWZka0lBQUxnb0poOTJRZ0FBb0pjcEgzWkNBQUNJQmkwZmRrSUFBSEIxTUI5MlFnQUFXT1F6SDNaQ0FBQkFVemNmZGtJPSIsImR0eXBlIjoiZmxvYXQ2NCIsInNoYXBlIjpbMTIxXX0sInkiOnsiX19uZGFycmF5X18iOiJBQUFBUUlQVTA3OEFBQUFnaVV6U1B3QUFBS0FaMWVVL0FBQUFZRTA4NlQ4QUFBQWdSQ3pqUHdBQUFDQ0NpOEkvQUFBQWdKR04zYjhBQUFBZ3BzM3d2d0FBQU1EdlZ2ZS9BQUFBb0JwdCtiOEFBQURBZnpiMnZ3QUFBQ0JCWCt5L0FBQUFJT0h5eWI4QUFBQWdpaGJlUHdBQUFBQVRydTQvQUFBQVFEU284ajhBQUFCQUh1WHdQd0FBQUtDdkxPUS9BQUFBQUdBNm9MOEFBQUFBb0hIb3Z3QUFBR0Q0WGZhL0FBQUFJRk9jL0w4QUFBRGdWRTc5dndBQUFHQUdLdmkvQUFBQVFBdXM3TDhBQUFCZy9nZlF2d0FBQUFCa3F0Zy9BQUFBb09lNTZEOEFBQUNBaGxUclB3QUFBR0NPcitNL0FBQUFnQld4dVQ4QUFBREFTaFRpdndBQUFJQ1dOZk8vQUFBQUFDWWIrcjhBQUFBZ0lQLzd2d0FBQU1EeisvZS9BQUFBUUdLZjdiOEFBQUFBS3pMR3Z3QUFBS0FOVCtFL0FBQUFZQUNZOEQ4QUFBRGdwY2p6UHdBQUFFRExlZkUvQUFBQUlLYkw0ejhBQUFDZ1Z4TzJ2d0FBQUtBdzErcS9BQUFBSU5sWjk3OEFBQUFBbGQ3OHZ3QUFBTUMyYXZ5L0FBQUF3Tno3OWI4QUFBQUFSNnZsdndBQUFHQThXcjQvQUFBQUlCNEo2VDhBQUFDQXMrVHlQd0FBQUtDbUEvUS9BQUFBQUx4UDd6OEFBQURna1lQYVB3QUFBSUNHK2RHL0FBQUFJTEJCN2I4QUFBQkEwTlAwdndBQUFNQUwrdlMvQUFBQTRCRUE3YjhBQUFCQTFMZkt2d0FBQUlBNnR1SS9BQUFBWVA1djlEOEFBQUJBT2pYOFB3QUFBSUQ1ZWY4L0FBQUFnSmdkL1Q4QUFBRGdra2YxUHdBQUFDQUFMdVEvQUFBQUlMUGZ1cjhBQUFBZ1RXam12d0FBQUVCQldlKy9BQUFBb0lPMTZyOEFBQUNBQ0RuYXZ3QUFBS0JoSnRNL0FBQUFnTWVMN3o4QUFBQmdDSXI0UHdBQUFDQXZ0ZjAvQUFBQW9OMWovVDhBQUFDQVF5cjNQd0FBQUtBR1orZy9BQUFBUUNxMXA3OEFBQUNBeHREb3Z3QUFBRUFnYXZPL0FBQUFBQktNODc4QUFBQ0E0RGpwdndBQUFDQ2tCcmkvQUFBQVlBUXA1VDhBQUFEZzMwbjFQd0FBQU1DWXJ2dy9BQUFBWUdQLy9qOEFBQUFBWGgvN1B3QUFBR0F6d3ZFL0FBQUFJRUhzMVQ4QUFBQmczMTNhdndBQUFPREhwTzYvQUFBQXdBRUE4cjhBQUFBZzB0SHR2d0FBQUlDUmR0Vy9BQUFBSU8rSjJEOEFBQUNnUlR2eFB3QUFBR0FVU3ZrL0FBQUFRT1RkL0Q4QUFBQmdHTzc2UHdBQUFNQWpSL00vQUFBQWdLRSszejhBQUFCQVo3elJ2d0FBQUNCU0dPMi9BQUFBWUN5dTg3OEFBQUNBeEVIeXZ3QUFBSUN3aHVXL0FBQUFJQ2RoaUQ4QUFBQUF1enJuUHdBQUFHQk82dlEvQUFBQUlJSmorajhBQUFCQXd1YjZQd0FBQUtCNndQVS9BQUFBb0YzWTV6OEFBQURBOTA2SVB3QUFBQUR5K09TL0FBQUFBUEw0NUw4PSIsImR0eXBlIjoiZmxvYXQ2NCIsInNoYXBlIjpbMTIxXX19LCJzZWxlY3RlZCI6eyJpZCI6IjE5NTgiLCJ0eXBlIjoiU2VsZWN0aW9uIn0sInNlbGVjdGlvbl9wb2xpY3kiOnsiaWQiOiIxOTU5IiwidHlwZSI6IlVuaW9uUmVuZGVyZXJzIn19LCJpZCI6IjE5MTciLCJ0eXBlIjoiQ29sdW1uRGF0YVNvdXJjZSJ9LHsiYXR0cmlidXRlcyI6eyJsaW5lX2FscGhhIjowLjEsImxpbmVfY2FwIjoicm91bmQiLCJsaW5lX2NvbG9yIjoiIzFmNzdiNCIsImxpbmVfam9pbiI6InJvdW5kIiwibGluZV93aWR0aCI6NSwieCI6eyJmaWVsZCI6IngifSwieSI6eyJmaWVsZCI6InkifX0sImlkIjoiMTkyNiIsInR5cGUiOiJMaW5lIn0seyJhdHRyaWJ1dGVzIjp7ImxpbmVfY2FwIjoicm91bmQiLCJsaW5lX2NvbG9yIjoiY3JpbXNvbiIsImxpbmVfam9pbiI6InJvdW5kIiwibGluZV93aWR0aCI6NSwieCI6eyJmaWVsZCI6IngifSwieSI6eyJmaWVsZCI6InkifX0sImlkIjoiMTkwNCIsInR5cGUiOiJMaW5lIn0seyJhdHRyaWJ1dGVzIjp7ImxpbmVfYWxwaGEiOjAuNjUsImxpbmVfY2FwIjoicm91bmQiLCJsaW5lX2NvbG9yIjoiIzk4ZGY4YSIsImxpbmVfam9pbiI6InJvdW5kIiwibGluZV93aWR0aCI6NSwieCI6eyJmaWVsZCI6IngifSwieSI6eyJmaWVsZCI6InkifX0sImlkIjoiMTkyNSIsInR5cGUiOiJMaW5lIn0seyJhdHRyaWJ1dGVzIjp7fSwiaWQiOiIxOTU5IiwidHlwZSI6IlVuaW9uUmVuZGVyZXJzIn0seyJhdHRyaWJ1dGVzIjp7fSwiaWQiOiIxOTYxIiwidHlwZSI6IlVuaW9uUmVuZGVyZXJzIn0seyJhdHRyaWJ1dGVzIjp7Im1vbnRocyI6WzAsMSwyLDMsNCw1LDYsNyw4LDksMTAsMTFdfSwiaWQiOiIxOTQ4IiwidHlwZSI6Ik1vbnRoc1RpY2tlciJ9LHsiYXR0cmlidXRlcyI6eyJudW1fbWlub3JfdGlja3MiOjUsInRpY2tlcnMiOlt7ImlkIjoiMTk0MSIsInR5cGUiOiJBZGFwdGl2ZVRpY2tlciJ9LHsiaWQiOiIxOTQyIiwidHlwZSI6IkFkYXB0aXZlVGlja2VyIn0seyJpZCI6IjE5NDMiLCJ0eXBlIjoiQWRhcHRpdmVUaWNrZXIifSx7ImlkIjoiMTk0NCIsInR5cGUiOiJEYXlzVGlja2VyIn0seyJpZCI6IjE5NDUiLCJ0eXBlIjoiRGF5c1RpY2tlciJ9LHsiaWQiOiIxOTQ2IiwidHlwZSI6IkRheXNUaWNrZXIifSx7ImlkIjoiMTk0NyIsInR5cGUiOiJEYXlzVGlja2VyIn0seyJpZCI6IjE5NDgiLCJ0eXBlIjoiTW9udGhzVGlja2VyIn0seyJpZCI6IjE5NDkiLCJ0eXBlIjoiTW9udGhzVGlja2VyIn0seyJpZCI6IjE5NTAiLCJ0eXBlIjoiTW9udGhzVGlja2VyIn0seyJpZCI6IjE5NTEiLCJ0eXBlIjoiTW9udGhzVGlja2VyIn0seyJpZCI6IjE5NTIiLCJ0eXBlIjoiWWVhcnNUaWNrZXIifV19LCJpZCI6IjE4ODUiLCJ0eXBlIjoiRGF0ZXRpbWVUaWNrZXIifSx7ImF0dHJpYnV0ZXMiOnsiZGF5cyI6WzEsMTVdfSwiaWQiOiIxOTQ3IiwidHlwZSI6IkRheXNUaWNrZXIifSx7ImF0dHJpYnV0ZXMiOnsiZGF0YV9zb3VyY2UiOnsiaWQiOiIxOTEwIiwidHlwZSI6IkNvbHVtbkRhdGFTb3VyY2UifSwiZ2x5cGgiOnsiaWQiOiIxOTExIiwidHlwZSI6IkxpbmUifSwiaG92ZXJfZ2x5cGgiOm51bGwsIm11dGVkX2dseXBoIjpudWxsLCJub25zZWxlY3Rpb25fZ2x5cGgiOnsiaWQiOiIxOTEyIiwidHlwZSI6IkxpbmUifSwic2VsZWN0aW9uX2dseXBoIjpudWxsLCJ2aWV3Ijp7ImlkIjoiMTkxNCIsInR5cGUiOiJDRFNWaWV3In19LCJpZCI6IjE5MTMiLCJ0eXBlIjoiR2x5cGhSZW5kZXJlciJ9LHsiYXR0cmlidXRlcyI6eyJsYWJlbCI6eyJ2YWx1ZSI6IlRpbWVfdjJfQXZlcmFnZXNfQmVzdCJ9LCJyZW5kZXJlcnMiOlt7ImlkIjoiMTkxMyIsInR5cGUiOiJHbHlwaFJlbmRlcmVyIn1dfSwiaWQiOiIxOTMzIiwidHlwZSI6IkxlZ2VuZEl0ZW0ifSx7ImF0dHJpYnV0ZXMiOnsiY2FsbGJhY2siOm51bGx9LCJpZCI6IjE4NzgiLCJ0eXBlIjoiRGF0YVJhbmdlMWQifV0sInJvb3RfaWRzIjpbIjE4NzQiXX0sInRpdGxlIjoiQm9rZWggQXBwbGljYXRpb24iLCJ2ZXJzaW9uIjoiMS4wLjEifX0KICAgICAgICA8L3NjcmlwdD4KICAgICAgICA8c2NyaXB0IHR5cGU9InRleHQvamF2YXNjcmlwdCI+CiAgICAgICAgICAoZnVuY3Rpb24oKSB7CiAgICAgICAgICAgIHZhciBmbiA9IGZ1bmN0aW9uKCkgewogICAgICAgICAgICAgIEJva2VoLnNhZmVseShmdW5jdGlvbigpIHsKICAgICAgICAgICAgICAgIChmdW5jdGlvbihyb290KSB7CiAgICAgICAgICAgICAgICAgIGZ1bmN0aW9uIGVtYmVkX2RvY3VtZW50KHJvb3QpIHsKICAgICAgICAgICAgICAgICAgICAKICAgICAgICAgICAgICAgICAgdmFyIGRvY3NfanNvbiA9IGRvY3VtZW50LmdldEVsZW1lbnRCeUlkKCcyMTQ0JykudGV4dENvbnRlbnQ7CiAgICAgICAgICAgICAgICAgIHZhciByZW5kZXJfaXRlbXMgPSBbeyJkb2NpZCI6IjQ2Mjk2NjdiLWM5ZDgtNDMyYy05MTA5LWJhMjcxZWFkNGRjMiIsInJvb3RzIjp7IjE4NzQiOiI3OWJmODcwYy1jNjIwLTRkMzEtODk3YS00YWZkMTliYmRjNTQifX1dOwogICAgICAgICAgICAgICAgICByb290LkJva2VoLmVtYmVkLmVtYmVkX2l0ZW1zKGRvY3NfanNvbiwgcmVuZGVyX2l0ZW1zKTsKICAgICAgICAgICAgICAgIAogICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICAgIGlmIChyb290LkJva2VoICE9PSB1bmRlZmluZWQpIHsKICAgICAgICAgICAgICAgICAgICBlbWJlZF9kb2N1bWVudChyb290KTsKICAgICAgICAgICAgICAgICAgfSBlbHNlIHsKICAgICAgICAgICAgICAgICAgICB2YXIgYXR0ZW1wdHMgPSAwOwogICAgICAgICAgICAgICAgICAgIHZhciB0aW1lciA9IHNldEludGVydmFsKGZ1bmN0aW9uKHJvb3QpIHsKICAgICAgICAgICAgICAgICAgICAgIGlmIChyb290LkJva2VoICE9PSB1bmRlZmluZWQpIHsKICAgICAgICAgICAgICAgICAgICAgICAgZW1iZWRfZG9jdW1lbnQocm9vdCk7CiAgICAgICAgICAgICAgICAgICAgICAgIGNsZWFySW50ZXJ2YWwodGltZXIpOwogICAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgICAgICAgYXR0ZW1wdHMrKzsKICAgICAgICAgICAgICAgICAgICAgIGlmIChhdHRlbXB0cyA+IDEwMCkgewogICAgICAgICAgICAgICAgICAgICAgICBjb25zb2xlLmxvZygiQm9rZWg6IEVSUk9SOiBVbmFibGUgdG8gcnVuIEJva2VoSlMgY29kZSBiZWNhdXNlIEJva2VoSlMgbGlicmFyeSBpcyBtaXNzaW5nIik7CiAgICAgICAgICAgICAgICAgICAgICAgIGNsZWFySW50ZXJ2YWwodGltZXIpOwogICAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgICAgIH0sIDEwLCByb290KQogICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICB9KSh3aW5kb3cpOwogICAgICAgICAgICAgIH0pOwogICAgICAgICAgICB9OwogICAgICAgICAgICBpZiAoZG9jdW1lbnQucmVhZHlTdGF0ZSAhPSAibG9hZGluZyIpIGZuKCk7CiAgICAgICAgICAgIGVsc2UgZG9jdW1lbnQuYWRkRXZlbnRMaXN0ZW5lcigiRE9NQ29udGVudExvYWRlZCIsIGZuKTsKICAgICAgICAgIH0pKCk7CiAgICAgICAgPC9zY3JpcHQ+CiAgICAKICA8L2JvZHk+CiAgCjwvaHRtbD4=&quot; width=&quot;790&quot; style=&quot;border:none !important;&quot; height=&quot;330&quot;></iframe>`)[0];
                popup_e194341ac9f2433c82b71f9482595645.setContent(i_frame_5c12e4dbdf28402ca44a364c57c903ce);


            marker_af4e9250c52146579f066384e525743a.bindPopup(popup_e194341ac9f2433c82b71f9482595645)
            ;




        var marker_a62433cefca84b2b93a6aa8a24859061 = L.marker(
            [41.5236, -70.6711],
            {
                icon: new L.Icon.Default()
                }
            ).addTo(map_54cb488c2c244a8daafaa65ad6b9b54b);



                var icon_e09ac17164094d5dacad308e782eb2f4 = L.AwesomeMarkers.icon({
                    icon: 'stats',
                    iconColor: 'white',
                    markerColor: 'green',
                    prefix: 'glyphicon',
                    extraClasses: 'fa-rotate-0'
                    });
                marker_a62433cefca84b2b93a6aa8a24859061.setIcon(icon_e09ac17164094d5dacad308e782eb2f4);


            var popup_1baab68c28f7418db6c0b2faa1a4d950 = L.popup({maxWidth: '2650'

            });


                var i_frame_c56b6e3fe26340cc9abf9821d72b3fc6 = $(`<iframe src=&quot;data:text/html;charset=utf-8;base64,CiAgICAKCgoKPCFET0NUWVBFIGh0bWw+CjxodG1sIGxhbmc9ImVuIj4KICAKICA8aGVhZD4KICAgIAogICAgICA8bWV0YSBjaGFyc2V0PSJ1dGYtOCI+CiAgICAgIDx0aXRsZT44NDQ3OTMwPC90aXRsZT4KICAgICAgCiAgICAgIAogICAgICAgIAogICAgICAgICAgCiAgICAgICAgPGxpbmsgcmVsPSJzdHlsZXNoZWV0IiBocmVmPSJodHRwczovL2Nkbi5weWRhdGEub3JnL2Jva2VoL3JlbGVhc2UvYm9rZWgtMS4wLjEubWluLmNzcyIgdHlwZT0idGV4dC9jc3MiIC8+CiAgICAgICAgCiAgICAgICAgCiAgICAgICAgICAKICAgICAgICA8c2NyaXB0IHR5cGU9InRleHQvamF2YXNjcmlwdCIgc3JjPSJodHRwczovL2Nkbi5weWRhdGEub3JnL2Jva2VoL3JlbGVhc2UvYm9rZWgtMS4wLjEubWluLmpzIj48L3NjcmlwdD4KICAgICAgICA8c2NyaXB0IHR5cGU9InRleHQvamF2YXNjcmlwdCI+CiAgICAgICAgICAgIEJva2VoLnNldF9sb2dfbGV2ZWwoImluZm8iKTsKICAgICAgICA8L3NjcmlwdD4KICAgICAgICAKICAgICAgCiAgICAgIAogICAgCiAgPC9oZWFkPgogIAogIAogIDxib2R5PgogICAgCiAgICAgIAogICAgICAgIAogICAgICAgICAgCiAgICAgICAgICAKICAgICAgICAgICAgCiAgICAgICAgICAgICAgPGRpdiBjbGFzcz0iYmstcm9vdCIgaWQ9Ijc0NTdjZmY3LTg0YTktNDFhNi05MTE4LTY1YTZhMDFmNTdlZCI+PC9kaXY+CiAgICAgICAgICAgIAogICAgICAgICAgCiAgICAgICAgCiAgICAgIAogICAgICAKICAgICAgICA8c2NyaXB0IHR5cGU9ImFwcGxpY2F0aW9uL2pzb24iIGlkPSIyNDE2Ij4KICAgICAgICAgIHsiOTk3NDFmYzctNDFhMi00MWJkLTgzYWYtZjFlZjNhNDU1MzM1Ijp7InJvb3RzIjp7InJlZmVyZW5jZXMiOlt7ImF0dHJpYnV0ZXMiOnsiZGF0YV9zb3VyY2UiOnsiaWQiOiIyMTk2IiwidHlwZSI6IkNvbHVtbkRhdGFTb3VyY2UifSwiZ2x5cGgiOnsiaWQiOiIyMTk3IiwidHlwZSI6IkxpbmUifSwiaG92ZXJfZ2x5cGgiOm51bGwsIm11dGVkX2dseXBoIjpudWxsLCJub25zZWxlY3Rpb25fZ2x5cGgiOnsiaWQiOiIyMTk4IiwidHlwZSI6IkxpbmUifSwic2VsZWN0aW9uX2dseXBoIjpudWxsLCJ2aWV3Ijp7ImlkIjoiMjIwMCIsInR5cGUiOiJDRFNWaWV3In19LCJpZCI6IjIxOTkiLCJ0eXBlIjoiR2x5cGhSZW5kZXJlciJ9LHsiYXR0cmlidXRlcyI6eyJkYXlzIjpbMSwxNV19LCJpZCI6IjIyMTkiLCJ0eXBlIjoiRGF5c1RpY2tlciJ9LHsiYXR0cmlidXRlcyI6eyJheGlzX2xhYmVsIjoiRGF0ZS90aW1lIiwiZm9ybWF0dGVyIjp7ImlkIjoiMjIxMCIsInR5cGUiOiJEYXRldGltZVRpY2tGb3JtYXR0ZXIifSwicGxvdCI6eyJpZCI6IjIxNDYiLCJzdWJ0eXBlIjoiRmlndXJlIiwidHlwZSI6IlBsb3QifSwidGlja2VyIjp7ImlkIjoiMjE1NyIsInR5cGUiOiJEYXRldGltZVRpY2tlciJ9fSwiaWQiOiIyMTU2IiwidHlwZSI6IkRhdGV0aW1lQXhpcyJ9LHsiYXR0cmlidXRlcyI6eyJtb250aHMiOlswLDEsMiwzLDQsNSw2LDcsOCw5LDEwLDExXX0sImlkIjoiMjIyMCIsInR5cGUiOiJNb250aHNUaWNrZXIifSx7ImF0dHJpYnV0ZXMiOnsiY2xpY2tfcG9saWN5IjoibXV0ZSIsIml0ZW1zIjpbeyJpZCI6IjIyMDQiLCJ0eXBlIjoiTGVnZW5kSXRlbSJ9LHsiaWQiOiIyMjA1IiwidHlwZSI6IkxlZ2VuZEl0ZW0ifSx7ImlkIjoiMjIwNiIsInR5cGUiOiJMZWdlbmRJdGVtIn0seyJpZCI6IjIyMDciLCJ0eXBlIjoiTGVnZW5kSXRlbSJ9XSwibG9jYXRpb24iOlswLDYwXSwicGxvdCI6eyJpZCI6IjIxNDYiLCJzdWJ0eXBlIjoiRmlndXJlIiwidHlwZSI6IlBsb3QifX0sImlkIjoiMjIwMyIsInR5cGUiOiJMZWdlbmQifSx7ImF0dHJpYnV0ZXMiOnsibGluZV9jYXAiOiJyb3VuZCIsImxpbmVfY29sb3IiOiJjcmltc29uIiwibGluZV9qb2luIjoicm91bmQiLCJsaW5lX3dpZHRoIjo1LCJ4Ijp7ImZpZWxkIjoieCJ9LCJ5Ijp7ImZpZWxkIjoieSJ9fSwiaWQiOiIyMTc2IiwidHlwZSI6IkxpbmUifSx7ImF0dHJpYnV0ZXMiOnt9LCJpZCI6IjIyMjciLCJ0eXBlIjoiVW5pb25SZW5kZXJlcnMifSx7ImF0dHJpYnV0ZXMiOnsiY2FsbGJhY2siOm51bGwsInJlbmRlcmVycyI6W3siaWQiOiIyMTk5IiwidHlwZSI6IkdseXBoUmVuZGVyZXIifV0sInRvb2x0aXBzIjpbWyJOYW1lIiwiVGltZV92Ml9IaXN0b3J5X0Jlc3QiXSxbIkJpYXMiLCItMC45MiJdLFsiU2tpbGwiLCIwLjE1Il1dfSwiaWQiOiIyMjAxIiwidHlwZSI6IkhvdmVyVG9vbCJ9LHsiYXR0cmlidXRlcyI6e30sImlkIjoiMjE2MiIsInR5cGUiOiJCYXNpY1RpY2tlciJ9LHsiYXR0cmlidXRlcyI6eyJjYWxsYmFjayI6bnVsbCwiZGF0YSI6eyJ4Ijp7Il9fbmRhcnJheV9fIjoiQUFDQVZwc2Rka0lBQUdqRm5oMTJRZ0FBVURTaUhYWkNBQUE0bzZVZGRrSUFBQ0FTcVIxMlFnQUFDSUdzSFhaQ0FBRHc3NjhkZGtJQUFOaGVzeDEyUWdBQXdNMjJIWFpDQUFDb1BMb2Rka0lBQUpDcnZSMTJRZ0FBZUJyQkhYWkNBQUJnaWNRZGRrSUFBRWo0eHgxMlFnQUFNR2ZMSFhaQ0FBQVkxczRkZGtJQUFBQkYwaDEyUWdBQTZMUFZIWFpDQUFEUUl0a2Rka0lBQUxpUjNCMTJRZ0FBb0FEZ0hYWkNBQUNJYitNZGRrSUFBSERlNWgxMlFnQUFXRTNxSFhaQ0FBQkF2TzBkZGtJQUFDZ3I4UjEyUWdBQUVKcjBIWFpDQUFENENQZ2Rka0lBQU9CMyt4MTJRZ0FBeU9iK0hYWkNBQUN3VlFJZWRrSUFBSmpFQlI1MlFnQUFnRE1KSG5aQ0FBQm9vZ3dlZGtJQUFGQVJFQjUyUWdBQU9JQVRIblpDQUFBZzd4WWVka0lBQUFoZUdoNTJRZ0FBOE13ZEhuWkNBQURZT3lFZWRrSUFBTUNxSkI1MlFnQUFxQmtvSG5aQ0FBQ1FpQ3NlZGtJQUFIajNMaDUyUWdBQVlHWXlIblpDQUFCSTFUVWVka0lBQURCRU9SNTJRZ0FBR0xNOEhuWkNBQUFBSWtBZWRrSUFBT2lRUXg1MlFnQUEwUDlHSG5aQ0FBQzRia29lZGtJQUFLRGRUUjUyUWdBQWlFeFJIblpDQUFCd3UxUWVka0lBQUZncVdCNTJRZ0FBUUpsYkhuWkNBQUFvQ0Y4ZWRrSUFBQkIzWWg1MlFnQUErT1ZsSG5aQ0FBRGdWR2tlZGtJQUFNakRiQjUyUWdBQXNESndIblpDQUFDWW9YTWVka0lBQUlBUWR4NTJRZ0FBYUg5NkhuWkNBQUJRN24wZWRrSUFBRGhkZ1I1MlFnQUFJTXlFSG5aQ0FBQUlPNGdlZGtJQUFQQ3BpeDUyUWdBQTJCaVBIblpDQUFEQWg1SWVka0lBQUtqMmxSNTJRZ0FBa0dXWkhuWkNBQUI0MUp3ZWRrSUFBR0JEb0I1MlFnQUFTTEtqSG5aQ0FBQXdJYWNlZGtJQUFCaVFxaDUyUWdBQUFQK3RIblpDQUFEb2JiRWVka0lBQU5EY3RCNTJRZ0FBdUV1NEhuWkNBQUNndXJzZWRrSUFBSWdwdng1MlFnQUFjSmpDSG5aQ0FBQllCOFllZGtJQUFFQjJ5UjUyUWdBQUtPWE1IblpDQUFBUVZOQWVka0lBQVBqQzB4NTJRZ0FBNERIWEhuWkNBQURJb05vZWRrSUFBTEFQM2g1MlFnQUFtSDdoSG5aQ0FBQ0E3ZVFlZGtJQUFHaGM2QjUyUWdBQVVNdnJIblpDQUFBNE91OGVka0lBQUNDcDhoNTJRZ0FBQ0JqMkhuWkNBQUR3aHZrZWRrSUFBTmoxL0I1MlFnQUF3R1FBSDNaQ0FBQ28wd01mZGtJQUFKQkNCeDkyUWdBQWVMRUtIM1pDQUFCZ0lBNGZka0lBQUVpUEVSOTJRZ0FBTVA0VUgzWkNBQUFZYlJnZmRrSUFBQURjR3g5MlFnQUE2RW9mSDNaQ0FBRFF1U0lmZGtJQUFMZ29KaDkyUWdBQW9KY3BIM1pDQUFDSUJpMGZka0lBQUhCMU1COTJRZ0FBV09RekgzWkNBQUJBVXpjZmRrST0iLCJkdHlwZSI6ImZsb2F0NjQiLCJzaGFwZSI6WzEyMV19LCJ5Ijp7Il9fbmRhcnJheV9fIjoiOWloY2o4TDE0RDlNTjRsQllPWFlQODNNek16TXpNdy9sa09MYk9mN3lUK0JsVU9MYk9lN1A5djVmbXE4ZEpNL1dtUTczMCtObDc5WU9iVElkcjZ2UDd4MGt4Z0VWczQvN253L05WNjYyVCtzSEZwa085L2pQOFpMTjRsQllPay83bncvTlY2NjZUKy9ueG92M1NUbVB4RllPYlRJZHVJL1lPWFFJdHY1M2o4VXJrZmhlaFRXUHhLRHdNcWhSY1kvR3kvZEpBYUJwVCtjeENDd2NtaVJ2L3ArYXJ4MGs1Zy8rbjVxdkhTVHlEOHNoeGJaenZmYlAwNWlFRmc1dE9RL2YycThkSk1ZNkQvVFRXSVFXRG5rUDgvM1UrT2xtOXcvZEpNWUJGWU8xVDkvYXJ4MGt4alVQeVBiK1g1cXZNUS8rbjVxdkhTVHFEOGJMOTBrQm9HbFAveXA4ZEpOWXNBLzIvbCthcngwMHorN1NRd0NLNGZlUDNqcEpqRUlyT2cvRGkyeW5lK243ai9oZWhTdVIrSHVQK3hSdUI2RjYray9JYkJ5YUpIdDVEK1I3WHcvTlY3aVAra21NUWlzSE5vL3BIQTlDdGVqMEQrTWJPZjdxZkhDUDZSd1BRclhvN0EvMi9sK2FyeDB3eitzSEZwa085L1hQMDVpRUZnNXRPUS9KUWFCbFVPTDdEL3VmRDgxWHJydFB4U3VSK0Y2Rk9vLytuNXF2SFNUNUQ4NnRNaDJ2cC9pUDhxaFJiYnovZHcvU2d3Q0s0Y1cwVDlXRGkyeW5lL0hQd2FCbFVPTGJNYy9qR3puKzZueDBqL0Q5U2hjajhMaFA4bDJ2cDhhTCtrLzVkQWkyL2wrOGovVmVPa21NUWowUDVodUVvUEF5dk0vRGkyeW5lK244ai92cDhaTE40bnhQL3ArYXJ4MGsvQS9qR3puKzZueDZqOHhDS3djV21UalA5OVBqWmR1RXRzL0p6RUlyQnhhMUQrSlFXRGwwQ0xiUC9Zb1hJL0M5ZVEvUHpWZXVra004RDhsQm9HVlE0dnlQdzBDSzRjVzJmUS9qOEwxS0Z5UDhqOTdGSzVINFhyeVA0UEF5cUZGdHU4L1VyZ2VoZXRSNkQ4VXJrZmhlaFRlUHhTdVIrRjZGTlkvMTZOd1BRclgwejlLREFJcmh4YlpQNUx0ZkQ4MVh1WS9XRG0weUhhKzhUODlDdGVqY0QzMlB5UGIrWDVxdlBnL0liQnlhSkh0OWo4cmh4Ylp6dmZ6UHhzdjNTUUdnZkUvbk1RZ3NISm83VC9iK1g1cXZIVG5QMlptWm1abVp1SS9KekVJckJ4YTREOWtPOTlQalpmaVB3aXNIRnBrTytjL2hldFJ1QjZGN3orV1E0dHM1L3Z6UHhGWU9iVElkdlkvVk9PbG04UWc5ajgvTlY2NlNRejBQd3dDSzRjVzJmQS9nOERLb1VXMjd6K2N4Q0N3Y21qcFA2NUg0WG9VcnVNL1ZPT2xtOFFnNEQ5TU40bEJZT1hnUHdSV0RpMnluZU0vM1NRR2daVkQ2ei9zVWJnZWhldnhQNHhzNS91cDhmUS8yczczVStPbDlUOHYzU1FHZ1pYelA5RWkyL2wrYXZBL21wbVptWm1aN1QrQmxVT0xiT2ZyUDdnZWhldFJ1T1kvRVZnNXRNaDI0ai9EOVNoY2o4TGhQMXlQd3ZVb1hPTS9KUWFCbFVPTDZEOD0iLCJkdHlwZSI6ImZsb2F0NjQiLCJzaGFwZSI6WzEyMV19fSwic2VsZWN0ZWQiOnsiaWQiOiIyMjI2IiwidHlwZSI6IlNlbGVjdGlvbiJ9LCJzZWxlY3Rpb25fcG9saWN5Ijp7ImlkIjoiMjIyNyIsInR5cGUiOiJVbmlvblJlbmRlcmVycyJ9fSwiaWQiOiIyMTc1IiwidHlwZSI6IkNvbHVtbkRhdGFTb3VyY2UifSx7ImF0dHJpYnV0ZXMiOnsic291cmNlIjp7ImlkIjoiMjE5NiIsInR5cGUiOiJDb2x1bW5EYXRhU291cmNlIn19LCJpZCI6IjIyMDAiLCJ0eXBlIjoiQ0RTVmlldyJ9LHsiYXR0cmlidXRlcyI6e30sImlkIjoiMjIyNCIsInR5cGUiOiJZZWFyc1RpY2tlciJ9LHsiYXR0cmlidXRlcyI6e30sImlkIjoiMjIzMSIsInR5cGUiOiJVbmlvblJlbmRlcmVycyJ9LHsiYXR0cmlidXRlcyI6eyJsaW5lX2FscGhhIjowLjY1LCJsaW5lX2NhcCI6InJvdW5kIiwibGluZV9jb2xvciI6IiM5NDY3YmQiLCJsaW5lX2pvaW4iOiJyb3VuZCIsImxpbmVfd2lkdGgiOjUsIngiOnsiZmllbGQiOiJ4In0sInkiOnsiZmllbGQiOiJ5In19LCJpZCI6IjIxOTciLCJ0eXBlIjoiTGluZSJ9LHsiYXR0cmlidXRlcyI6eyJsaW5lX2FscGhhIjowLjEsImxpbmVfY2FwIjoicm91bmQiLCJsaW5lX2NvbG9yIjoiIzFmNzdiNCIsImxpbmVfam9pbiI6InJvdW5kIiwibGluZV93aWR0aCI6NSwieCI6eyJmaWVsZCI6IngifSwieSI6eyJmaWVsZCI6InkifX0sImlkIjoiMjE4NCIsInR5cGUiOiJMaW5lIn0seyJhdHRyaWJ1dGVzIjp7Im1vbnRocyI6WzAsMiw0LDYsOCwxMF19LCJpZCI6IjIyMjEiLCJ0eXBlIjoiTW9udGhzVGlja2VyIn0seyJhdHRyaWJ1dGVzIjp7fSwiaWQiOiIyMjI2IiwidHlwZSI6IlNlbGVjdGlvbiJ9LHsiYXR0cmlidXRlcyI6e30sImlkIjoiMjE2NiIsInR5cGUiOiJQYW5Ub29sIn0seyJhdHRyaWJ1dGVzIjp7ImxpbmVfYWxwaGEiOjAuNjUsImxpbmVfY2FwIjoicm91bmQiLCJsaW5lX2NvbG9yIjoiI2Q2MjcyOCIsImxpbmVfam9pbiI6InJvdW5kIiwibGluZV93aWR0aCI6NSwieCI6eyJmaWVsZCI6IngifSwieSI6eyJmaWVsZCI6InkifX0sImlkIjoiMjE4MyIsInR5cGUiOiJMaW5lIn0seyJhdHRyaWJ1dGVzIjp7ImJlbG93IjpbeyJpZCI6IjIxNTYiLCJ0eXBlIjoiRGF0ZXRpbWVBeGlzIn1dLCJsZWZ0IjpbeyJpZCI6IjIxNjEiLCJ0eXBlIjoiTGluZWFyQXhpcyJ9XSwicGxvdF9oZWlnaHQiOjI1MCwicGxvdF93aWR0aCI6NzUwLCJyZW5kZXJlcnMiOlt7ImlkIjoiMjE1NiIsInR5cGUiOiJEYXRldGltZUF4aXMifSx7ImlkIjoiMjE2MCIsInR5cGUiOiJHcmlkIn0seyJpZCI6IjIxNjEiLCJ0eXBlIjoiTGluZWFyQXhpcyJ9LHsiaWQiOiIyMTY1IiwidHlwZSI6IkdyaWQifSx7ImlkIjoiMjE3MCIsInR5cGUiOiJCb3hBbm5vdGF0aW9uIn0seyJpZCI6IjIxNzgiLCJ0eXBlIjoiR2x5cGhSZW5kZXJlciJ9LHsiaWQiOiIyMTg1IiwidHlwZSI6IkdseXBoUmVuZGVyZXIifSx7ImlkIjoiMjE5MiIsInR5cGUiOiJHbHlwaFJlbmRlcmVyIn0seyJpZCI6IjIxOTkiLCJ0eXBlIjoiR2x5cGhSZW5kZXJlciJ9LHsiaWQiOiIyMjAzIiwidHlwZSI6IkxlZ2VuZCJ9XSwicmlnaHQiOlt7ImlkIjoiMjIwMyIsInR5cGUiOiJMZWdlbmQifV0sInRpdGxlIjp7ImlkIjoiMjE0NSIsInR5cGUiOiJUaXRsZSJ9LCJ0b29sYmFyIjp7ImlkIjoiMjE2OSIsInR5cGUiOiJUb29sYmFyIn0sInRvb2xiYXJfbG9jYXRpb24iOiJhYm92ZSIsInhfcmFuZ2UiOnsiaWQiOiIyMTQ4IiwidHlwZSI6IkRhdGFSYW5nZTFkIn0sInhfc2NhbGUiOnsiaWQiOiIyMTUyIiwidHlwZSI6IkxpbmVhclNjYWxlIn0sInlfcmFuZ2UiOnsiaWQiOiIyMTUwIiwidHlwZSI6IkRhdGFSYW5nZTFkIn0sInlfc2NhbGUiOnsiaWQiOiIyMTU0IiwidHlwZSI6IkxpbmVhclNjYWxlIn19LCJpZCI6IjIxNDYiLCJzdWJ0eXBlIjoiRmlndXJlIiwidHlwZSI6IlBsb3QifSx7ImF0dHJpYnV0ZXMiOnsibGFiZWwiOnsidmFsdWUiOiJTRUNPT1JBL0NOQVBTIn0sInJlbmRlcmVycyI6W3siaWQiOiIyMTg1IiwidHlwZSI6IkdseXBoUmVuZGVyZXIifV19LCJpZCI6IjIyMDUiLCJ0eXBlIjoiTGVnZW5kSXRlbSJ9LHsiYXR0cmlidXRlcyI6eyJtb250aHMiOlswLDQsOF19LCJpZCI6IjIyMjIiLCJ0eXBlIjoiTW9udGhzVGlja2VyIn0seyJhdHRyaWJ1dGVzIjp7Im1vbnRocyI6WzAsNl19LCJpZCI6IjIyMjMiLCJ0eXBlIjoiTW9udGhzVGlja2VyIn0seyJhdHRyaWJ1dGVzIjp7ImxhYmVsIjp7InZhbHVlIjoiT2JzZXJ2YXRpb25zIn0sInJlbmRlcmVycyI6W3siaWQiOiIyMTc4IiwidHlwZSI6IkdseXBoUmVuZGVyZXIifV19LCJpZCI6IjIyMDQiLCJ0eXBlIjoiTGVnZW5kSXRlbSJ9LHsiYXR0cmlidXRlcyI6eyJvdmVybGF5Ijp7ImlkIjoiMjE3MCIsInR5cGUiOiJCb3hBbm5vdGF0aW9uIn19LCJpZCI6IjIxNjciLCJ0eXBlIjoiQm94Wm9vbVRvb2wifSx7ImF0dHJpYnV0ZXMiOnsiY2FsbGJhY2siOm51bGwsImRhdGEiOnsieCI6eyJfX25kYXJyYXlfXyI6IkFBQmdpY1FkZGtJQUFFajR4eDEyUWdBQU1HZkxIWFpDQUFBZzd4WWVka0lBQUFoZUdoNTJRZ0FBOE13ZEhuWkNBQURnVkdrZWRrSUFBTWpEYkI1MlFnQUFzREp3SG5aQ0FBQ2d1cnNlZGtJQUFJZ3B2eDUyUWdBQWNKakNIblpDIiwiZHR5cGUiOiJmbG9hdDY0Iiwic2hhcGUiOlsxMl19LCJ5Ijp7Il9fbmRhcnJheV9fIjoiQUFBQXdMZXozYjhBQUFCZ1RwL2R2d0FBQUNEbGl0Mi9BQUFBSU5qSjI3OEFBQUFnQ0UvYnZ3QUFBQUE0MU5xL0FBQUFRRmRHMEw4QUFBQ0E1dS9QdndBQUFJQWVVOCsvQUFBQVlPN1p3YjhBQUFCZzd0bkJ2d0FBQUdEdTJjRy8iLCJkdHlwZSI6ImZsb2F0NjQiLCJzaGFwZSI6WzEyXX19LCJzZWxlY3RlZCI6eyJpZCI6IjIyMzAiLCJ0eXBlIjoiU2VsZWN0aW9uIn0sInNlbGVjdGlvbl9wb2xpY3kiOnsiaWQiOiIyMjMxIiwidHlwZSI6IlVuaW9uUmVuZGVyZXJzIn19LCJpZCI6IjIxODkiLCJ0eXBlIjoiQ29sdW1uRGF0YVNvdXJjZSJ9LHsiYXR0cmlidXRlcyI6eyJjYWxsYmFjayI6bnVsbCwiZGF0YSI6eyJ4Ijp7Il9fbmRhcnJheV9fIjoiQUFDQVZwc2Rka0lBQUdqRm5oMTJRZ0FBVURTaUhYWkNBQUE0bzZVZGRrSUFBQ0FTcVIxMlFnQUFDSUdzSFhaQ0FBRHc3NjhkZGtJQUFOaGVzeDEyUWdBQXdNMjJIWFpDQUFDb1BMb2Rka0lBQUpDcnZSMTJRZ0FBZUJyQkhYWkNBQUJnaWNRZGRrSUFBRWo0eHgxMlFnQUFNR2ZMSFhaQ0FBQVkxczRkZGtJQUFBQkYwaDEyUWdBQTZMUFZIWFpDQUFEUUl0a2Rka0lBQUxpUjNCMTJRZ0FBb0FEZ0hYWkNBQUNJYitNZGRrSUFBSERlNWgxMlFnQUFXRTNxSFhaQ0FBQkF2TzBkZGtJQUFDZ3I4UjEyUWdBQUVKcjBIWFpDQUFENENQZ2Rka0lBQU9CMyt4MTJRZ0FBeU9iK0hYWkNBQUN3VlFJZWRrSUFBSmpFQlI1MlFnQUFnRE1KSG5aQ0FBQm9vZ3dlZGtJQUFGQVJFQjUyUWdBQU9JQVRIblpDQUFBZzd4WWVka0lBQUFoZUdoNTJRZ0FBOE13ZEhuWkNBQURZT3lFZWRrSUFBTUNxSkI1MlFnQUFxQmtvSG5aQ0FBQ1FpQ3NlZGtJQUFIajNMaDUyUWdBQVlHWXlIblpDQUFCSTFUVWVka0lBQURCRU9SNTJRZ0FBR0xNOEhuWkNBQUFBSWtBZWRrSUFBT2lRUXg1MlFnQUEwUDlHSG5aQ0FBQzRia29lZGtJQUFLRGRUUjUyUWdBQWlFeFJIblpDQUFCd3UxUWVka0lBQUZncVdCNTJRZ0FBUUpsYkhuWkNBQUFvQ0Y4ZWRrSUFBQkIzWWg1MlFnQUErT1ZsSG5aQ0FBRGdWR2tlZGtJQUFNakRiQjUyUWdBQXNESndIblpDQUFDWW9YTWVka0lBQUlBUWR4NTJRZ0FBYUg5NkhuWkNBQUJRN24wZWRrSUFBRGhkZ1I1MlFnQUFJTXlFSG5aQ0FBQUlPNGdlZGtJQUFQQ3BpeDUyUWdBQTJCaVBIblpDQUFEQWg1SWVka0lBQUtqMmxSNTJRZ0FBa0dXWkhuWkNBQUI0MUp3ZWRrSUFBR0JEb0I1MlFnQUFTTEtqSG5aQ0FBQXdJYWNlZGtJQUFCaVFxaDUyUWdBQUFQK3RIblpDQUFEb2JiRWVka0lBQU5EY3RCNTJRZ0FBdUV1NEhuWkNBQUNndXJzZWRrSUFBSWdwdng1MlFnQUFjSmpDSG5aQ0FBQllCOFllZGtJQUFFQjJ5UjUyUWdBQUtPWE1IblpDQUFBUVZOQWVka0lBQVBqQzB4NTJRZ0FBNERIWEhuWkNBQURJb05vZWRrSUFBTEFQM2g1MlFnQUFtSDdoSG5aQ0FBQ0E3ZVFlZGtJQUFHaGM2QjUyUWdBQVVNdnJIblpDQUFBNE91OGVka0lBQUNDcDhoNTJRZ0FBQ0JqMkhuWkNBQUR3aHZrZWRrSUFBTmoxL0I1MlFnQUF3R1FBSDNaQ0FBQ28wd01mZGtJQUFKQkNCeDkyUWdBQWVMRUtIM1pDQUFCZ0lBNGZka0lBQUVpUEVSOTJRZ0FBTVA0VUgzWkNBQUFZYlJnZmRrSUFBQURjR3g5MlFnQUE2RW9mSDNaQ0FBRFF1U0lmZGtJQUFMZ29KaDkyUWdBQW9KY3BIM1pDQUFDSUJpMGZka0lBQUhCMU1COTJRZ0FBV09RekgzWkNBQUJBVXpjZmRrST0iLCJkdHlwZSI6ImZsb2F0NjQiLCJzaGFwZSI6WzEyMV19LCJ5Ijp7Il9fbmRhcnJheV9fIjoiQUFBQXdBMm14YjhBQUFDZ25wcld2d0FBQUVEcWt1Qy9BQUFBQUtLMTRiOEFBQUNnM2g3bnZ3QUFBTUFnY09lL0FBQUFRTHZ5NTc4QUFBQWdhLzdsdndBQUFHQVNwdCsvQUFBQVFLQnYxTDhBQUFBQXEyekd2d0FBQU1CUjBLTy9BQUFBZ0RvZ2hiOEFBQUFBNk9lOHZ3QUFBS0J0dHRPL0FBQUFvTXlLMnI4QUFBQWcxdkxrdndBQUFBRHhhT2UvQUFBQWdKTHk2YjhBQUFCZ1pFM3J2d0FBQUNEU1UrYS9BQUFBWUNuZTM3OEFBQUFnZ2hMV3Z3QUFBR0RoSU1pL0FBQUF3R05Pdjc4QUFBQ0FvQTdIdndBQUFLRElMTm0vQUFBQVFIS2w0YjhBQUFCQWJHRGp2d0FBQUtDOEgraS9BQUFBWUduTjU3OEFBQUJBQ0J6cHZ3QUFBQUFoN2VhL0FBQUFBUGhTMzc4QUFBQUFQM1RVdndBQUFHQk1Tc0svQUFBQTRDT05uYjhBQUFDZ2M1QTJQd0FBQU9CNmlNRy9BQUFBWUo3eTFMOEFBQUFBSFpMYnZ3QUFBR0MrNnVTL0FBQUFJRTFsNWI4QUFBQUEydi9tdndBQUFHQlNLZWUvQUFBQUlCcTE0TDhBQUFDQUFDUFV2d0FBQUFCSTNNSy9BQUFBb0Q2cWE3OEFBQUJnSzFPbFB3QUFBT0R2NDdLL0FBQUFvQkVvMHI4QUFBQWdkOGZWdndBQUFLQnhzdCsvQUFBQUlCUmY0NzhBQUFCZ2RCUGt2d0FBQUVEaVFlVy9BQUFBNExieDRMOEFBQUNnR3B2UnZ3QUFBQUFKSTZ1L0FBQUFJRmpPdkQ4QUFBQUFOSDdOUHdBQUFDRGZRTXcvQUFBQW9DcUd3RDhBQUFBZ1dxNmZQd0FBQUFBQkc5Ry9BQUFBWUZFYjJiOEFBQURnRmpYaHZ3QUFBRUEwZithL0FBQUFRTzE5NUw4QUFBREFtQ3phdndBQUFLQ0V6N0cvQUFBQUFNWHl5ajhBQUFBZ3RyYlFQd0FBQU1ENXFOQS9BQUFBb0x4U3dUOEFBQURBNXd5YXZ3QUFBRUFWY3RDL0FBQUFZQ2ZzMnI4QUFBQkFjK0hodndBQUFDQzhMZVMvQUFBQUFLTDg0YjhBQUFBZ0ZpYlh2d0FBQU1BTU84Ry9BQUFBQUxuS2xyOEFBQUFBZEplMlB3QUFBR0RUWXNFL0FBQUFvQzFQdWo4QUFBQkFkUUt6UHdBQUFLQm1hNm8vQUFBQUlHUjd5TDhBQUFEQUVVcld2d0FBQUtBdkh0Ni9BQUFBZ0hlYTNiOEFBQUJBaTRyUHZ3QUFBS0RIVjVrL0FBQUFvRDVLekQ4QUFBQWdYT1hUUHdBQUFFQThsdGsvQUFBQWdNUzIxajhBQUFCZ21KM1BQd0FBQU9BUTBNby9BQUFBd0VMUHNqOEFBQUNBYTZiQnZ3QUFBTUNWd3RDL0FBQUFnRlFQMkw4QUFBQUFhbi9WdndBQUFPRHpLc0svQUFBQXdHZmFzVDhBQUFDZ1lHTERQd0FBQUNBcmdzMC9BQUFBUUZyNXl6OEFBQUJnRlcyOVB3QUFBTUQwMlprL0FBQUFnTUZ6aUQ4QUFBREFjVzdHdndBQUFJQmJoODYvQUFBQVFEbWMxYjhBQUFEQUpHVFh2d0FBQU9BbnU4ZS9BQUFBNENlN3g3OD0iLCJkdHlwZSI6ImZsb2F0NjQiLCJzaGFwZSI6WzEyMV19fSwic2VsZWN0ZWQiOnsiaWQiOiIyMjMyIiwidHlwZSI6IlNlbGVjdGlvbiJ9LCJzZWxlY3Rpb25fcG9saWN5Ijp7ImlkIjoiMjIzMyIsInR5cGUiOiJVbmlvblJlbmRlcmVycyJ9fSwiaWQiOiIyMTk2IiwidHlwZSI6IkNvbHVtbkRhdGFTb3VyY2UifSx7ImF0dHJpYnV0ZXMiOnsicGxvdCI6eyJpZCI6IjIxNDYiLCJzdWJ0eXBlIjoiRmlndXJlIiwidHlwZSI6IlBsb3QifSwidGlja2VyIjp7ImlkIjoiMjE1NyIsInR5cGUiOiJEYXRldGltZVRpY2tlciJ9fSwiaWQiOiIyMTYwIiwidHlwZSI6IkdyaWQifSx7ImF0dHJpYnV0ZXMiOnsiZGF0YV9zb3VyY2UiOnsiaWQiOiIyMTgyIiwidHlwZSI6IkNvbHVtbkRhdGFTb3VyY2UifSwiZ2x5cGgiOnsiaWQiOiIyMTgzIiwidHlwZSI6IkxpbmUifSwiaG92ZXJfZ2x5cGgiOm51bGwsIm11dGVkX2dseXBoIjpudWxsLCJub25zZWxlY3Rpb25fZ2x5cGgiOnsiaWQiOiIyMTg0IiwidHlwZSI6IkxpbmUifSwic2VsZWN0aW9uX2dseXBoIjpudWxsLCJ2aWV3Ijp7ImlkIjoiMjE4NiIsInR5cGUiOiJDRFNWaWV3In19LCJpZCI6IjIxODUiLCJ0eXBlIjoiR2x5cGhSZW5kZXJlciJ9LHsiYXR0cmlidXRlcyI6eyJsaW5lX2FscGhhIjowLjEsImxpbmVfY2FwIjoicm91bmQiLCJsaW5lX2NvbG9yIjoiIzFmNzdiNCIsImxpbmVfam9pbiI6InJvdW5kIiwibGluZV93aWR0aCI6NSwieCI6eyJmaWVsZCI6IngifSwieSI6eyJmaWVsZCI6InkifX0sImlkIjoiMjE5MSIsInR5cGUiOiJMaW5lIn0seyJhdHRyaWJ1dGVzIjp7ImRhdGFfc291cmNlIjp7ImlkIjoiMjE3NSIsInR5cGUiOiJDb2x1bW5EYXRhU291cmNlIn0sImdseXBoIjp7ImlkIjoiMjE3NiIsInR5cGUiOiJMaW5lIn0sImhvdmVyX2dseXBoIjpudWxsLCJtdXRlZF9nbHlwaCI6bnVsbCwibm9uc2VsZWN0aW9uX2dseXBoIjp7ImlkIjoiMjE3NyIsInR5cGUiOiJMaW5lIn0sInNlbGVjdGlvbl9nbHlwaCI6bnVsbCwidmlldyI6eyJpZCI6IjIxNzkiLCJ0eXBlIjoiQ0RTVmlldyJ9fSwiaWQiOiIyMTc4IiwidHlwZSI6IkdseXBoUmVuZGVyZXIifSx7ImF0dHJpYnV0ZXMiOnsiYmFzZSI6NjAsIm1hbnRpc3NhcyI6WzEsMiw1LDEwLDE1LDIwLDMwXSwibWF4X2ludGVydmFsIjoxODAwMDAwLjAsIm1pbl9pbnRlcnZhbCI6MTAwMC4wLCJudW1fbWlub3JfdGlja3MiOjB9LCJpZCI6IjIyMTQiLCJ0eXBlIjoiQWRhcHRpdmVUaWNrZXIifSx7ImF0dHJpYnV0ZXMiOnsiZGF5cyI6WzEsOCwxNSwyMl19LCJpZCI6IjIyMTgiLCJ0eXBlIjoiRGF5c1RpY2tlciJ9LHsiYXR0cmlidXRlcyI6e30sImlkIjoiMjE2OCIsInR5cGUiOiJSZXNldFRvb2wifSx7ImF0dHJpYnV0ZXMiOnsibGluZV9hbHBoYSI6MC4xLCJsaW5lX2NhcCI6InJvdW5kIiwibGluZV9jb2xvciI6IiMxZjc3YjQiLCJsaW5lX2pvaW4iOiJyb3VuZCIsImxpbmVfd2lkdGgiOjUsIngiOnsiZmllbGQiOiJ4In0sInkiOnsiZmllbGQiOiJ5In19LCJpZCI6IjIxNzciLCJ0eXBlIjoiTGluZSJ9LHsiYXR0cmlidXRlcyI6eyJsYWJlbCI6eyJ2YWx1ZSI6IlRpbWVfdjJfSGlzdG9yeV9CZXN0In0sInJlbmRlcmVycyI6W3siaWQiOiIyMTk5IiwidHlwZSI6IkdseXBoUmVuZGVyZXIifV19LCJpZCI6IjIyMDciLCJ0eXBlIjoiTGVnZW5kSXRlbSJ9LHsiYXR0cmlidXRlcyI6eyJwbG90IjpudWxsLCJ0ZXh0IjoiODQ0NzkzMCJ9LCJpZCI6IjIxNDUiLCJ0eXBlIjoiVGl0bGUifSx7ImF0dHJpYnV0ZXMiOnsiY2FsbGJhY2siOm51bGx9LCJpZCI6IjIxNTAiLCJ0eXBlIjoiRGF0YVJhbmdlMWQifSx7ImF0dHJpYnV0ZXMiOnsiZGF5cyI6WzEsMiwzLDQsNSw2LDcsOCw5LDEwLDExLDEyLDEzLDE0LDE1LDE2LDE3LDE4LDE5LDIwLDIxLDIyLDIzLDI0LDI1LDI2LDI3LDI4LDI5LDMwLDMxXX0sImlkIjoiMjIxNiIsInR5cGUiOiJEYXlzVGlja2VyIn0seyJhdHRyaWJ1dGVzIjp7Im1hbnRpc3NhcyI6WzEsMiw1XSwibWF4X2ludGVydmFsIjo1MDAuMCwibnVtX21pbm9yX3RpY2tzIjowfSwiaWQiOiIyMjEzIiwidHlwZSI6IkFkYXB0aXZlVGlja2VyIn0seyJhdHRyaWJ1dGVzIjp7InNvdXJjZSI6eyJpZCI6IjIxNzUiLCJ0eXBlIjoiQ29sdW1uRGF0YVNvdXJjZSJ9fSwiaWQiOiIyMTc5IiwidHlwZSI6IkNEU1ZpZXcifSx7ImF0dHJpYnV0ZXMiOnt9LCJpZCI6IjIyMTAiLCJ0eXBlIjoiRGF0ZXRpbWVUaWNrRm9ybWF0dGVyIn0seyJhdHRyaWJ1dGVzIjp7ImNhbGxiYWNrIjpudWxsLCJyZW5kZXJlcnMiOlt7ImlkIjoiMjE4NSIsInR5cGUiOiJHbHlwaFJlbmRlcmVyIn1dLCJ0b29sdGlwcyI6W1siTmFtZSIsIlNFQ09PUkEvQ05BUFMiXSxbIkJpYXMiLCItMS4xNCJdLFsiU2tpbGwiLCIwLjM1Il1dfSwiaWQiOiIyMTg3IiwidHlwZSI6IkhvdmVyVG9vbCJ9LHsiYXR0cmlidXRlcyI6eyJzb3VyY2UiOnsiaWQiOiIyMTg5IiwidHlwZSI6IkNvbHVtbkRhdGFTb3VyY2UifX0sImlkIjoiMjE5MyIsInR5cGUiOiJDRFNWaWV3In0seyJhdHRyaWJ1dGVzIjp7ImRpbWVuc2lvbiI6MSwicGxvdCI6eyJpZCI6IjIxNDYiLCJzdWJ0eXBlIjoiRmlndXJlIiwidHlwZSI6IlBsb3QifSwidGlja2VyIjp7ImlkIjoiMjE2MiIsInR5cGUiOiJCYXNpY1RpY2tlciJ9fSwiaWQiOiIyMTY1IiwidHlwZSI6IkdyaWQifSx7ImF0dHJpYnV0ZXMiOnsibnVtX21pbm9yX3RpY2tzIjo1LCJ0aWNrZXJzIjpbeyJpZCI6IjIyMTMiLCJ0eXBlIjoiQWRhcHRpdmVUaWNrZXIifSx7ImlkIjoiMjIxNCIsInR5cGUiOiJBZGFwdGl2ZVRpY2tlciJ9LHsiaWQiOiIyMjE1IiwidHlwZSI6IkFkYXB0aXZlVGlja2VyIn0seyJpZCI6IjIyMTYiLCJ0eXBlIjoiRGF5c1RpY2tlciJ9LHsiaWQiOiIyMjE3IiwidHlwZSI6IkRheXNUaWNrZXIifSx7ImlkIjoiMjIxOCIsInR5cGUiOiJEYXlzVGlja2VyIn0seyJpZCI6IjIyMTkiLCJ0eXBlIjoiRGF5c1RpY2tlciJ9LHsiaWQiOiIyMjIwIiwidHlwZSI6Ik1vbnRoc1RpY2tlciJ9LHsiaWQiOiIyMjIxIiwidHlwZSI6Ik1vbnRoc1RpY2tlciJ9LHsiaWQiOiIyMjIyIiwidHlwZSI6Ik1vbnRoc1RpY2tlciJ9LHsiaWQiOiIyMjIzIiwidHlwZSI6Ik1vbnRoc1RpY2tlciJ9LHsiaWQiOiIyMjI0IiwidHlwZSI6IlllYXJzVGlja2VyIn1dfSwiaWQiOiIyMTU3IiwidHlwZSI6IkRhdGV0aW1lVGlja2VyIn0seyJhdHRyaWJ1dGVzIjp7ImJvdHRvbV91bml0cyI6InNjcmVlbiIsImZpbGxfYWxwaGEiOnsidmFsdWUiOjAuNX0sImZpbGxfY29sb3IiOnsidmFsdWUiOiJsaWdodGdyZXkifSwibGVmdF91bml0cyI6InNjcmVlbiIsImxldmVsIjoib3ZlcmxheSIsImxpbmVfYWxwaGEiOnsidmFsdWUiOjEuMH0sImxpbmVfY29sb3IiOnsidmFsdWUiOiJibGFjayJ9LCJsaW5lX2Rhc2giOls0LDRdLCJsaW5lX3dpZHRoIjp7InZhbHVlIjoyfSwicGxvdCI6bnVsbCwicmVuZGVyX21vZGUiOiJjc3MiLCJyaWdodF91bml0cyI6InNjcmVlbiIsInRvcF91bml0cyI6InNjcmVlbiJ9LCJpZCI6IjIxNzAiLCJ0eXBlIjoiQm94QW5ub3RhdGlvbiJ9LHsiYXR0cmlidXRlcyI6eyJsaW5lX2FscGhhIjowLjY1LCJsaW5lX2NhcCI6InJvdW5kIiwibGluZV9jb2xvciI6IiNmZjk4OTYiLCJsaW5lX2pvaW4iOiJyb3VuZCIsImxpbmVfd2lkdGgiOjUsIngiOnsiZmllbGQiOiJ4In0sInkiOnsiZmllbGQiOiJ5In19LCJpZCI6IjIxOTAiLCJ0eXBlIjoiTGluZSJ9LHsiYXR0cmlidXRlcyI6eyJkYXlzIjpbMSw0LDcsMTAsMTMsMTYsMTksMjIsMjUsMjhdfSwiaWQiOiIyMjE3IiwidHlwZSI6IkRheXNUaWNrZXIifSx7ImF0dHJpYnV0ZXMiOnsiYWN0aXZlX2RyYWciOiJhdXRvIiwiYWN0aXZlX2luc3BlY3QiOiJhdXRvIiwiYWN0aXZlX211bHRpIjpudWxsLCJhY3RpdmVfc2Nyb2xsIjoiYXV0byIsImFjdGl2ZV90YXAiOiJhdXRvIiwidG9vbHMiOlt7ImlkIjoiMjE2NiIsInR5cGUiOiJQYW5Ub29sIn0seyJpZCI6IjIxNjciLCJ0eXBlIjoiQm94Wm9vbVRvb2wifSx7ImlkIjoiMjE2OCIsInR5cGUiOiJSZXNldFRvb2wifSx7ImlkIjoiMjE4MCIsInR5cGUiOiJIb3ZlclRvb2wifSx7ImlkIjoiMjE4NyIsInR5cGUiOiJIb3ZlclRvb2wifSx7ImlkIjoiMjE5NCIsInR5cGUiOiJIb3ZlclRvb2wifSx7ImlkIjoiMjIwMSIsInR5cGUiOiJIb3ZlclRvb2wifV19LCJpZCI6IjIxNjkiLCJ0eXBlIjoiVG9vbGJhciJ9LHsiYXR0cmlidXRlcyI6eyJjYWxsYmFjayI6bnVsbCwicmVuZGVyZXJzIjpbeyJpZCI6IjIxNzgiLCJ0eXBlIjoiR2x5cGhSZW5kZXJlciJ9XSwidG9vbHRpcHMiOltbIk5hbWUiLCJPYnNlcnZhdGlvbnMiXSxbIkJpYXMiLCJOQSJdLFsiU2tpbGwiLCJOQSJdXX0sImlkIjoiMjE4MCIsInR5cGUiOiJIb3ZlclRvb2wifSx7ImF0dHJpYnV0ZXMiOnsiZGF0YV9zb3VyY2UiOnsiaWQiOiIyMTg5IiwidHlwZSI6IkNvbHVtbkRhdGFTb3VyY2UifSwiZ2x5cGgiOnsiaWQiOiIyMTkwIiwidHlwZSI6IkxpbmUifSwiaG92ZXJfZ2x5cGgiOm51bGwsIm11dGVkX2dseXBoIjpudWxsLCJub25zZWxlY3Rpb25fZ2x5cGgiOnsiaWQiOiIyMTkxIiwidHlwZSI6IkxpbmUifSwic2VsZWN0aW9uX2dseXBoIjpudWxsLCJ2aWV3Ijp7ImlkIjoiMjE5MyIsInR5cGUiOiJDRFNWaWV3In19LCJpZCI6IjIxOTIiLCJ0eXBlIjoiR2x5cGhSZW5kZXJlciJ9LHsiYXR0cmlidXRlcyI6eyJzb3VyY2UiOnsiaWQiOiIyMTgyIiwidHlwZSI6IkNvbHVtbkRhdGFTb3VyY2UifX0sImlkIjoiMjE4NiIsInR5cGUiOiJDRFNWaWV3In0seyJhdHRyaWJ1dGVzIjp7ImJhc2UiOjI0LCJtYW50aXNzYXMiOlsxLDIsNCw2LDgsMTJdLCJtYXhfaW50ZXJ2YWwiOjQzMjAwMDAwLjAsIm1pbl9pbnRlcnZhbCI6MzYwMDAwMC4wLCJudW1fbWlub3JfdGlja3MiOjB9LCJpZCI6IjIyMTUiLCJ0eXBlIjoiQWRhcHRpdmVUaWNrZXIifSx7ImF0dHJpYnV0ZXMiOnsibGFiZWwiOnsidmFsdWUiOiJUaW1lX3YyX0F2ZXJhZ2VzX0Jlc3QifSwicmVuZGVyZXJzIjpbeyJpZCI6IjIxOTIiLCJ0eXBlIjoiR2x5cGhSZW5kZXJlciJ9XX0sImlkIjoiMjIwNiIsInR5cGUiOiJMZWdlbmRJdGVtIn0seyJhdHRyaWJ1dGVzIjp7ImF4aXNfbGFiZWwiOiJXYXRlciBIZWlnaHQgKG0pIiwiZm9ybWF0dGVyIjp7ImlkIjoiMjIxMiIsInR5cGUiOiJCYXNpY1RpY2tGb3JtYXR0ZXIifSwicGxvdCI6eyJpZCI6IjIxNDYiLCJzdWJ0eXBlIjoiRmlndXJlIiwidHlwZSI6IlBsb3QifSwidGlja2VyIjp7ImlkIjoiMjE2MiIsInR5cGUiOiJCYXNpY1RpY2tlciJ9fSwiaWQiOiIyMTYxIiwidHlwZSI6IkxpbmVhckF4aXMifSx7ImF0dHJpYnV0ZXMiOnt9LCJpZCI6IjIyMTIiLCJ0eXBlIjoiQmFzaWNUaWNrRm9ybWF0dGVyIn0seyJhdHRyaWJ1dGVzIjp7ImxpbmVfYWxwaGEiOjAuMSwibGluZV9jYXAiOiJyb3VuZCIsImxpbmVfY29sb3IiOiIjMWY3N2I0IiwibGluZV9qb2luIjoicm91bmQiLCJsaW5lX3dpZHRoIjo1LCJ4Ijp7ImZpZWxkIjoieCJ9LCJ5Ijp7ImZpZWxkIjoieSJ9fSwiaWQiOiIyMTk4IiwidHlwZSI6IkxpbmUifSx7ImF0dHJpYnV0ZXMiOnt9LCJpZCI6IjIxNTIiLCJ0eXBlIjoiTGluZWFyU2NhbGUifSx7ImF0dHJpYnV0ZXMiOnt9LCJpZCI6IjIyMjgiLCJ0eXBlIjoiU2VsZWN0aW9uIn0seyJhdHRyaWJ1dGVzIjp7ImNhbGxiYWNrIjpudWxsLCJkYXRhIjp7IngiOnsiX19uZGFycmF5X18iOiJBQUNBVnBzZGRrSUFBR2pGbmgxMlFnQUFVRFNpSFhaQ0FBQTRvNlVkZGtJQUFDQVNxUjEyUWdBQUNJR3NIWFpDQUFEdzc2OGRka0lBQU5oZXN4MTJRZ0FBd00yMkhYWkNBQUNvUExvZGRrSUFBSkNydlIxMlFnQUFlQnJCSFhaQ0FBQmdpY1FkZGtJQUFFajR4eDEyUWdBQU1HZkxIWFpDQUFBWTFzNGRka0lBQUFCRjBoMTJRZ0FBNkxQVkhYWkNBQURRSXRrZGRrSUFBTGlSM0IxMlFnQUFvQURnSFhaQ0FBQ0liK01kZGtJQUFIRGU1aDEyUWdBQVdFM3FIWFpDQUFBQUlrQWVka0lBQU9pUVF4NTJRZ0FBMFA5R0huWkNBQUM0YmtvZWRrSUFBS0RkVFI1MlFnQUFpRXhSSG5aQ0FBQnd1MVFlZGtJQUFGZ3FXQjUyUWdBQVFKbGJIblpDQUFBb0NGOGVka0lBQUJCM1loNTJRZ0FBK09WbEhuWkNBQURnVkdrZWRrSUFBTWpEYkI1MlFnQUFzREp3SG5aQ0FBQ1lvWE1lZGtJQUFJQVFkeDUyUWdBQWFIOTZIblpDQUFCUTduMGVka0lBQURoZGdSNTJRZ0FBSU15RUhuWkNBQUFJTzRnZWRrSUFBUENwaXg1MlFnQUEyQmlQSG5aQ0FBREFoNUllZGtJQUFLajJsUjUyUWdBQWtHV1pIblpDQUFCNDFKd2Vka0lBQUdCRG9CNTJRZ0FBU0xLakhuWkNBQUF3SWFjZWRrSUFBQmlRcWg1MlFnQUFBUCt0SG5aQ0FBRG9iYkVlZGtJQUFORGN0QjUyUWdBQXVFdTRIblpDQUFDZ3Vyc2Vka0lBQUlncHZ4NTJRZ0FBY0pqQ0huWkNBQUJZQjhZZWRrSUFBRUIyeVI1MlFnQUFLT1hNSG5aQ0FBQVFWTkFlZGtJQUFQakMweDUyUWdBQTRESFhIblpDQUFESW9Ob2Vka0lBQUxBUDNoNTJRZ0FBbUg3aEhuWkNBQUNBN2VRZWRrSUFBR2hjNkI1MlFnQUFVTXZySG5aQ0FBQTRPdThlZGtJQUFDQ3A4aDUyUWdBQUNCajJIblpDQUFEd2h2a2Vka0lBQU5qMS9CNTJRZ0FBd0dRQUgzWkNBQUNvMHdNZmRrSUFBSkJDQng5MlFnQUFlTEVLSDNaQ0FBQmdJQTRmZGtJQUFFaVBFUjkyUWdBQU1QNFVIM1pDQUFBWWJSZ2Zka0lBQUFEY0d4OTJRZ0FBNkVvZkgzWkNBQURRdVNJZmRrSUFBTGdvSmg5MlFnQUFvSmNwSDNaQ0FBQ0lCaTBmZGtJQUFIQjFNQjkyUWdBQVdPUXpIM1pDIiwiZHR5cGUiOiJmbG9hdDY0Iiwic2hhcGUiOls5Nl19LCJ5Ijp7Il9fbmRhcnJheV9fIjoiQUFBQUlJdm53YjhBQUFEQUxqTy92d0FBQUVCSGw3cS9BQUFBd0YvN3RiOEFBQURndVJYZ3Z3QUFBT0FIYk8yL0FBQUE0Q3BoOWI4QUFBQ0FHWER5dndBQUFBQVEvdTYvQUFBQUlPMGI2YjhBQUFBQWZ4VGp2d0FBQU9BaEd0cS9BQUFBWUlzV3pMOEFBQUNneDFhdXZ3QUFBQ0JQMXJrL0FBQUFnQUMyMEQ4QUFBQUFtSFhSdndBQUFFQ1kwT20vQUFBQVFESno5YjhBQUFCZ1lVSHp2d0FBQUlDUUQvRy9BQUFBWUgrNzdiOEFBQUNBbmN6c3Z3QUFBS0M3M2V1L0FBQUFJTEltd3I4QUFBQmdNTnVndndBQUFPQXpjck0vQUFBQUFBQ3B4ejhBQUFEZ1pETEV2d0FBQUdCeUErQy9BQUFBb0V2NjZyOEFBQUJnL2tudXZ3QUFBSURZelBDL0FBQUE0TEYwOHI4QUFBQUFLd0RvdndBQUFLRGtMZGEvQUFBQTRHVWtyVDhBQUFEQWVSRE5Qd0FBQUNEdGE5ay9BQUFBb000bjRqOEFBQUNBbEJyVVB3QUFBS0JkTEs4L0FBQUFJUHFleUw4QUFBRGdFbjdodndBQUFDQm4xT3kvQUFBQXdGMFY5TDhBQUFDZ1d6cnF2d0FBQUVEM2s5aS9BQUFBWUVSbXFqOEFBQUNBZXpxK1B3QUFBR0Rxb01jL0FBQUFnRXNTMEQ4QUFBQ2dOam0wUHdBQUFLREExcmUvQUFBQUFLNzUwTDhBQUFDQW1iVGt2d0FBQUFBdWR2Qy9BQUFBUUErUzlyOEFBQUNBM3hmd3Z3QUFBS0JmTytPL0FBQUFvQUFjeWI4QUFBQWdqdU9vdndBQUFDQnpWTGsvQUFBQW9GYU56ejhBQUFDQXVtUE1Qd0FBQUdBZU9zay9BQUFBUUlJUXhqOEFBQUJBYzlEU3Z3QUFBTUNUVk9pL0FBQUFBSGVnODc4QUFBRGdPZERzdndBQUFPQ0ZYK0svQUFBQUFFZTd6NzhBQUFEQU8wSER2d0FBQU1EQkhLdS9BQUFBWUd2THBqOEFBQURBaGlDNVB3QUFBT0NyYmNNL0FBQUFZQlJMeWo4QUFBQ0FHN1RSdndBQUFLRGdSdWkvQUFBQXdOblo4NzhBQUFDZ2V0cnd2d0FBQUNBM3R1dS9BQUFBNEhpMzViOEFBQUFnYVZuaHZ3QUFBT0N5OXRtL0FBQUFnSk02MGI4QUFBQkFXOE84dndBQUFBQXZ4NlkvQUFBQUlFWEZ5VDhBQUFCQTgyRFJ2d0FBQUtCRTB1ZS9BQUFBd0FkNjg3OEFBQURBQjNyenZ3QUFBTUFIZXZPLyIsImR0eXBlIjoiZmxvYXQ2NCIsInNoYXBlIjpbOTZdfX0sInNlbGVjdGVkIjp7ImlkIjoiMjIyOCIsInR5cGUiOiJTZWxlY3Rpb24ifSwic2VsZWN0aW9uX3BvbGljeSI6eyJpZCI6IjIyMjkiLCJ0eXBlIjoiVW5pb25SZW5kZXJlcnMifX0sImlkIjoiMjE4MiIsInR5cGUiOiJDb2x1bW5EYXRhU291cmNlIn0seyJhdHRyaWJ1dGVzIjp7ImNhbGxiYWNrIjpudWxsLCJyZW5kZXJlcnMiOlt7ImlkIjoiMjE5MiIsInR5cGUiOiJHbHlwaFJlbmRlcmVyIn1dLCJ0b29sdGlwcyI6W1siTmFtZSIsIlRpbWVfdjJfQXZlcmFnZXNfQmVzdCJdLFsiQmlhcyIsIi0xLjM2Il0sWyJTa2lsbCIsIjAuMTciXV19LCJpZCI6IjIxOTQiLCJ0eXBlIjoiSG92ZXJUb29sIn0seyJhdHRyaWJ1dGVzIjp7fSwiaWQiOiIyMjMyIiwidHlwZSI6IlNlbGVjdGlvbiJ9LHsiYXR0cmlidXRlcyI6e30sImlkIjoiMjIzMyIsInR5cGUiOiJVbmlvblJlbmRlcmVycyJ9LHsiYXR0cmlidXRlcyI6e30sImlkIjoiMjE1NCIsInR5cGUiOiJMaW5lYXJTY2FsZSJ9LHsiYXR0cmlidXRlcyI6e30sImlkIjoiMjIyOSIsInR5cGUiOiJVbmlvblJlbmRlcmVycyJ9LHsiYXR0cmlidXRlcyI6eyJjYWxsYmFjayI6bnVsbH0sImlkIjoiMjE0OCIsInR5cGUiOiJEYXRhUmFuZ2UxZCJ9LHsiYXR0cmlidXRlcyI6e30sImlkIjoiMjIzMCIsInR5cGUiOiJTZWxlY3Rpb24ifV0sInJvb3RfaWRzIjpbIjIxNDYiXX0sInRpdGxlIjoiQm9rZWggQXBwbGljYXRpb24iLCJ2ZXJzaW9uIjoiMS4wLjEifX0KICAgICAgICA8L3NjcmlwdD4KICAgICAgICA8c2NyaXB0IHR5cGU9InRleHQvamF2YXNjcmlwdCI+CiAgICAgICAgICAoZnVuY3Rpb24oKSB7CiAgICAgICAgICAgIHZhciBmbiA9IGZ1bmN0aW9uKCkgewogICAgICAgICAgICAgIEJva2VoLnNhZmVseShmdW5jdGlvbigpIHsKICAgICAgICAgICAgICAgIChmdW5jdGlvbihyb290KSB7CiAgICAgICAgICAgICAgICAgIGZ1bmN0aW9uIGVtYmVkX2RvY3VtZW50KHJvb3QpIHsKICAgICAgICAgICAgICAgICAgICAKICAgICAgICAgICAgICAgICAgdmFyIGRvY3NfanNvbiA9IGRvY3VtZW50LmdldEVsZW1lbnRCeUlkKCcyNDE2JykudGV4dENvbnRlbnQ7CiAgICAgICAgICAgICAgICAgIHZhciByZW5kZXJfaXRlbXMgPSBbeyJkb2NpZCI6Ijk5NzQxZmM3LTQxYTItNDFiZC04M2FmLWYxZWYzYTQ1NTMzNSIsInJvb3RzIjp7IjIxNDYiOiI3NDU3Y2ZmNy04NGE5LTQxYTYtOTExOC02NWE2YTAxZjU3ZWQifX1dOwogICAgICAgICAgICAgICAgICByb290LkJva2VoLmVtYmVkLmVtYmVkX2l0ZW1zKGRvY3NfanNvbiwgcmVuZGVyX2l0ZW1zKTsKICAgICAgICAgICAgICAgIAogICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICAgIGlmIChyb290LkJva2VoICE9PSB1bmRlZmluZWQpIHsKICAgICAgICAgICAgICAgICAgICBlbWJlZF9kb2N1bWVudChyb290KTsKICAgICAgICAgICAgICAgICAgfSBlbHNlIHsKICAgICAgICAgICAgICAgICAgICB2YXIgYXR0ZW1wdHMgPSAwOwogICAgICAgICAgICAgICAgICAgIHZhciB0aW1lciA9IHNldEludGVydmFsKGZ1bmN0aW9uKHJvb3QpIHsKICAgICAgICAgICAgICAgICAgICAgIGlmIChyb290LkJva2VoICE9PSB1bmRlZmluZWQpIHsKICAgICAgICAgICAgICAgICAgICAgICAgZW1iZWRfZG9jdW1lbnQocm9vdCk7CiAgICAgICAgICAgICAgICAgICAgICAgIGNsZWFySW50ZXJ2YWwodGltZXIpOwogICAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgICAgICAgYXR0ZW1wdHMrKzsKICAgICAgICAgICAgICAgICAgICAgIGlmIChhdHRlbXB0cyA+IDEwMCkgewogICAgICAgICAgICAgICAgICAgICAgICBjb25zb2xlLmxvZygiQm9rZWg6IEVSUk9SOiBVbmFibGUgdG8gcnVuIEJva2VoSlMgY29kZSBiZWNhdXNlIEJva2VoSlMgbGlicmFyeSBpcyBtaXNzaW5nIik7CiAgICAgICAgICAgICAgICAgICAgICAgIGNsZWFySW50ZXJ2YWwodGltZXIpOwogICAgICAgICAgICAgICAgICAgICAgfQogICAgICAgICAgICAgICAgICAgIH0sIDEwLCByb290KQogICAgICAgICAgICAgICAgICB9CiAgICAgICAgICAgICAgICB9KSh3aW5kb3cpOwogICAgICAgICAgICAgIH0pOwogICAgICAgICAgICB9OwogICAgICAgICAgICBpZiAoZG9jdW1lbnQucmVhZHlTdGF0ZSAhPSAibG9hZGluZyIpIGZuKCk7CiAgICAgICAgICAgIGVsc2UgZG9jdW1lbnQuYWRkRXZlbnRMaXN0ZW5lcigiRE9NQ29udGVudExvYWRlZCIsIGZuKTsKICAgICAgICAgIH0pKCk7CiAgICAgICAgPC9zY3JpcHQ+CiAgICAKICA8L2JvZHk+CiAgCjwvaHRtbD4=&quot; width=&quot;790&quot; style=&quot;border:none !important;&quot; height=&quot;330&quot;></iframe>`)[0];
                popup_1baab68c28f7418db6c0b2faa1a4d950.setContent(i_frame_c56b6e3fe26340cc9abf9821d72b3fc6);


            marker_a62433cefca84b2b93a6aa8a24859061.bindPopup(popup_1baab68c28f7418db6c0b2faa1a4d950)
            ;




            var layer_control_e34bfdbcd37e491bb2212c3ec9f522b3 = {
                base_layers : { &quot;openstreetmap&quot; : tile_layer_378680a0ecc34181a212e41b10fee026, },
                overlays : { &quot;Cluster&quot; : marker_cluster_b7855381e5c049d9b9e9f1975d065c66, }
                };
            L.control.layers(
                layer_control_e34bfdbcd37e491bb2212c3ec9f522b3.base_layers,
                layer_control_e34bfdbcd37e491bb2212c3ec9f522b3.overlays,
                {position: 'topright',
                 collapsed: true,
                 autoZIndex: true
                }).addTo(map_54cb488c2c244a8daafaa65ad6b9b54b);


</script>" style="width: 100%; height: 750px; border: none"></iframe>


<br>
Right click and choose Save link as... to
[download](https://raw.githubusercontent.com/ioos/notebooks_demos/master/notebooks/2018-03-15-ssh-skillscore.ipynb)
this notebook, or click [here](https://binder.pangeo.io/v2/gh/ioos/notebooks_demos/master?filepath=notebooks/2018-03-15-ssh-skillscore.ipynb) to run a live instance of this notebook.