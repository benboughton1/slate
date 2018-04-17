---
title: SAPI Reference

language_tabs: # must be one of https://git.io/vQNgJ
  - shell
  - python
  - javascript

toc_footers:
  - <a href='https://sapi.satamap.com.au/signup' target=_top>Sign Up for a Developer Key</a>

includes:
  - errors

search: true
---

# Introduction

Welcome to the Satamap API (SAPI)! You can use SAPI to access API endpoints for information on various areas of interest, such as paddocks, fields, etc.

We have language examples using Curl, Javascript, and Python. You can view these code examples in the dark area to the right, and you can switch the programming language of the examples with the tabs in the top right.

# Demo App

There is a Demo App available at [https://sapi.satamap.com.au/demo](https://sapi.satamap.com.au/demo). It allows you to create new polygons, get previously created polygons and get coverage information and downloads
of those polygons. You will need an API key to use the Demo App.

# Sentinel 2 vs Landsat 8

At this time, coverage is available from Sentinel 2 only. Requests for Landsat 8 coverage will return a 200 response
but with a message indicating that L8 is not yet available.

# HTTPS

SAPI provides all API endpoints across HTTPS (TLS). Request to API endpoints via HTTP will return a 307 response
code with the URI requested mapped to https instead.

# Tutorial

The following is an overview of the likely steps that your application will make using SAPI (the Satamap API) in order to get information about areas of interest. SAPI allows you to define areas (known as polygons) and then query for both currently available satellite coverage of that area and also whether updates to that area have occurred since a certain date or since a certain number of hours ago.

### API Key

To use SAPI, you will need to register and get an API key. Your API keys ensures access to SAPI and also associates any polygons you define with your account.

Once you have your API key, the first step is usually to add a polygon to SAPI. It is possible to do a one-time query about a polygon and get present coverage, but usually you will want to define a polygon with SAPI and then query about that polygon over time.

### 1. Add a polygon to SAPI using `/poly` POST

Suppose we have a simple rectangular area which consists of a rectangle i.e. four coordinates defining each point of the rectangle. Polygons can consist of multiple coordinates but for the purposes of this exercise we will use a simple rectangle. Lets the define the rectangle as four pairs of longitude-latitude coordinates, namely:

`150.21966 -29.37155,
150.21966 -29.36138,
150.23494 -29.36138 and
150.23494 -29.37155`

We need to restructure these coordinates and pass them to `https://sapi.satamap.com.au/v1/<apikey>/`. All SAPI URIs use this base URI.

The specific URI request to create a polygon is a POST request to `https://sapi.satamap.com.au/v1/<apikey>/poly/` followed by the coordinates. To create a polygon, we need to provide a bounded area and so we pass the first coordinate both as the first and last in the coordinate series.

The sequence of coordinates are passed in a format known as WKT (Well-Known Text) which for the above coordinates would be:
`((150.21966 -29.37155,150.21966 -29.36138,150.23494 -29.36138,150.23494 -29.37155,150.21966 -29.37155))`

If your API key was `mykey`, then using the command-line utility `curl`, a POST can be performed with:
 
`curl -X POST "https://sapi.satamap.com.au/v1/mykey/poly/((150.21966%20-29.37155,150.21966%20-29.36138,150.23494%20-29.36138,150.23494%20-29.37155,150.21966%20-29.37155))"`

Note that as the coordinates form a URI, we encode spaces as `%20`.

From the above POST, a UUID will be returned by SAPI in a JSON response:

`{
  "message": "polygon created",
  "uuid": "123e4567-e89b-12d3-a456-426655440000"
}`

Store this UUID safely (e.g. in your database) as this is how you will refer to this polygon in the future.
You now have a couple options to download imagery, either the last captured or from the SAPI archive.

### 2. Get the latest imagery

Once the polygon is defined and you have the UUID, you can download the latest image capture by using the `/poly/download_latest` URI. The full URI format is:

`https://sapi.satamap.com.au/v1/<apikey>/poly/download_latest/<uuid>/<coverage>/<filetype>`

where UUID is the reference to the polygon.

SAPI detects cloud in images and the `coverage` parameter is a value from 0 to 1 that allows you to specify how much non-cloud coverage is acceptable to you. For example, let's specify 1 which will mean the latest image that is clear of any cloud.

SAPI supports RGB, NDVI and PCD. Let's use RGB and using the UUID example provided previously, that will give us
a valid URI of:

`https://sapi.satamap.com.au/v1/mykey/poly/download_latest/123e4567-e89b-12d3-a456-426655440000/1/RGB`

You can try this with `curl` or even just put the URI into your browser. A recent browser will save the file with a filename that includes the filetype and the imagery date e.g. `RGB_2016-12-01.tif`

### 3. Find all available imagery

```
{
  "availabledates": [
    {
      "tile": "t091080",
      "source": "l8",
      "pass": 0,
      "date": "2016-11-27",
      "coverage": 1
    },
    {
      "tile": "t56jkn",
      "source": "s2",
      "pass": 0,
      "date": "2016-11-26",
      "coverage": 1
    }
  ]
}
```

You can also get information about all available imagery for your polygon using the `/poly/availabledates/<uuid>` URI.

This will return a JSON response that lists every date for which imagery is available and details about how that imagery was captured (e.e satellite source, satellite pass, etc.). An example response to *availabledates* is listed in the dark area to the right.

The JSON response provides the date of each image, as well as the satellite source ('s2' is Sentinel 2, 'l8' is Landsat 8), the satellite 'tile', the satellite pass (usually 0) and the coverage. Coverage can range from 0% to 100% and so the coverage value is normalized to between 0 and 1.

### 4. Find recent imagery

```
{
  "updated": "2017-01-02 00:11 UTC",
  "uuids": [
    "583fb96c-5f91-4ffa-8fca-bd799460e5ff"
  ]
}
```

If you want to check if your defined polygons have been updated recently, you can use `/poly/updates` which will return a list of UUIDs of your polygons that have been updated since duration you specify. 

So to get all polygons updated in the last 24 hours with 100% coverage, the URI would be:

`https://sapi.satamap.com.au/v1/<apikey>/updates/24/1`

An example response of updated uuids is listed to the right.


In addition to the list of updated UUIDS (which may be zero), the response also provides the UTC time that was used when doing the search for you.

With the list of UUIDs returned, you can use `/poly/download_latest` described in Step 2 to get the most recent update for each polygon.


# Authentication

> To authorize, use this code:

```python
# just pass the correct API Key with each request
import requests

requests.get("https://sapi.satamap.com.au/v1/mykey/...")
```

```shell
# just pass the correct API Key with each request
curl -X GET "https://sapi.satamap.com.au/v1/mykey/..."
```

```javascript
// just pass the correct API Key with each request
e.g. fetch("https://sapi.satamap.com.au/v1/mykey/...")
```

> Make sure to replace `mykey` with your API key.

SAPI uses organisation keys to allow access to the API. You can register for a new SAPI key at our [developer portal](https://sapi.satamap.com.au/signup).

SAPI expects the API key to be included in **ALL** API requests to the server in the submitted URL. All URLs should start with:

`https://sapi.satamap.com.au/v1/<orgkey>/.....`

<aside class="notice">
You must replace <code>orgkey</code> with your organisation API key.
</aside>

# API Documentation

## Request Information About An Area

```python
import requests

r = requests.get("https://sapi.satamap.com.au/v1/mykey/poly/((150.2871 -29.1527,150.2871 -29.1484,150.3044 -29.14841,150.3044 -29.1527,150.2871 -29.1527))")

if r.status_code == 200:
    data = r.json()
    availabledates = data['availabledates']
    if availabledates:
        # print first available date details
        print("Date: %s" % availabledates[0]['date'])
        print("Source: %s" % availabledates[0]['source'])
        print("Tile: %s" % availabledates[0]['tile'])
        print("Pass: %s" % availabledates[0]['pass'])
        print("Coverage: %s" % availabledates[0]['coverage'])
else:
    print('Failed: status code %d' % r.status_code)
    error = r.json()
    if 'message' in error:
        print(error['message'])
```

```shell
curl -X GET "https://sapi.satamap.com.au/v1/mykey/poly/((150.2871%20-29.1527%2C150.2871%20-29.1484%2C150.3044%20-29.14841%2C150.3044%20-29.1527%2C150.2871%20-29.1527))"
```

```javascript
fetch("https://sapi.satamap.com.au/v1/mykey/poly/((150.2871%20-29.1527%2C150.2871%20-29.1484%2C150.3044%20-29.14841%2C150.3044%20-29.1527%2C150.2871%20-29.1527))")  
  .then(  
    function(response) {  
      if (response.status != 200) {  
        console.log('Error Status Code: ' + response.status);  
        // check body for error message
        response.json().then(function(error) {
          if ("message" in error) {
            console.log(error.message);
          }
        });  
        return;  
      }
      // examine successful response  
      response.json().then(function(data) {  
        console.log("Retrieved " + data.availabledates.length + " coverage dates");
        console.log("Most recent date: " + data.availabledates[0].date);  
        console.log("Source: " + data.availabledates[0].source);  
        console.log("Coverage: " + data.availabledates[0].coverage);  
        console.log("Tile: " + data.availabledates[0].tile);  
      });  
    }  
  )  
  .catch(function(err) {  
    console.log('Fetch Error:', err);  
  });
```

> This URI returns JSON structured like this:

```json
{
  "availabledates": [
    {
      "tile": "t56jkn",
      "source": "s2",
      "pass": 0,
      "date": "2016-12-29",
      "coverage": 1
    },
    {
      "tile": "t56jkn",
      "source": "s2",
      "pass": 0,
      "date": "2016-12-26",
      "coverage": 1
    },
    {
      "tile": "t56jkn",
      "source": "s2",
      "pass": 0,
      "date": "2016-12-19",
      "coverage": 1
    },
    {
      "tile": "t56jkn",
      "source": "s2",
      "pass": 0,
      "date": "2016-12-09",
      "coverage": 1
    },
    {
      "tile": "t56jkn",
      "source": "s2",
      "pass": 0,
      "date": "2016-12-06",
      "coverage": 0.807296
    }
  ]
}
```

This endpoint retrieves all available coverage information about the polygon specified from Sentinel 2 and Landsat 8. The URI consists of api key, **/poly/**, followed by a series of longitude-latitude
pairs in [Well-Known Text format](https://en.wikipedia.org/wiki/Well-known_text).

### HTTP Request

`GET https://sapi.satamap.com.au/v1/<apikey>/poly/<wkt-coordinates>`

### URL Parameters

Parameter | Description
--------- | -----------
API Key | Your API key
WKT coordinates | The bounding longitude-latitude pairs that define the area e.g. ((150.2871%20-29.1527,150.2871%20-29.1484,150.3044%20-29.14841,150.3044%20-29.1527,150.2871%20-29.1527)) Spaces are encoded.

### Query Parameters

None

## Save An Area for Later Use

```python
import requests

r = requests.post("https://sapi.satamap.com.au/v1/mykey/poly/((150.2871 -29.1527,150.2871 -29.1484,150.3044 -29.14841,150.3044 -29.1527,150.2871 -29.1527))")

if r.status_code == 200:
    data = r.json()
    print(data['message'])
    print(data['uuid'])
else:
    print('Failed: status code %d' % r.status_code)
    error = r.json()
    if 'message' in error:
        print(error['message'])
```

```shell
curl -X POST "https://sapi.satamap.com.au/v1/mykey/poly/((150.2871%20-29.1527%2C150.2871%20-29.1484%2C150.3044%20-29.14841%2C150.3044%20-29.1527%2C150.2871%20-29.1527))"
```

```javascript
// use an init for a POST. Otherwise, exactly the same as a GET.
var myInit = { method: 'POST' };
fetch("https://sapi.satamap.com.au/v1/mykey/poly/((150.2871%20-29.1527%2C150.2871%20-29.1484%2C150.3044%20-29.14841%2C150.3044%20-29.1527%2C150.2871%20-29.1527))", init)  
  .then(  
    function(response) {  
      if (response.status != 200) {  
        console.log('Error Status Code: ' + response.status);  
        // check body for error message
        response.json().then(function(error) {
          if ("message" in error) {
            console.log(error.message);
          }
        });  
        return;  
      }
      // examine successful response  
      response.json().then(function(data) {  
        console.log(data.message);
        console.log("UUID: " + data.uuid);  
      });  
    }  
  )  
  .catch(function(err) {  
    console.log('Fetch Error:', err);  
  });
```

> This URI returns JSON structured like this:

```json
{
  "message": "polygon created",
  "uuid": "123e4567-e89b-12d3-a456-426655440000"
}
```

This endpoint allows you to save a predefined area in the SAPI system. It returns a confirmation message and a Universally unique identifer]
(https://en.wikipedia.org/wiki/Universally/_unique_identifier), hereon referred to as a UUID.

The syntax is exactly the same as for getting information about an area however POST is used instead of GET to create the area and it's UUID.

### HTTP Request

`POST https://sapi.satamap.com.au/v1/<apikey>/poly/<wkt-coordinates>`

### URL Parameters

Parameter | Description
--------- | -----------
API Key | Your API key
WKT coordinates | The bounding longitude-latitude pairs that define the area e.g. ((150.2871%20-29.1527,150.2871%20-29.1484,150.3044%20-29.14841,150.3044%20-29.1527,150.2871%20-29.1527)) Spaces are encoded.

### Query Parameters

None

## Get All Saved Areas

```python
import requests

r = requests.get("https://sapi.satamap.com.au/v1/mykey/polys")

if r.status_code == 200:
    data = r.json()
    for uuid in data['uuids']:
        print(uuid)
else:
    print('Failed: status code %d' % r.status_code)
    error = r.json()
    if 'message' in error:
        print(error['message'])
```

```shell
curl -X GET "https://sapi.satamap.com.au/v1/mykey/polys"
```

```javascript
fetch("https://sapi.satamap.com.au/v1/mykey/polys")  
  .then(  
    function(response) {  
      if (response.status != 200) {  
        console.log('Error Status Code: ' + response.status);  
        // check body for error message
        response.json().then(function(error) {
          if ("message" in error) {
            console.log(error.message);
          }
        });  
        return;  
      }
      // examine successful response  
      response.json().then(function(data) {  
        if ("uuids" in data) {
          for (i=0; i < data.uuids.length; i++) {
            console.log(data.uuids[i]);
          }
        } else {
          console.log("No polygons defined yet");
        }
      });  
    }  
  )  
  .catch(function(err) {  
    console.log('Fetch Error:', err);  
  });
```

> This URI returns JSON structured like this:

```json
{
  "uuids": [
    "6ad1a06f-aa95-43b8-8331-28fc59e42daa",
    "64257a32-4159-4c30-af98-dc185374324a",
    "6653c7a6-ad05-48a9-a16b-aa53ee7a7a8f",
    "36a39fe0-f6da-4051-819e-715434519a71"
  ]
}
```

This endpoint allows you to get the UUIDs of all polygons that you have predefined.


### HTTP Request

`GET https://sapi.satamap.com.au/v1/<apikey>/polys`

### URL Parameters

Parameter | Description
--------- | -----------
API Key | Your API key

### Query Parameters

Query parameters are optional parameters appended to the URL

Parameter | Default | Description
--------- | ------- | -----------
extref | false | If set to true, return all uuids supplied by you

## Get A Saved Area's Dates

```python
import requests

r = requests.get("https://sapi.satamap.com.au/v1/mykey/poly/availabledates/90722b22-8ea2-4e2a-be45-ec943afe9e2c")

if r.status_code == 200:
    data = r.json()
    availabledates = data['availabledates']
    if availabledates:
        # print first available date details
        print("Date: %s" % availabledates[0]['date'])
        print("Source: %s" % availabledates[0]['source'])
        print("Tile: %s" % availabledates[0]['tile'])
        print("Pass: %s" % availabledates[0]['pass'])
        print("Coverage: %s" % availabledates[0]['coverage'])
else:
    print('Failed: status code %d' % r.status_code)
    error = r.json()
    if 'message' in error:
        print(error['message'])
```

```shell
curl -X GET "https://sapi.satamap.com.au/v1/mykey/poly/availabledates/90722b22-8ea2-4e2a-be45-ec943afe9e2c"
```

```javascript
fetch("https://sapi.satamap.com.au/v1/mykey/poly/availabledates/90722b22-8ea2-4e2a-be45-ec943afe9e2c")  
  .then(  
    function(response) {  
      if (response.status != 200) {  
        console.log('Error Status Code: ' + response.status);  
        // check body for error message
        response.json().then(function(error) {
          if ("message" in error) {
            console.log(error.message);
          }
        });  
        return;  
      }
      // Examine the response  
      response.json().then(function(data) {  
        console.log("Retrieved " + data.availabledates.length + " coverage dates");
        console.log("Most recent date: " + data.availabledates[0].date);  
        console.log("Source: " + data.availabledates[0].source);  
        console.log("Coverage: " + data.availabledates[0].coverage);  
        console.log("Tile: " + data.availabledates[0].tile);  
      });  
    }  
  )  
  .catch(function(err) {  
    console.log('Fetch Error:', err);  
  });
```

> This URI returns JSON structured like this:

```json
{
  "availabledates": [
    {
      "tile": "t091080",
      "source": "l8",
      "pass": 0,
      "date": "2016-11-27",
      "coverage": 1
    },
    {
      "tile": "t56jkn",
      "source": "s2",
      "pass": 0,
      "date": "2016-11-26",
      "coverage": 1
    },
    {
      "tile": "t55jgh",
      "source": "s2",
      "pass": 0,
      "date": "2016-11-19",
      "coverage": 1
    }
{
```

This endpoint allows you to get all coverage dates of a specific area by referencing it's UUID.


### HTTP Request

`GET https://sapi.satamap.com.au/v1/<apikey>/poly/availabledates/<uuid>`

### URL Parameters

Parameter | Description
--------- | -----------
API Key | Your API key
UUID | e.g. 90722b22-8ea2-4e2a-be45-ec943afe9e2c`

### Query Parameters

Query parameters are optional parameters appended to the URL

Parameter | Default | Description
--------- | ------- | -----------
extref | false | If set to true, regard uuid as defined by you

## Get An Area As GeoJSON

```python
import requests

r = requests.get("https://sapi.satamap.com.au/v1/mykey/poly/geojson/90722b22-8ea2-4e2a-be45-ec943afe9e2c")

if r.status_code == 200:
    data = r.json()
    print("GeoJSON Type %s" % data['type'])
    print(data['coordinates'][0][0])
else:
    print('Failed: status code %d' % r.status_code)
    error = r.json()
    if 'message' in error:
        print(error['message'])
```

```shell
curl -X GET "http://sapi.satamap.com.au/v1/mykey/poly/geojson/90722b22-8ea2-4e2a-be45-ec943afe9e2c"
```

```javascript
fetch("https://sapi.satamap.com.au/v1/mykey/poly/geojson/90722b22-8ea2-4e2a-be45-ec943afe9e2c")  
  .then(  
    function(response) {  
      if (response.status != 200) {  
        console.log('Error Status Code: ' + response.status);  
        // check body for error message
        response.json().then(function(error) {
          if ("message" in error) {
            console.log(error.message);
          }
        });  
        return;  
      }
      // examine successful response  
      response.json().then(function(data) {  
        console.log(data.type);
        console.log(data.coordinates[0][0]);  
      });  
    }  
  )  
  .catch(function(err) {  
    console.log('Fetch Error:', err);  
  });
```

> This URI returns JSON structured like this:

```json
{
  "type": "MultiPolygon",
  "coordinates": [
    [
      [
        150.1032,
        -29.18915
      ],
      [
        150.09951,
        -29.21072
      ],
      [
        150.10127,
        -29.21147
      ],
      [
        150.1023,
        -29.20994
      ],
      [
        150.1074,
        -29.21069
      ]
    ]
  ]
}
```

This endpoint provides the GeoJSON for a predefined area.


### HTTP Request

`GET https://sapi.satamap.com.au/v1/<apikey>/poly/geojson/<uuid>`

### URL Parameters

Parameter | Description
--------- | -----------
API Key | Your API key
UUID | e.g. 90722b22-8ea2-4e2a-be45-ec943afe9e2c`

### Query Parameters

Query parameters are optional parameters appended to the URL

Parameter | Default | Description
--------- | ------- | -----------
extref | false | If set to true, regard uuid as defined by you

## Get Areas Updated

```python
import requests

r = requests.get("https://sapi.satamap.com.au/v1/mykey/updates/24/1")

if r.status_code == 200:
    data = r.json()
    print('Updated since %s' % data['updated'])
    for uuid in data['uuids']:
        print(uuid)
else:
    print('Failed: status code %d' % r.status_code)
    error = r.json()
    if 'message' in error:
        print(error['message'])
```

```shell
# Get all updates of full coverage from November 1st, 2016
curl -X GET "https://sapi.satamap.com.au/v1/mykey/updates/20161101/1"
```

```javascript
// fetch all full coverage updates in the last 24 hours
fetch("https://sapi.satamap.com.au/v1/mykey/updates/24/1")  
  .then(  
    function(response) {  
      if (response.status != 200) {  
        console.log('Error Status Code: ' + response.status);  
        // check body for error message
        response.json().then(function(error) {
          if ("message" in error) {
            console.log(error.message);
          }
        });  
        return;  
      }
      // examine successful response  
      response.json().then(function(data) {  
        console.log("Updated from: " + data.updated);
        for (i=0; i < data.uuids.length; i++) {
          console.log(data.uuids[i]);  
        }
      });  
    }  
  )  
  .catch(function(err) {  
    console.log('Fetch Error:', err);  
  });
```

> This URI returns JSON structured like this:

```json
{
  "updated": "2017-01-06 11:33:34 UTC",
  "uuids": [
    "a8e7937c-6361-4530-85ce-0432a9bb9e2e",
    "6027446d-04c3-445d-8397-1de0a7983a54",
    "6920e27e-1554-4378-b735-4d68588e4a9a",
    "d84045d4-a4e5-40cc-913f-85bb20dc324c",
    "8aa12952-0b25-406c-ba11-ec7a9575b21b",
    "e7e61227-1a33-46ac-9af9-bbf5b8fb5199",
    "0ee66046-e3a7-4953-9f6f-204952dfcd92",
    "7545a96c-0fcd-40ec-a205-7bccbca4e517",
    "a23dc9e1-0199-42f0-a132-11e28bcf8dd9",
    "7dc16a6b-2586-4131-867a-bbf2a693cc0d",
    "8b0e3382-b7f0-432e-a6e8-7ebc3df7f065"
  ]
}
```

This endpoint provides all UUIDs of areas that have received an update since a given date.


### HTTP Request

`GET https://sapi.satamap.com.au/v1/<apikey>/updates/<date>/<coverage>`


### URL Parameters

Parameter | Description
--------- | -----------
API Key | Your API key
Date | Date can be specified as either YYYYMMDD or YYYYMMDDHHMMSS or an integer of the number of hours since now. i.e. for all updates in the last 24 hours, use 24. Note that the *updated* field in the JSON response specifies the date used in UTC.
Coverage | The minimum coverage required. Coverage should be between 0 and 1 i.e. 0% to 100%. If in doubt, use 1.

### Query Parameters

Query parameters are optional parameters appended to the URL

Parameter | Default | Description
--------- | ------- | -----------
extref | false | If set to true, provide uuids as defined by you

## Download Latest Area Image

```python
import requests
from io import BytesIO
from PIL import Image # Python pillow

r = requests.get("https://sapi.satamap.com.au/v1/mykey/poly/download_latest/90722b22-8ea2-4e2a-be45-ec943afe9e2c/1/rgb?source=s2")

if r.status_code == 200:
    # Content-Disposition suggests a filename to use
    # usually of the form 'attachment; filename="RGB_2017-01-30.tif"'
    if 'Content-Disposition' in r.headers:
        attachment = r.headers['Content-Disposition']
        filename = attachment[attachment.find("filename=")+10:-1]

    tif = Image.open(BytesIO(r.content))
    tif.save(filename)
else:
    print('Failed: status code %d' % r.status_code)
    error = r.json()
    if 'message' in error:
        print(error['message'])
```

```shell
# Get the most recent RGB TIF of area UUID 05157fea-6d7b-4c5b-86ac-48124cfd393c with 100% coverage
curl -X GET "https://sapi.satamap.com.au/v1/mykey/poly/download_latest/05157fea-6d7b-4c5b-86ac-48124cfd393c/1/rgb"
```

```javascript
fetch("https://sapi.satamap.com.au/v1/mykey/poly/download_latest/90722b22-8ea2-4e2a-be45-ec943afe9e2c/1/RGB")
  .then(  
    function(response) {  
      if (response.status != 200) {  
        console.log('Error Status Code: ' + response.status);  
        // check body for error message
        response.json().then(function(error) {
          if ("message" in error) {
            console.log(error.message);
          }
        });  
        return;  
      }
      // examine successful response  
      response.blob().then(function(blob) {  
        console.log("Filesize is: " + blob.size);
      });  
    }  
  )  
  .catch(function(err) {  
    console.log('Fetch Error:', err);  
  });
```

> This URI returns a TIF image.


This endpoint provides the latest TIF image of the area specified.

For testing, these URLs can be entered into most browsers, which will save the file as *filetype*_*date*.*tif*.


### HTTP Request

`GET https://sapi.satamap.com.au/v1/<apikey>/poly/download_latest/<uuid>/<coverage>/<filetype>`"

### URL Parameters

Parameter | Description
--------- | -----------
API Key | Your API key
UUID | e.g. 90722b22-8ea2-4e2a-be45-ec943afe9e2c`
Coverage | Coverage should be between 0 and 1 (i.e. 0% to 100%).
Data Type | Data type can be one of: RGB, NDVI, NDRE, SVI or PCD. See below definitions.

*RGB* - a regular visible TIF file

*NDVI* - Normalized Difference Vegetation Index

*NDRE* - Normalized Difference Red Edge Index. NDRE is only available with source parameter of s2 (i.e. Sentinel)

*SVI*  - Satamap Vegetation Index

*PCD*  - Plant Cell Density

### Query Parameters

Query parameters are optional parameters appended to the URL

Parameter | Default | Description
--------- | ------- | -----------
source | all | If set to s2, only Sentinel 2 coverage is provided. Set to l8 for Landsat 8 coverage only.
nomask | false | If set to true, partial coverage images will not have cloud masked out
fixedge | 0 | If set to a positive integer and NDVI or PCD requested, improves edge boundary in meters of coarse resolution.
extref | false | If set to true, use uuid defined by you

i.e. `?source=s2` can be appended to the URL

## Download Historical Area Image

```python
import requests
from io import BytesIO
from PIL import Image # Python pillow

r = requests.get("https://sapi.satamap.com.au/v1/mykey/poly/download/90722b22-8ea2-4e2a-be45-ec943afe9e2c/20161126/s2/t56jkn/0/RGB/tif")

if r.status_code == 200:
    # Content-Disposition suggests a filename to use
    # usually of the form 'attachment; filename="RGB_2016-11-26.tif"'
    if 'Content-Disposition' in r.headers:
        attachment = r.headers['Content-Disposition']
        filename = attachment[attachment.find("filename=")+10:-1]

    tif = Image.open(BytesIO(r.content))
    tif.save(filename)
else:
    print('Failed: status code %d' % r.status_code)
    error = r.json()
    if 'message' in error:
        print(error['message'])
```

```shell
# Get an RGB TIF of area UUID 90722b22-8ea2-4e2a-be45-ec943afe9e2c on 2016-11-26 from Sentinel 2 on pass 0 of tile t56jkn
curl -X GET "https://sapi.satamap.com.au/v1/mykey/poly/download/90722b22-8ea2-4e2a-be45-ec943afe9e2c/20161126/s2/t56jkn/0/RGB/tif"
```

```javascript
fetch("https://sapi.satamap.com.au/v1/mykey/poly/download/90722b22-8ea2-4e2a-be45-ec943afe9e2c/20161126/s2/t56jkn/0/RGB/tif")
  .then(  
    function(response) {  
      if (response.status != 200) {  
        console.log('Error Status Code: ' + response.status);  
        // check body for error message
        response.json().then(function(error) {
          if ("message" in error) {
            console.log(error.message);
          }
        });  
        return;  
      }
      // examine successful response  
      response.blob().then(function(blob) {  
        console.log("Filesize is: " + blob.size);
      });  
    }  
  )  
  .catch(function(err) {  
    console.log('Fetch Error:', err);  
  });
```

> This URI returns a TIF image.


This endpoint provides a TIF image of the area from the date, tile, pass and satellite specified.

For testing, these URLs can be entered into most browsers, which will save the file as *filetype*_*date*.*tif*.


### HTTP Request

`GET https://sapi.satamap.com.au/v1/<apikey>/poly/download/<uuid>/<date>/<source>/<tile>/<pass>/<datatype>/<filetype>`

### URL Parameters

Parameter | Description
--------- | -----------
API Key | Your API key
UUID | e.g. 90722b22-8ea2-4e2a-be45-ec943afe9e2c`
Date | Date should be specified as YYYYMMDD.
Source | Source should be either 's2' or 'l8'.
Tile | Tile is a specific tile e.g. 't56jkn'
Pass | Pass is generally 0, but can be a higher integer e.g. 1 or 2 if multiple passes have ocurred.
Data Type | Data type can be one of: RGB, NDVI, NDRE, SVI or PCD. See below definitions.
File Type | File type can be either 'tif' or 'csv'. Usually it will be tif.

**Data Types:**

*RGB* - a regular visible RGB file

*NDVI* - Normalized Difference Vegetation Index

*NDRE* - Normalized Difference Red Edge Index. NDRE is only available with source parameter of s2 (i.e. Sentinel)

*SVI*  - Satamap Vegetation Index

*PCD*  - Plant Cell Density

**File Types:**

*tif* - a TIF file. The most frequently used value as the data types listed above are all delivered in a TIF format file.

*csv* - a CSV (Comma Seperated) list of all the boundary points for the polygon where each row defines an x, y, z point

### Query Parameters

Query parameters are optional parameters appended to the URL

Parameter | Default | Description
--------- | ------- | -----------
nomask | false | If set to true, partial coverage images will not have cloud masked out
fixedge | 0 | If set to a positive integer and NDVI or PCD requested, improves edge boundary in meters of coarse resolution.
extref | false | If set to true, use uuid defined by you

i.e. `?nomask=true` can be appended to the URL


## Get An Area As KML

```python
import requests

r = requests.get("https://sapi.satamap.com.au/v1/mykey/poly/kml/90722b22-8ea2-4e2a-be45-ec943afe9e2c")

if r.status_code == 200:
    # body text is KML
    print(r.text)
else:
    print('Failed: status code %d' % r.status_code)
    error = r.json()
    if 'message' in error:
        print(error['message'])
```

```shell
# Get an KML file of area UUID 90722b22-8ea2-4e2a-be45-ec943afe9e2c
curl -X GET "https://sapi.satamap.com.au/v1/mykey/poly/kml/90722b22-8ea2-4e2a-be45-ec943afe9e2c"
```

```javascript
fetch("https://sapi.satamap.com.au/v1/mykey/poly/kml/90722b22-8ea2-4e2a-be45-ec943afe9e2c")  
  .then(  
    function(response) {  
      if (response.status != 200) {  
        console.log('Error Status Code: ' + response.status);  
        // check body for error message
        response.json().then(function(error) {
          if ("message" in error) {
            console.log(error.message);
          }
        });  
        return;  
      }
      // body text is KML
      response.text().then(function(data) {  
        console.log(data);
      });  
    }  
  )  
  .catch(function(err) {  
    console.log('Fetch Error:', err);  
  });
```

> The above command returns a KML file.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<MultiGeometry>
    <Polygon>
        <outerBoundaryIs>
            <LinearRing>
                <coordinates>150.1032,-29.18915 150.09951,-29.21072 150.10127,-29.21147 150.1023,-29.20994 150.1074,-29.21069 150.10775,-29.21151 150.11187,-29.21061 150.11478,-29.21159 150.1156,-29.2114 150.1159,-29.21031 150.11757,-29.19994 150.11629,-29.19949 150.11757,-29.1993 150.11826,-29.19537 150.11796,-29.19537 150.1174,-29.19589 150.11744,-29.19522 150.11607,-29.1951 150.11427,-29.19405 150.11268,-29.19271 150.11024,-29.19226 150.10933,-29.19121 150.10637,-29.18956 150.1047,-29.18922 150.1032,-29.18915</coordinates>
            </LinearRing>
        </outerBoundaryIs>
        <innerBoundaryIs>
            <LinearRing>
                <coordinates>150.11313,-29.20668 150.11137,-29.2073 150.11167,-29.20647 150.11313,-29.20668</coordinates>
            </LinearRing>
        </innerBoundaryIs>
    </Polygon>
</MultiGeometry>
```

This endpoint provides a KML file that defines the area.

For testing, these URLs can be entered into most browsers, which will save the file as *date*.*kml*.


### HTTP Request

`GET https://sapi.satamap.com.au/v1/<apikey>/poly/kml/<uuid>`

### URL Parameters

Parameter | Description
--------- | -----------
API Key | Your API key
UUID | e.g. 90722b22-8ea2-4e2a-be45-ec943afe9e2c`

### Query Parameters

Query parameters are optional parameters appended to the URL

Parameter | Default | Description
--------- | ------- | -----------
extref | false | If set to true, regard uuid as defined by you

## Save An Area with specific UUID

```python
import requests

r = requests.post("https://sapi.satamap.com.au/v1/mykey/poly/37fa49f3-8ef4-4f03-bbef-29b491637bd2/((150.2871 -29.1527,150.2871 -29.1484,150.3044 -29.14841,150.3044 -29.1527,150.2871 -29.1527))")

if r.status_code == 200:
    data = r.json()
    print(data['message'])
    print(data['uuid'])
else:
    print('Failed: status code %d' % r.status_code)
    error = r.json()
    if 'message' in error:
        print(error['message'])
```

```shell
curl -X POST "https://sapi.satamap.com.au/v1/mykey/poly/37fa49f3-8ef4-4f03-bbef-29b491637bd2/((150.2871%20-29.1527%2C150.2871%20-29.1484%2C150.3044%20-29.14841%2C150.3044%20-29.1527%2C150.2871%20-29.1527))"
```

```javascript
// use an init for a POST. Otherwise, exactly the same as a GET.
var myInit = { method: 'POST' };
fetch("https://sapi.satamap.com.au/v1/mykey/poly/37fa49f3-8ef4-4f03-bbef-29b491637bd2/((150.2871%20-29.1527%2C150.2871%20-29.1484%2C150.3044%20-29.14841%2C150.3044%20-29.1527%2C150.2871%20-29.1527))", init)  
  .then(  
    function(response) {  
      if (response.status != 200) {  
        console.log('Error Status Code: ' + response.status);  
        // check body for error message
        response.json().then(function(error) {
          if ("message" in error) {
            console.log(error.message);
          }
        });  
        return;  
      }
      // examine successful response  
      response.json().then(function(data) {  
        console.log(data.message);
        console.log("UUID: " + data.uuid);  
      });  
    }  
  )  
  .catch(function(err) {  
    console.log('Fetch Error:', err);  
  });
```

> This URI returns JSON structured like this:

```json
{
  "message": "polygon created",
  "uuid": "37fa49f3-8ef4-4f03-bbef-29b491637bd2"
}
```

This endpoint allows you to save a predefined area in the SAPI system to be referred to with a supplied UUID i.e. it essentially allows you to use your own UUIDs. The URI returns a confirmation message and a confirmation of the  Universally unique identifer
(https://en.wikipedia.org/wiki/Universally/_unique_identifier), hereon referred to as a UUID.

The syntax is exactly the same as for getting information about an area however POST is used instead of GET to create the area and it's UUID.

<aside class="notice">
By default, the UUIDs supplied to SAPI URIs are expected to refer to a UUID produced by SAPI for you. If you have used this URI to specify a new polygon with your own UUID,
on other URI requests where you request information about that polygon, you <b>must add the querystring</b> <code>extref=true</code> to tell SAPI to look in your
table of UUIDs rather than the internal UUIDs. Failure to add this query string will likely result in an <i>Invalid UUID</i> response.
</aside>

<aside class="warning">
If you choose to use this URI method for creating polygons, you must be consistent and use only this method for creating polygons.
</aside>

### HTTP Request

`POST https://sapi.satamap.com.au/v1/<apikey>/poly/<uuid>/<wkt-coordinates>`

### URL Parameters

Parameter | Description
--------- | -----------
API Key | Your API key
UUID | A UUID created by you to refer to this polygon.
WKT coordinates | The bounding longitude-latitude pairs that define the area e.g. ((150.2871%20-29.1527,150.2871%20-29.1484,150.3044%20-29.14841,150.3044%20-29.1527,150.2871%20-29.1527)) Spaces are encoded.

### Query Parameters

None

## Delete a Specific Kitten

```ruby
require 'kittn'

api = Kittn::APIClient.authorize!('meowmeowmeow')
api.kittens.delete(2)
```

```python
import kittn

api = kittn.authorize('meowmeowmeow')
api.kittens.delete(2)
```

```shell
curl "http://example.com/api/kittens/2"
  -X DELETE
  -H "Authorization: meowmeowmeow"
```

```javascript
const kittn = require('kittn');

let api = kittn.authorize('meowmeowmeow');
let max = api.kittens.delete(2);
```

> The above command returns JSON structured like this:

```json
{
  "id": 2,
  "deleted" : ":("
}
```

This endpoint deletes a specific kitten.

### HTTP Request

`DELETE http://example.com/kittens/<ID>`

### URL Parameters

Parameter | Description
--------- | -----------
ID | The ID of the kitten to delete

