## Plan a trip

Let's go on a journey together. But first, let's get some basic information so we can plan our trip. 

One item on our to-do list is to see the [Temple of Dendur](https://images.metmuseum.org/CRDImages/eg/original/DP240337.jpg) at the Met Museum in New York City. We can search for information about it using the museum's [Art Collection API](https://metmuseum.github.io/).

* Base URL: `https://collectionapi.metmuseum.org`
* Method: `GET /public/collection/v1/search`

| Parameter | Description |
| -- | -- |
| `q` | Search string |

```sh
$ curl -s https://collectionapi.metmuseum.org/public/collection/v1/search?q=Dendur | jq
```

The API returns an array of integer IDs of objects that contain text matching our query:

```
{
  "total": 27,
  "objectIDs": [
    547802,
    544227,
    ...
  ]
}
```

Or `null` if no results were found:

```
{
  "total": 0,
  "objectIDs": null
}
```

## See an object

Next, let's look up some basic facts about an object, using the `objects` route:

* Method: `GET /public/collection/v1/objects/{id}`
* Parameter: `id` - A numeric ObjectID.

The response schema is:
```
200 OK 

{
  "objectID": integer,
  ...
  "tags": [
    {
      "term": "string",
      "AAT_URL": "url",
      "Wikidata_URL": "url"
    }
  ],
  "title": "string", 
  "primaryImage": "url",
  "department": "string",
  "period": "string",
  "reign": "string",
  "objectDate": "string",
  "objectWikidata_URL": "url",
  "isTimelineWork": boolean,
  "GalleryNumber": "string"
}
```

Let's take a closer look at our specific object. We can `GET /objects/{id}`, where `id` is the first ObjectID returned above. Then we can select data fields with `jq`, and format the output as CSV for convenience:

```sh
curl -s https://collectionapi.metmuseum.org/public/collection/v1/objects/547802 | jq -r '[.title, .primaryImage, .department, .objectDate, .period, .reign] | @csv'
```
```
"The Temple of Dendur","https://images.metmuseum.org/CRDImages/eg/original/DP240337.jpg","Egyptian Art","completed by 10 B.C.","Roman Period","reign of Augustus"
```

We can also fetch multiple objects. Making one request per ID:

```sh
#!/usr/bin/env bash

IDS=547802,544227,544887

curl -s https://collectionapi.metmuseum.org/public/collection/v1/objects/{$IDS} \
  | jq -r '[.title, .primaryImage, .department, .objectDate, .period, .reign] | @csv' \
  | tee objects.csv
```
```
"The Temple of Dendur","https://images.metmuseum.org/CRDImages/eg/original/DP240337.jpg","Egyptian Art","completed by 10 B.C.","Roman Period","reign of Augustus"
"Hippopotamus (""William"")","https://images.metmuseum.org/CRDImages/eg/original/DP248993.jpg","Egyptian Art","ca. 1961-1878 B.C.","Middle Kingdom","Senwosret I to Senwosret II"
"Statue of Horus as a falcon protecting King Nectanebo II","https://images.metmuseum.org/CRDImages/eg/original/DP-38684-002.jpg","Egyptian Art","360-343 BCE","Late Period","reign of Nectanebo II"
```
Alternatively, in this example we use the Python `requests` library, and format the output as an array of JSON objects.

```python
#!/usr/bin/env python3

import requests, json

BASE_URL = "https://collectionapi.metmuseum.org"
IDS = (547802,544227,544887)
OBJECTS = []

for id in IDS:
  try:
    route = f'/public/collection/v1/objects/{id}'
    response = requests.get(BASE_URL + route)
    if response.status_code == 200:
      OBJECTS.append(response.json())
    else:
      print(f"HTTP error: {response.status_code} {response.reason}")
  except requests.exceptions.RequestException as e:
    print(f"Request exception: {str(e)}")

if len(OBJECTS):
  print(json.dumps(OBJECTS, indent=2))
```
```
[
  {
    "objectID": 547802,
    "primaryImage": "https://images.metmuseum.org/CRDImages/eg/original/DP240337.jpg",
    "title": "The Temple of Dendur",
    ...
  },
  {
    "objectID": 544227,
    "primaryImage": "https://images.metmuseum.org/CRDImages/eg/original/DP248993.jpg",
    "title": "Hippopotamus (\"William\")",
    ...
  },
  {
    "objectID": 544887,
    "primaryImage": "https://images.metmuseum.org/CRDImages/eg/original/DP-38684-002.jpg",
    "title": "Statue of Horus as a falcon protecting King Nectanebo II",
    ...
  }
]
```

The rate limit of the museum's API is 80 requests/sec. If we fetch more than 80 records at once, we'll need to limit our request rate.

## Map a location

Next, let's check the weather for the upcoming week. We know the Met is located at 1000 Fifth Avenue, New York City. Let's find its latitude and longitude using OpenStreetMap's free geocoding service, [Nominatim](https://nominatim.org/).

An API key is not required, but we must provide a `User-Agent` header in the request.

* Base URL: `https://nominatim.openstreetmap.org`
* Method: `GET /search`

| Parameter | Description |
| -- | -- |
| `q` | Free-form query string, e.g. street address |
| `format` | String, one of: `xml`, `json`, `jsonv2`, `geojson`, `geocodejson` |
| `limit` | Number of results to fetch, in range [`1`, `40`], default `10` |

Results are sorted by relevance. We only want the best match, so we specify `limit=1`. 

The query string is the url-encoding of "1000 Fifth Avenue, New York City, New York, U.S."

```sh
#!/usr/bin/env bash

curl -s -H "User-Agent: Neil/1.0" 'https://nominatim.openstreetmap.org/search?q=1000%20Fifth%20Avenue%2C%20New%20York%20City%2C%20New%20York%2C%20U.S.&format=json&limit=1' | jq
```   
```
[
  {
    "place_id": 338471781,
    "licence": "Data &copy; OpenStreetMap contributors, ODbL 1.0. http://osm.org/copyright",
    "osm_type": "way",
    "osm_id": 1271518873,
    "lat": "40.8021613",
    "lon": "-73.9453686",</b>
    "class": "highway",
    "type": "secondary",
    "place_rank": 26,
    "importance": 0.568635634246592,
    "addresstype": "road",
    "name": "5th Avenue",
    "display_name": "5th Avenue, Manhattan Community Board 11, Manhattan, New York County, City of New York, New York, 10035, United States of America",
    "boundingbox": [
      "40.8019710",
      "40.8023524",
      "-73.9455080",
      "-73.9452302"
    ]
  }
]
```

Or, to get the latitude and longitude using Python:

```python
#!/usr/bin/env python3

import requests, json

BASE_URL="https://nominatim.openstreetmap.org"
USER_AGENT="Neil/1.0"
ADDRESS="1000 Fifth Avenue, New York City, New York, U.S."

try:
  headers = {
    'User-Agent': USER_AGENT
  }
  params = {
    'q': ADDRESS,
    'format': 'json',
    'limit': 1
  }
  route = "/search"
  response = requests.get(BASE_URL + route, params=params, headers=headers)
  lat = response.json()[0]['lat']
  lon = response.json()[0]['lon']
  print(lat, lon)
except Exception as e:
  sys.exit(f"Exception: {str(e)}")
```
```
40.8021613 -73.9453686
```

## Check the weather

We can provide the latitude and longitude to the [Open Meteo API](https://open-meteo.com/) to find the weather forecast. The API is free for noncommercial use only. Request rate limit is 10,000 API calls per day, 5,000 per hour, and 600 per minute.

No API key is required.

* Base URL: `https://api.open-meteo.com/v1/`
* Method: `GET /forecast`

| Parameter | Description |
| -- | -- |
| `latitude` | Float in range `[-90, 90]` |
| `longitude` | Float in range `[-180, 180]` |
| `current` | List of measurement variables to fetch for the current weather |
| `daily` | List of forecast variables to fetch for the next `forecast_days` number of days |
| `forecast_days` | The forecast length in days (range [`0`, `16`], default `7`) |
| `timezone` | [TZ identifier code](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones) or `auto` for request origin TZ |

Variables:

| Variable | Valid time | Unit | Description |
| -- | -- | -- | -- |
| `temperature_2m` | Instant | C | Air temperature at 2 meters above ground |
| `is_day` | Instant | n/a | `1` if the current time step has daylight, or `0` at night |
| `precipitation` | Preceding 15 minutes sum | mm | Precipitation |
| `weather_code` | Instant | WMO code | Weather condition as a numeric WMO weather interpretation code |

```sh
curl -s "https://api.open-meteo.com/v1/forecast?latitude={40.8021613}&amp;longitude={-73.9453686}&amp;current=temperature_2m,is_day,precipitation,weather_code&amp;daily=weather_code,precipitation_probability_max&amp;timezone=America/New_York&amp;forecast_days=7" | jq
```
```json
{
  "latitude": 40.81466,
  "longitude": -73.9571,
  "generationtime_ms": 0.4341602325439453,
  "utc_offset_seconds": -18000,
  "timezone": "America/New_York",
  "timezone_abbreviation": "GMT-5",
  "elevation": 16.0,
  "current_units": {
    "time": "iso8601",
    "interval": "seconds",
    "temperature_2m": "°C",
    "is_day": "",
    "precipitation": "mm",
    "weather_code": "wmo code"
  },
  "current": {
    "time": "2025-11-06T21:30",
    "interval": 900,
    "temperature_2m": 5.4,
    "is_day": 0,
    "precipitation": 0.00,
    "weather_code": 3
  },
  "daily_units": {
    "time": "iso8601",
    "weather_code": "wmo code",
    "precipitation_probability_max": "%"
  },
  "daily": {
    "time": [
      "2025-11-06",
      "2025-11-07",
      "2025-11-08",
      "2025-11-09",
      "2025-11-10",
      "2025-11-11",
      "2025-11-12"
    ],
    "weather_code": [
      3,
      3,
      61,
      53,
      81,
      3,
      3
    ],
    "precipitation_probability_max": [
      7,
      42,
      65,
      55,
      48,
      11,
      5
    ]
  }
}
```

Or, in Python, with WMO code translation:

```python

import requests, json

BASE_URL = "https://api.open-meteo.com/v1"
ROUTE = "/forecast"
LAT = 40.8021613
LON = -73.9453686

def get_weather_forecast(lat, lon):
  params = {
    "latitude": lat,
    "longitude": lon,
    "current": "temperature_2m,is_day,precipitation,weather_code",
    "daily": "weather_code,precipitation_probability_max",
    "timezone": "auto"
  }

  try:
    response = requests.get(BASE_URL + ROUTE, params=params)
    return response.json()
  except Exception as e:
    sys.exit(f"Error fetching weather data: {str(e)}")

def wmo_decode(code):
  lut = {
    "0": "Clear sky",
    "1": "Mainly clear",
    "2": "Partly cloudy",
    "3": "Overcast",
    "45": "Fog",
    "48": "Heavy fog",
    "51": "Light drizzle",
    "53": "Moderate drizzle",
    "55": "Dense drizzle",
    "56": "Light freezing drizzle",
    "57": "Heavy freezing drizzle",
    "61": "Slight rain",
    "63": "Moderate rain",
    "65": "Heavy rain",
    "66": "Light freezing rain",
    "67": "Heavy freezing rain",
    "71": "Slight snow fall",
    "73": "Moderate snow fall",
    "75": "Heavy snow fall",
    "77": "Snow grains",
    "80": "Slight rain showers",
    "81": "Moderate rain showers",
    "82": "Violent rain showers",
    "85": "Slight snow showers",
    "86": "Heavy snow showers",
    "95": "Thunderstorm",
  }
  return lut[str(code)]

if __name__ == "__main__":
  weather_data = get_weather_forecast(LAT, LON)
  if weather_data:
    print(f"Current weather at [{LAT}, {LON}]:")
    print(f"Temperature: {weather_data['current']['temperature_2m']} C")
    print(f"Precipitation: {weather_data['current']['precipitation']} mm")
    print(f"Weather code: {wmo_decode(weather_data['current']['weather_code'])}")
    print("\nDaily Forecast:")
    for n in range(0, 7):
      print(weather_data['daily']['time'][n], wmo_decode(weather_data['daily']['weather_code'][n]))
```
```
Current weather at [40.8021613, -73.9453686]:
Temperature: 11.7 C
Precipitation: 0.0 mm
Weather code: Partly cloudy

Daily Forecast:
2025-11-06 Overcast
2025-11-07 Overcast
2025-11-08 Slight rain
2025-11-09 Moderate rain showers
2025-11-10 Moderate rain
2025-11-11 Overcast
2025-11-12 Overcast
```

That's all the info we need right now. We'll check the weather again as the date of our trip gets closer.

***
