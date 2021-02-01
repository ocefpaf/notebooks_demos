---
title: "Read realtime data from IOOS Sensor Map via ERDDAP tabledap"
layout: notebook

---

Web Map Services are a great way to find data you may be looking for in a particular geographic area.

Suppose you were exploring the [IOOS Sensor Map](https://via.hypothes.is/https://sensors.ioos.us/#map),
and after selecting Significant Wave Height,
had selected buoy 44011 on George's Bank:

![2017-03-27_16-21-08](https://cloud.githubusercontent.com/assets/1872600/24376518/213e6c4c-130a-11e7-9744-2f23e9660adf.png)

You click the `ERDDAP` link and generate a URL to download the data as `CSV` ![2017-03-27_16-24-35](https://cloud.githubusercontent.com/assets/1872600/24376521/2377afc8-130a-11e7-9a3b-c1c46e43d20d.png).

You notice that the URL that is generated

[`https://erddap.axiomdatascience.com/erddap/tabledap/sensor_service.csvp?time,depth,station,parameter,unit,value&time>=2017-02-27T12:00:00Z&station="urn:ioos:station:wmo:44011"&parameter="Significant Wave Height"&unit="m"`](http://erddap.axiomdatascience.com/erddap/tabledap/sensor_service.csvp?time,depth,station,parameter,unit,value&time>=2017-02-27T12:00:00Z&station="urn:ioos:station:wmo:44011"&parameter="Significant Wave Height"&unit="m")

is fairly easy to understand,
and that a program could construct that URL fairly easily.
Let's explore how that could work...

<div class="prompt input_prompt">
In&nbsp;[1]:
</div>

```python
import requests

try:
    from urllib.parse import urlencode
except ImportError:
    from urllib import urlencode


def encode_erddap(urlbase, fname, columns, params):
    """
    urlbase: the base string for the endpoint
             (e.g.: https://erddap.axiomdatascience.com/erddap/tabledap).
    fname: the data source (e.g.: `sensor_service`) and the response (e.g.: `.csvp` for CSV).
    columns: the columns of the return table.
    params: the parameters for the query.

    Returns a valid ERDDAP endpoint.
    """
    urlbase = urlbase.rstrip("/")
    if not urlbase.lower().startswith(("http:", "https:")):
        msg = "Expected valid URL but got {}".format
        raise ValueError(msg(urlbase))

    columns = ",".join(columns)
    params = urlencode(params)
    endpoint = "{urlbase}/{fname}?{columns}&{params}".format

    url = endpoint(urlbase=urlbase, fname=fname, columns=columns, params=params)
    r = requests.get(url)
    r.raise_for_status()
    return url
```

Using the function we defined above, we can now bypass the forms and get the data by generating the URL "by hand". Below we have a query for `Significant Wave Height` from buoy `44011`, a buoy on George's Bank off the coast of Cape Cod, MA, starting at the beginning of the year 2017.

\* For more information on how to use tabledap, please check the [NOAA ERDDAP documentation](https://via.hypothes.is/http://coastwatch.pfeg.noaa.gov/erddap/tabledap/documentation.html) for more information on the various parameters and responses of ERDDAP.

<div class="prompt input_prompt">
In&nbsp;[2]:
</div>

```python
try:
    from urllib.parse import unquote
except ImportError:
    from urllib2 import unquote


urlbase = "https://erddap.axiomdatascience.com/erddap/tabledap"

fname = "sensor_service.csvp"

columns = (
    "time",
    "value",
    "station",
    "longitude",
    "latitude",
    "parameter",
    "unit",
    "depth",
)
params = {
    # Inequalities do not exist in HTTP parameters,
    # so we need to hardcode the `>` in the time key to get a '>='.
    # Note that a '>' or '<' cannot be encoded with `urlencode`, only `>=` and `<=`.
    "time>": "2017-01-00T00:00:00Z",
    "station": '"urn:ioos:station:wmo:44011"',
    "parameter": '"Significant Wave Height"',
    "unit": '"m"',
}

url = encode_erddap(urlbase, fname, columns, params)

print(unquote(url))
```
<div class="output_area"><div class="prompt"></div>
<pre>
    https://erddap.axiomdatascience.com/erddap/tabledap/sensor_service.csvp?time,value,station,longitude,latitude,parameter,unit,depth&time>=2017-01-00T00:00:00Z&station="urn:ioos:station:wmo:44011"&parameter="Significant+Wave+Height"&unit="m"

</pre>
</div>
Here is a cool part about ERDDAP `tabledap` - The data `tabledap` `csvp` response can be easily read by Python's pandas `read_csv` function.

<div class="prompt input_prompt">
In&nbsp;[3]:
</div>

```python
from pandas import read_csv

df = read_csv(url, index_col=0, parse_dates=True)

# Prevent :station: from turning into an emoji in the webpage.
df["station"] = df.station.str.split(":").str.join("_")

df.head()
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
      <th>value</th>
      <th>station</th>
      <th>longitude (degrees_east)</th>
      <th>latitude (degrees_north)</th>
      <th>parameter</th>
      <th>unit</th>
      <th>depth (m)</th>
    </tr>
    <tr>
      <th>time (UTC)</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>2018-03-25 14:00:00+00:00</th>
      <td>5.6</td>
      <td>urn_ioos_station_wmo_44011</td>
      <td>-66.619</td>
      <td>41.098</td>
      <td>Significant Wave Height</td>
      <td>m</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>2018-03-25 13:50:00+00:00</th>
      <td>5.6</td>
      <td>urn_ioos_station_wmo_44011</td>
      <td>-66.619</td>
      <td>41.098</td>
      <td>Significant Wave Height</td>
      <td>m</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>2018-03-25 13:00:00+00:00</th>
      <td>4.6</td>
      <td>urn_ioos_station_wmo_44011</td>
      <td>-66.619</td>
      <td>41.098</td>
      <td>Significant Wave Height</td>
      <td>m</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>2018-03-25 12:50:00+00:00</th>
      <td>4.6</td>
      <td>urn_ioos_station_wmo_44011</td>
      <td>-66.619</td>
      <td>41.098</td>
      <td>Significant Wave Height</td>
      <td>m</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>2018-03-25 12:00:00+00:00</th>
      <td>5.1</td>
      <td>urn_ioos_station_wmo_44011</td>
      <td>-66.619</td>
      <td>41.098</td>
      <td>Significant Wave Height</td>
      <td>m</td>
      <td>0.0</td>
    </tr>
  </tbody>
</table>
</div>



With the `DataFrame` we can easily plot the data.

<div class="prompt input_prompt">
In&nbsp;[4]:
</div>

```python
%matplotlib inline

ax = df["value"].plot(figsize=(11, 2.75), title=df["parameter"][0])
```
<div class="warning" style="border:thin solid red">
    /home/filipe/miniconda3/envs/IOOS/lib/python3.7/site-
packages/pandas/core/sorting.py:257: FutureWarning: Converting timezone-aware
DatetimeArray to timezone-naive ndarray with 'datetime64[ns]' dtype. In the
future, this will return an ndarray with 'object' dtype where each element is a
'pandas.Timestamp' with the correct 'tz'.
        To accept the future behavior, pass 'dtype=object'.
        To keep the old behavior, pass 'dtype="datetime64[ns]"'.
      items = np.asanyarray(items)

</div>

![png](data:image/png;base64,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
)


You may notice that slicing the time dimension on the sever side is very fast when compared with an OPeNDAP request. The downloading of the time dimension data, slice, and subsequent downloading of the actual data are all much faster.

ERDDAP also allows for filtering of the variable's values. For example, let's get Wave Heights that are bigger than 6 meters starting from 2016.

\*\* Note how we can lazily build on top of the previous query using Python's dictionaries.

<div class="prompt input_prompt">
In&nbsp;[5]:
</div>

```python
params.update(
    {"value>": 6, "time>": "2016-01-00T00:00:00Z",}
)

url = encode_erddap(urlbase, fname, columns, params)

df = read_csv(url, index_col=0, parse_dates=True)

# Prevent :station: from turning into an emoji in the webpage.
df["station"] = df.station.str.split(":").str.join("_")

df.head()
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
      <th>value</th>
      <th>station</th>
      <th>longitude (degrees_east)</th>
      <th>latitude (degrees_north)</th>
      <th>parameter</th>
      <th>unit</th>
      <th>depth (m)</th>
    </tr>
    <tr>
      <th>time (UTC)</th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>2018-03-17 06:00:00+00:00</th>
      <td>6.0</td>
      <td>urn_ioos_station_wmo_44011</td>
      <td>-66.619</td>
      <td>41.098</td>
      <td>Significant Wave Height</td>
      <td>m</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>2018-03-17 05:50:00+00:00</th>
      <td>6.0</td>
      <td>urn_ioos_station_wmo_44011</td>
      <td>-66.619</td>
      <td>41.098</td>
      <td>Significant Wave Height</td>
      <td>m</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>2018-03-17 05:00:00+00:00</th>
      <td>6.0</td>
      <td>urn_ioos_station_wmo_44011</td>
      <td>-66.619</td>
      <td>41.098</td>
      <td>Significant Wave Height</td>
      <td>m</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>2018-03-17 04:50:00+00:00</th>
      <td>6.0</td>
      <td>urn_ioos_station_wmo_44011</td>
      <td>-66.619</td>
      <td>41.098</td>
      <td>Significant Wave Height</td>
      <td>m</td>
      <td>0.0</td>
    </tr>
    <tr>
      <th>2018-03-17 02:00:00+00:00</th>
      <td>6.1</td>
      <td>urn_ioos_station_wmo_44011</td>
      <td>-66.619</td>
      <td>41.098</td>
      <td>Significant Wave Height</td>
      <td>m</td>
      <td>0.0</td>
    </tr>
  </tbody>
</table>
</div>



And now we can visualize the frequency of `Significant Wave Height` greater than 6 meters by month.

<div class="prompt input_prompt">
In&nbsp;[6]:
</div>

```python
def key(x):
    return x.month


grouped = df["value"].groupby(key)

ax = grouped.count().plot.bar()
ax.set_ylabel("Significant Wave Height events > 6 meters")
m = ax.set_xticklabels(
    ["Jan", "Feb", "Mar", "Apr", "May", "Jun", "Jul", "Ago", "Sep", "Oct", "Nov", "Dec"]
)
```


![png](data:image/png;base64,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
)


Wow! Wintertime is pretty rough out on George's Bank!

There is also a built-in relative time functionality so you can specify a specific time frame you look at. Here we demonstrate this part of the tool by getting the last 2 hours and displaying that with the `HTML` response in an `IFrame`.

<div class="prompt input_prompt">
In&nbsp;[7]:
</div>

```python
from IPython.display import HTML

fname = "sensor_service.htmlTable"

params = {
    "time>": "now-2hours",
    "time<": "now",
    "station": '"urn:ioos:station:nerrs:wqbchmet"',
    "parameter": '"Wind Speed"',
    "unit": '"m.s-1"',
}

url = encode_erddap(urlbase, fname, columns, params)

iframe = '<iframe src="{src}" width="650" height="370"></iframe>'.format
HTML(iframe(src=url))
```
<div class="warning" style="border:thin solid red">
    /home/filipe/miniconda3/envs/IOOS/lib/python3.7/site-
packages/IPython/core/display.py:689: UserWarning: Consider using
IPython.display.IFrame instead
      warnings.warn("Consider using IPython.display.IFrame instead")

</div>



<iframe src="https://erddap.axiomdatascience.com/erddap/tabledap/sensor_service.htmlTable?time,value,station,longitude,latitude,parameter,unit,depth&time%3E=now-2hours&time%3C=now&station=%22urn%3Aioos%3Astation%3Anerrs%3Awqbchmet%22&parameter=%22Wind+Speed%22&unit=%22m.s-1%22" width="650" height="370"></iframe>



`ERDDAP` responses are very rich. There are even multiple image formats in the automate graph responses.
Here is how to get a `.png` file for the temperature time-series. While you can specify the width and height, we chose just an arbitrary size.

<div class="prompt input_prompt">
In&nbsp;[8]:
</div>

```python
fname = "sensor_service.png"

params = {
    "time>": "now-7days",
    "station": '"urn:ioos:station:nerrs:wqbchmet"',
    "parameter": '"Wind Speed"',
    "unit": '"m.s-1"',
}


width, height = 450, 500
params.update({".size": "{}|{}".format(width, height)})

url = encode_erddap(urlbase, fname, columns, params)

iframe = '<iframe src="{src}" width="{width}" height="{height}"></iframe>'.format
HTML(iframe(src=url, width=width + 5, height=height + 5))
```




<iframe src="https://erddap.axiomdatascience.com/erddap/tabledap/sensor_service.png?time,value,station,longitude,latitude,parameter,unit,depth&time%3E=now-7days&station=%22urn%3Aioos%3Astation%3Anerrs%3Awqbchmet%22&parameter=%22Wind+Speed%22&unit=%22m.s-1%22&.size=450%7C500" width="455" height="505"></iframe>



This example tells us it is rough and cold out on George's Bank!

To explore more datasets, use the IOOS sensor map [website](https://sensors.ioos.us/#map)!
<br>
Right click and choose Save link as... to
[download](https://raw.githubusercontent.com/ioos/notebooks_demos/master/notebooks/2017-03-21-ERDDAP_IOOS_Sensor_Map.ipynb)
this notebook, or click [here](https://binder.pangeo.io/v2/gh/ioos/notebooks_demos/master?filepath=notebooks/2017-03-21-ERDDAP_IOOS_Sensor_Map.ipynb) to run a live instance of this notebook.