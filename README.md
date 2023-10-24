# python-api-challenge
Week 6 Assignment - (WeatherPy and VacationPy)

# Dependencies and Setup
import matplotlib.pyplot as plt
import pandas as pd
import numpy as np
import requests
import time
from scipy.stats import stats
from scipy.stats import linregress

from pprint import pprint
import json

# Import the OpenWeatherMap API key
from api_keys import weather_api_key

# Import citipy to determine the cities based on latitude and longitude
from citipy import citipy

# Empty list for holding the latitude and longitude combinations
lat_lngs = []

# Empty list for holding the cities names
cities = []

# Range of latitudes and longitudes
lat_range = (-90, 90)
lng_range = (-180, 180)

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
print(f"Number of cities in the list: {len(cities)}")

# Set the API base URL
url = "http://api.openweathermap.org/data/2.5/weather"
weather_api_key = "87b82ac74a0d881bca35f1c30d74d4fd"

# Convert Kelvin standard to Celsius
unit = 'metric'

# Define an empty list to fetch the weather data for each city
city_data = []

# Print to logger
print("Beginning Data Retrieval     ")
print("-----------------------------")

# Create counters
record_count = 1
set_count = 1

# Loop through all the cities in our list to fetch weather data
for i, city in enumerate(cities):
        
# Group cities in sets of 50 for logging purposes
if (i % 50 == 0 and i >= 50):
set_count += 1
record_count = 0

# Create endpoint URL with each city
city_url = f'{url}?q={city}&appid={weather_api_key}&units={unit}'
    
    
# Log the url, record, and set numbers
    print("Processing Record %s of Set %s | %s" % (record_count, set_count, city))

# Add 1 to the record count
    record_count += 1

# Run an API request for each of the cities

    city_weather = requests.get(city_url).json()
    print(json.dumps(city_weather,indent=4,sort_keys=True))

    try:    

        
# Parse out latitude, longitude, max temp, humidity, cloudiness, wind speed, country, and date
        city_lat = city_weather['coord']['lat']
        city_lng = city_weather['coord']['lon']
        city_max_temp =city_weather['main']['temp_max']
        city_humidity = city_weather['main']['humidity']
        city_clouds = city_weather['clouds']['all']
        city_wind = city_weather['wind']['speed']
        city_country = city_weather['sys']['country']
        city_date = city_weather['dt']

# Append the City information into city_data list
        city_data.append({"City": city, 
                          "Lat": city_lat, 
                          "Lng": city_lng, 
                          "Max Temp": city_max_temp,
                          "Humidity": city_humidity,
                          "Cloudiness": city_clouds,
                          "Wind Speed": city_wind,
                          "Country": city_country,
                          "Date": city_date})
        
# If an error is experienced, skip the city
    except:
        print("City not found. Skipping...")
        pass

# Convert the cities weather data into a Pandas DataFrame
city_data_df = pd.DataFrame(city_data)

# Show Record Count
city_data_df.count()

# Display sample data
city_data_df.head()

# Export Data into CSV
city_data_df.to_csv("../output_data/cities.csv", index_label="City_ID")

# Read saved data
city_data_df = pd.read_csv("../output_data/cities.csv", index_col="City_ID")

# Display sample data
city_data_df.head()

# Build scatter plot for latitude vs. temperature
city_data_df.plot('Lat','Max Temp',kind='scatter',edgecolors ='black')

# Incorporate the other graph properties
plt.xlabel('Latitude')
plt.ylabel('Max Temperature (C)')
plt.title('City Max Latitude vs. Temperature')
plt.grid()

# Save the figure
plt.savefig("../output_data/Fig1.png")

# Show plot
plt.show()

# Perform likewise for following 3 graphs

## Linear Regression
# Define a function to create Linear Regression plots
x_values = city_data_df['Lat']
y_values = city_data_df['Max Temp']

(slope, intercept, rvalue, pvalue, stderr) = stats.linregress(x_values, y_values)

# Get regression values
regress_values_lat_temp = x_values * slope + intercept
print(regress_values_lat_temp)

# Create a DataFrame with the Northern Hemisphere data (Latitude >= 0)
northern_hemi_df = pd.DataFrame(city_data_df.loc[city_data_df['Lat']>=0,:])

# Display sample data
northern_hemi_df.head()

# Create a DataFrame with the Southern Hemisphere data (Latitude < 0)
southern_hemi_df = pd.DataFrame(city_data_df.loc[city_data_df['Lat']<=0,:])

# Display sample data
southern_hemi_df.head()

# Linear regression on Northern Hemisphere

x_values_north = northern_hemi_df['Lat']
y_values_north = northern_hemi_df['Max Temp']

(slope, intercept, rvalue, pvalue, stderr) = stats.linregress(x_values_north, y_values_north)

# Get regression values
regress_values_northern = x_values_north * slope + intercept
print(regress_values_northern)

# Perform likewise for next linear regression values for Lat vs respective columns

#### VACATIONPY.IPYNB

# Dependencies and Setup
import hvplot.pandas
import pandas as pd
import requests

# Import API key
from api_keys import geoapify_key

# Load the CSV file created in Part 1 into a Pandas DataFrame
city_data_df = pd.read_csv("../output_data/cities.csv")

# Display sample data
city_data_df.head()

# Configure the map plot
map_plot = city_data_df.hvplot.points(
    "Lng",
    "Lat",
    geo = True,
    tiles = "OSM",
    frame_width = 1000,
    frame_height = 700,
    size = "Humidity",
    scale = 1,
    color = "City"

)

# Display the map plot_1
map_plot

# Narrow down cities that fit criteria and drop any results with null values
city_data_df = city_data_df.loc[city_data_df['Humidity']<=50,:]

# Drop any rows with null values
city_data_df.dropna()

# Display sample data
city_data_df

# Use the Pandas copy function to create DataFrame called hotel_df to store the city, country, coordinates, and humidity
hotel_df = city_data_df[['City','Country','Lat','Lng','Humidity']].copy()

# Add an empty column, "Hotel Name," to the DataFrame so you can store the hotel found using the Geoapify API
hotel_df['Hotel Name'] = ""

# Display sample data
hotel_df.head()

# Set parameters to search for a hotel
radius = 10000

params = {
    'categories':'accommodation.hotel',
    'filter': "",
    'bias': "",
    'apiKey': geoapify_key,
}


# Print a message to follow up the hotel search
print("Starting hotel search")

# Iterate through the hotel_df DataFrame
for index, row in hotel_df.iterrows():
# get latitude, longitude from the DataFrame
    latitude = hotel_df.loc[index,'Lat']
    longitude = hotel_df.loc[index,'Lng']

# Add filter and bias parameters with the current city's latitude and longitude to the params dictionary
    params["filter"] = f"circle:{longitude},{latitude},{radius}"
    params["bias"] = f"proximity:{longitude},{latitude}"

    
# Set base URL
    base_url = "https://api.geoapify.com/v2/places?"
# Make and API request using the params dictionaty
    name_address =  requests.get(base_url, params=params) 
    
# Convert the API response to JSON format
    name_address = name_address.json()
    
# Grab the first hotel from the results and store the name in the hotel_df DataFrame
    try:
       hotel_df.loc[index, "Hotel Name"] = name_address["features"][0]["properties"]["name"]
    except (KeyError, IndexError):
         # If no hotel is found, set the hotel name as "No hotel found".
        hotel_df.loc[index, "Hotel Name"] = "No hotel found"
        
# Log the search results
    print(f"{hotel_df.loc[index, 'City']} - nearest hotel: {hotel_df.loc[index, 'Hotel Name']}")

# Display sample data
hotel_df

# Configure the map plot
map_plotz = hotel_df.hvplot.points(
    "Lng",
    "Lat",
    geo = True,
    tiles = "OSM",
    frame_width = 1000,
    frame_height = 700,
    size = "Hotel",
    scale = 1,
    color = "City"

)

# Display the map plot_1
map_plotz


