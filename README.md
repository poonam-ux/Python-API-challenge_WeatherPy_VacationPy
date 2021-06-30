# Python-API-challenge_WeatherPy_VacationPy

This challenge includes two parts:

## Part I - WeatherPy

* Create a Python script to visualize the weather of 500+ cities across the world of varying distance from the equator. To accomplish this, you’ll be utilizing a simple Python library, Citipy, and the OpenWeatherMap API.

* The first requirement is to create a series of scatter plots to showcase the relationships between location and temperature, humidity, wind speed, and cloudiness. 

* The second requirement is to run linear regression on each relationship for the northern and southern hemispheres.

### Steps to follow:

* Randomly select at least 500 unique (non-repeat) cities based on latitude and longitude.
* Perform a weather check on each of the cities using a series of successive API calls.
* Include a print log of each city as it’s being processed with the city number and city name.
* Save a CSV of all retrieved data and a PNG image for each scatter plot.

## Part II - VacationPy

* Work with weather data created in Part I to plan future vacations. Use jupyter-gmaps and the Google Places API for this part of the assignment.
* Create a heat map that displays the humidity for every city from Part I.
* Narrow down the dataframe to find your ideal weather condition. For example:
    * A max temperature lower than 80 degrees but higher than 70.
    * Wind speed less than 10 mph.
    * Zero cloudiness.
    * Drop any rows that don’t contain all three conditions. You want to be sure the weather is ideal.
* Using Google Places API to find the first hotel for each city located within 5000 meters of your coordinates.
* Plot the hotels on top of the humidity heatmap with each pin containing the Hotel Name, City, and Country.

## Observable Trends for WeatherPy
* After collecting and analyzing weather data from 626 random cities around the world using the OpenWeatherMap API, it shows that the temperatures are higher closer to the Equator (0° latitude) for both, northern and southern hemispheres. Since the data is collected in January, the maximum temperatures are lower for northern hemisphere than those in southern hemisphere. This is because of the earth’s rotation around sun.
* There seems to be very little to no correlation between humidity and latitude, as well as cloudiness and latitude. The scatter plots show evenly spread values across the latitudes.
* Wind speed seems to be similar for latitudes close to the equator (latitudes between 0 and 40 degrees, in both hemispheres).

## Script - WeatherPy

```python
# Dependencies and Setup
import matplotlib.pyplot as plt
import pandas as pd
import numpy as np
import requests
import time
from datetime import datetime
from scipy.stats import linregress
import scipy.stats as st

# Import API key
from api_keys import weather_api_key

# Incorporated citipy to determine city based on latitude and longitude
from citipy import citipy

# Output File (CSV)
output_data_file = "output_data/cities.csv"

# Range of latitudes and longitudes
lat_range = (-90, 90)
lng_range = (-180, 180)
```
#### Generate cities list

```python
# List for holding lat_lngs and cities
lat_lngs = []
cities = []

# Create a set of random lat and lng combinations
lats = np.random.uniform(lat_range[0], lat_range[1], size=1500)
lngs = np.random.uniform(lng_range[0], lng_range[1], size=1500)
lat_lngs = zip(lats, lngs)

# Identify nearest city for each lat, lng combination
for lat_lng in lat_lngs:
    city = citipy.nearest_city(lat_lng[0], lat_lng[1]).city_name
    
    # If the city is unique, then add it to a our cities list
    if city not in cities:
        cities.append(city)

# Print the city count to confirm sufficient count
len(cities)
```
* There are 626 cities in the study.

#### Perform API Calls

```python
# URL for GET requests to retrieve city data
url = "http://api.openweathermap.org/data/2.5/weather?"
units = "imperial"

# Build partial query URL
query_url = f"{url}appid={weather_api_key}&units={units}&q="

# List for holding reponse information
lon = []
temp = []
temp_max = []
humidity = []
wind_speed = []
lat = []
date = []
country = []
cloudiness = []

# Loop through the list of cities and request for data on each
print("Beginning Data Retrieval")
print("-------------------------------------")
count = 0
set = 1
for index, city in enumerate(cities):
    count = count + 1
    # To avoid api call rate limits, get city weather data in sets of 50 cities,
    # with 5 seconds sleep time, and then continue
    if count == 51:
        count = 1
        set = set + 1
        time.sleep(5)
    print(f"Processing Record {count} of Set {set} | {city}")
    try:
        response = requests.get(query_url + city).json()
        lat.append(response['coord']['lat'])
        lon.append(response['coord']['lon'])
        temp.append(response['main']['temp'])
        temp_max.append(response['main']['temp_max'])
        humidity.append(response['main']['humidity'])
        wind_speed.append(response['wind']['speed'])
        date.append(response['dt'])
        country.append(response['sys']['country'])
        cloudiness.append(response['clouds']['all'])
    except KeyError:
        print("City not found. Skipping...")
        lat.append(np.nan)
        lon.append(np.nan)
        temp.append(np.nan)
        temp_max.append(np.nan)
        humidity.append(np.nan)
        wind_speed.append(np.nan)
        date.append(np.nan)
        country.append(np.nan)
        cloudiness.append(np.nan)
print("-------------------------------------")
print("Data Retrieval Complete")
print("-------------------------------------")
```
![](https://github.com/poonam-ux/Python-API-challenge_WeatherPy_VacationPy/blob/main/WeatherPy/output_data/data_retrieval_screenshot.pdf)

#### Convert raw data to dataframe

```python
cities_df = pd.DataFrame({
    "City": cities,
    "Lat": lat,
    "Lng": lon,
    "Max Temp": temp_max,
    "Humidity": humidity,
    "Cloudiness": cloudiness,
    "Wind Speed": wind_speed,
    "Country": country,
    "Date": date,
})

# Drop any cities that were skipped because they could not return any response from OpenWeatherMap API.
cities_df = cities_df.dropna(how="any")

# Export the city data into a .csv file.
cities_df.to_csv("./output_data/cities.csv", index=False)

# Display the DataFrame
cities_df
```
![](https://github.com/poonam-ux/Python-API-challenge_WeatherPy_VacationPy/blob/main/WeatherPy/output_data/cities_df.png)

#### Inspect the data and remove the cities where the humidity > 100%

```python
#check if there are any cities with Humidity >100% 
cities_df["Humidity"].describe()
```
```python
#  Get the indices of cities that have humidity over 100%.
humidity_101 = cities_df[(cities_df["Humidity"] > 100)].index
humidity_101
```
```python
# Make a new DataFrame equal to the city data to drop all humidity outliers by index.
# Passing "inplace=False" will make a copy of the city_data DataFrame, which we call "clean_city_data".
clean_city_data = cities_df.drop(humidity_101, inplace=False)
clean_city_data
```
![](https://github.com/poonam-ux/Python-API-challenge_WeatherPy_VacationPy/blob/main/WeatherPy/output_data/clean_city_data.png)

## Plotting the data

#### Latitude vs Max Temp (F)

```python
date_now = datetime.date(datetime.now())

# Create a scatter plot for latitude vs max temperature.
x_values = clean_city_data["Lat"]
y_values = clean_city_data["Max Temp"]

fig1, ax1 = plt.subplots(figsize=(11,8))
plt.scatter(x_values, y_values, edgecolor="black", linewidth=1, marker="o", alpha=0.8)
plt.title(f"City Latitude vs Max Temperature {date_now}")
plt.xlabel("Latitude")
plt.ylabel("Max Temperature (F)")
plt.grid()

# Save the figure
plt.savefig("./output_data/latitude_vs_max_temp.png", bbox_inches="tight")
plt.show()
```
![](https://github.com/poonam-ux/Python-API-challenge_WeatherPy_VacationPy/blob/main/WeatherPy/output_data/latitude_vs_max_temp.png)

#### Latitude vs Humidity

```python
x_values = clean_city_data["Lat"]
y_values = clean_city_data["Humidity"]

fig1, ax1 = plt.subplots(figsize=(11, 8))
plt.scatter(x_values, y_values, edgecolor="black", linewidth=1, marker="o", alpha=0.8)
plt.xlabel("Latitude")
plt.ylabel("Humidity (%)")
plt.title(f"City Latitude vs Humidity {date_now}")
plt.grid()

# Save the figure
plt.savefig("./output_data/latitude_vs_humidity.png", bbox_inches="tight")
plt.show()
```
![](https://github.com/poonam-ux/Python-API-challenge_WeatherPy_VacationPy/blob/main/WeatherPy/output_data/latitude_vs_humidity.png)

#### Latitude vs Cloudiness

```python
# Create a scatter plot for latitude vs cloudiness.
x_values = clean_city_data["Lat"]
y_values = clean_city_data["Cloudiness"]

fig1, ax1 = plt.subplots(figsize=(10,8))
markersize=12
plt.scatter(x_values, y_values, edgecolor="black", linewidth=1, marker="o", alpha=0.8)
plt.xlabel("Latitude")
plt.ylabel("Cloudiness (%)")
plt.title(f"City Latitude vs Cloudiness {date_now}")
plt.grid()

# Save the figure
plt.savefig("./output_data/latitude_vs_cloudiness.png", bbox_inches="tight")
plt.show()
```
![](https://github.com/poonam-ux/Python-API-challenge_WeatherPy_VacationPy/blob/main/WeatherPy/output_data/latitude_vs_cloudiness.png)

#### Latitude vs Wind Speed

```python
# Create a scatter plot for latitude vs wind speed.
x_values = clean_city_data["Lat"]
y_values = clean_city_data["Wind Speed"]

fig1, ax1 = plt.subplots(figsize=(10,8))
markersize=12
plt.scatter(x_values, y_values, edgecolor="black", linewidth=1, marker="o", alpha=0.8)

plt.xlabel("Latitude")
plt.ylabel("Wind Speed (mph)")
plt.title(f"City Latitude vs Wind Speed {date_now}")
plt.grid()

# Save the figure
plt.savefig("./output_data/latitude_vs_wind_speed.png", bbox_inches="tight")
plt.show()
```
![](https://github.com/poonam-ux/Python-API-challenge_WeatherPy_VacationPy/blob/main/WeatherPy/output_data/latitude_vs_wind_speed.png)

#### Linear Regression

```python
# Create a function to create Linear Regression plots for remaining activities
def plot_linear_regression(x_values, y_values, x_label, y_label, hemisphere, text_coordinates, ylim=None):
    (slope, intercept, rvalue, pvalue, stderr) = linregress(x_values, y_values)

    # Get regression values
    regress_values = x_values * slope + intercept
    
    # Create line equation string
    line_eq = "y = " + str(round(slope,2)) + "x +" + str(round(intercept,2))
    
    # Generate plots     
    fig1, ax1 = plt.subplots(figsize=(12,8))
    plt.scatter(x_values, y_values, edgecolor="black", linewidth=1, marker="o", alpha=0.8)
    plt.plot(x_values,regress_values,"r-")
    date_now = datetime.date(datetime.now())
    plt.title(f"{hemisphere} Hemisphere - {x_label} vs {y_label} {date_now}",fontsize = 15)
    plt.xlabel(x_label,fontsize=14)
    plt.ylabel(y_label,fontsize=14)
    if ylim is not None:
        plt.ylim(0, ylim)
    plt.annotate(line_eq, text_coordinates, fontsize=20, color="red")
    
    # Print r square value
    print(f"The r-squared is: {rvalue**2}")
    correlation = st.pearsonr(x_values,y_values)
    print(f"The correlation between both factors is {round(correlation[0],2)}")
```
```python

# Create Northern and Southern Hemisphere DataFrames
northern_hemi_weather_df = clean_city_data.loc[clean_city_data["Lat"] >= 0]
southern_hemi_weather_df = clean_city_data.loc[clean_city_data["Lat"] < 0]
```
#### Northern hemisphere - Lat vs Max Temp (F)

```python
# Create a scatter plot for latitude vs max temp (northern hemisphere)
x_values = northern_hemi_weather_df["Lat"]
y_values = northern_hemi_weather_df["Max Temp"]
plot_linear_regression(x_values, y_values, "Latitude", "Max Temp (F)", "Northern", (10, 10))

# Save the figure
plt.savefig("./output_data/northern_hem_linear_lat_vs_max_temp.png", bbox_inches="tight")
plt.show()
```
* The r-squared is: 0.7485887373160589
* The correlation between both factors is -0.87

![](https://github.com/poonam-ux/Python-API-challenge_WeatherPy_VacationPy/blob/main/WeatherPy/output_data/northern_hem_linear_lat_vs_max_temp.png)

#### Southerm hemisphere - Lat vs Max Temp

```python
# Create a scatter plot for latitude vs cloudiness (southern hemisphere)
x_values = southern_hemi_weather_df["Lat"]
y_values = southern_hemi_weather_df["Max Temp"]
plot_linear_regression(x_values, y_values, "Latitude", "Max Temp (F)", "Southern", (-52, 75))

# Save the figure
plt.savefig("./output_data/southern_hem_linear_lat_vs_max_temp.png", bbox_inches="tight")
plt.show()
```
![](https://github.com/poonam-ux/Python-API-challenge_WeatherPy_VacationPy/blob/main/WeatherPy/output_data/southern_hem_linear_lat_vs_max_temp.png)

* The correlation between latitude and maximum temperature is strong for both the hemispheres. It is higher for northern hemisphere (0.87), indicating that as we move away from the equator, the maximum temperature keeps dropping in a more linear manner.

#### Northern hemisphere - Lat vs Humidity

```python
# Create a scatter plot for latitude vs humditiy (northern hemisphere)
x_values = northern_hemi_weather_df['Lat']
y_values = northern_hemi_weather_df['Humidity']
plot_linear_regression(x_values, y_values, "Latitude", "Humidity (%)", "Northern",(50,50))
plt.savefig("./output_data/northern_hem_linear_lat_vs_humidity.png", bbox_inches="tight")
plt.show()
```
![](https://github.com/poonam-ux/Python-API-challenge_WeatherPy_VacationPy/blob/main/WeatherPy/output_data/northern_hem_linear_lat_vs_humidity.png)

#### Southern hemisphere - Lat vs Humidity

```python
# Create a scatter plot for latitude vs humditiy (southern hemisphere)
x_values = southern_hemi_weather_df['Lat']
y_values = southern_hemi_weather_df['Humidity']
plot_linear_regression(x_values, y_values, "Latitude", "Humidity (%)", "Southern",(50, 50), 100)
plt.savefig("./output_data/southern_hem_linear_lat_vs_humudity.png", bbox_inches="tight")
plt.show()
```
![](https://github.com/poonam-ux/Python-API-challenge_WeatherPy_VacationPy/blob/main/WeatherPy/output_data/southern_hem_linear_lat_vs_humudity.png)

* There is no correlation between latitude and humidity for southern hemisphere (0.3). For northern hemisphere, it is the same case, expect for the higher latitudes, where we can see some correlation (but not very strong).

#### Northern hemisphere - Lat vs Cloudiness

```python
# Create a scatter plot for latitude vs cloudiness (northern hemisphere)
x_values = northern_hemi_weather_df['Lat']
y_values = northern_hemi_weather_df['Cloudiness']
plot_linear_regression(x_values, y_values, "Latitude", "Cloudiness (%)", "Northern", (20, 60))

plt.savefig("./output_data/northern_hem_linear_lat_vs_cloudiness.png", bbox_inches="tight")
plt.show()
```
![](https://github.com/poonam-ux/Python-API-challenge_WeatherPy_VacationPy/blob/main/WeatherPy/output_data/northern_hem_linear_lat_vs_cloudiness.png)

#### Southern hemisphere - Lat vs Cloudiness

```python
# Create a scatter plot for latitude vs cloudiness (southern hemisphere)
x_values = southern_hemi_weather_df['Lat']
y_values = southern_hemi_weather_df['Cloudiness']
plot_linear_regression(x_values, y_values, "Latitude", "Cloudiness(%)", "Southern",(-45, 60))
plt.savefig("./output_data/southern_hem_linear_lat_vs_cloudiness.png", bbox_inches="tight")
plt.show()
```
![](https://github.com/poonam-ux/Python-API-challenge_WeatherPy_VacationPy/blob/main/WeatherPy/output_data/southern_hem_linear_lat_vs_cloudiness.png)

* There is no correlation between latitude and cloudiness for both, southern and northern hemispheres. Both show scattered values all over the plots.

#### Northern hemisphere - Lat vs Wind Speed

```python
# Create a scatter plot for latitude vs wind speed(northern hemisphere)
x_values = northern_hemi_weather_df['Lat']
y_values = northern_hemi_weather_df['Wind Speed']
plot_linear_regression(x_values, y_values, "Latitude", "Wind Speed (mph)", "Northern",(20, 25))
plt.savefig("./output_data/northern_hem_linear_lat_vs_wind_speed.png", bbox_inches="tight")
plt.show()
```
![](https://github.com/poonam-ux/Python-API-challenge_WeatherPy_VacationPy/blob/main/WeatherPy/output_data/northern_hem_linear_lat_vs_wind_speed.png)

#### Southern hemisphere - Lat vs Wind Speed

```python
# Create a scatter plot for latitude vs wind speed (southern hemisphere)
x_values = southern_hemi_weather_df['Lat']
y_values = southern_hemi_weather_df['Wind Speed']
plot_linear_regression(x_values, y_values, "Latitude", "Wind Speed (mph)", "Southern",(-40, 25), ylim=40)
plt.savefig("./output_data/southern_hem_linear_lat_vs_wind_speed.png", bbox_inches="tight")
plt.show()
```
![](https://github.com/poonam-ux/Python-API-challenge_WeatherPy_VacationPy/blob/main/WeatherPy/output_data/southern_hem_linear_lat_vs_wind_speed.png)

* The r-squared is: 0.006068987073174655
* The correlation between both factors is -0.08

* There is no correlation between latitude and wind speed either, for both hemispheres. Both show evenly scattered values over the latitudes.

## VacationPy Images

* Heat Layer on Google maps for cities in Part I

![](https://github.com/poonam-ux/Python-API-challenge_WeatherPy_VacationPy/blob/main/VacationPy/output_data/heat_layer_weather_map.png)

* Hotels marked on Google Maps

![](https://github.com/poonam-ux/Python-API-challenge_WeatherPy_VacationPy/blob/main/VacationPy/output_data/hotels_vacation_map.png)
