# Background

This data-cube contains the centroid, amplitude, sigma and skew values of the spectral lines retrieved using NEID Solar Data from February 8, 2021, approximately 16:00 - 23:00.

## Structure
The architecture of the cube, a Numpy nested dictionary, is as follows:
```python
# DATA CUBE FORMAT
# For each spectral order from Order 40 - 110,
# 
# File Name (Date-Time)
#    Order
#       Line  
#          centroid (in angstroms)
#          centroid_pixval (in pixels)
#          norm_amplitude (normalized depth)
#          elec_amplitude (depth in electron flux)
#          sigma 
#          skew
```

## Imports
Since the data-cube is a Numpy nested dictionary, we need to import Numpy to work with the values. Matplotlib is also imported for visualizations:
```python
import numpy as np
import matplotlib.dates as mdates
import matplotlib.pyplot as plt
%matplotlib inline
```

## Using the Data
```python
# Reading the data cube (Replace 'x' with latest version)
data = np.load('data_cube_vx.npy',allow_pickle='TRUE').item()

# Returns the entire cube
print(data)

# Working with Values
# The example below would give you the centroid value for the 3rd Line in Order 50 for Feb 8, 2021 @ 19:10:24
centroid = data['Time 20210208T191024')]['Order 50']['Line 3']['centroid']
```

## Working with Date-times and Plots
The datetime values from the cube need to be converted to a format that is 'plotting-friendly' in order to be visualized correctly:
```python
# Organize datetimes
datetimes = []
for x in data:
    x = x.replace('Time ', '')
    datetimes.append(x)
datetimes.sort()


# Make datetimes 'readable' for visualization
import datetime
formatted = []
np_datetimes = np.array(datetimes)
for value in np_datetimes:
    d = datetime.datetime.strptime(value, '%Y%m%dT%H%M%S')
    formatted.append(d)
dates = mdates.date2num(formatted) # <- This would be your x-axis when plotting 
```

A plot may therefore be formatted as:
```python
plt.plot_date(dates, y, color='green', label='Values over Time')
```

# Methods used for Aquiring the Data

Here I will explain the methods for which I have arrived at the values in this data-cube. More information regarding this process can be found in the Main.ipynb notebook on GitHub.

## Skewed Gaussian Model
A wave chunk within a 15-pixel range is created using a list of reference line values. The wave chunk along with the normalized upside values are passed into a Skewed Gaussian Model. This model was used from the following library:
```python
# Import Models for Fitting
from lmfit.models import GaussianModel, SkewedGaussianModel, VoigtModel
```
```python
wchunk = wave[order,(reference_centroid-15):(reference_centroid+15)]
xchunk = list(range(0,30))

mod = SkewedGaussianModel()
pars = mod.guess(norm_upside, x=wchunk)
out = mod.fit(norm_upside, pars, x=wchunk)
```

## Centroid Values
The centroid values, both in Angstrom and pixel units, can therefore be accessed from the model as so:
```python
centroid = pars['center'].value

# Interpolating to arrive at the corresponding pixel value
centroid_pixval = np.interp(centroid, wchunk, reference_centroid+xchunk)
```

## Amplitude Values
The normalized amplitude, out of a scale of 1.0, is found using the maximum, or peak, value of the Skewed Gaussian fit Model. To find the depth in terms of electron flux, I had to create a second model using the non-normalized flux upside:
```python
# Normalized Amplitude
norm_amplitude = np.amax(out.best_fit, axis=0)

# Electron Amplitude
mod = SkewedGaussianModel()
pars2 = mod.guess(upside, x=wchunk)
out = mod.fit(upside, pars2, x=wchunk)
elec_amplitude = pars2['amplitude'].value
```

## Sigma
The sigma value is simply determined from the output of the normalized skewed model:
```python
sigma = pars['sigma'].value
```

## Skew
The Skewed Gaussian model does not include skew as a default parameter. Thus, I converted the data into a pandas DataFrame and extrapolated the skew value from the it.

To use pandas, import the following library:
```python
import pandas as pd
```
```python
get_skew = pd.DataFrame(out.best_fit).skew()
skew = get_skew[0]
```

## License
[NEID Archive Data](https://neid.ipac.caltech.edu/search.php)