# OpenCage Geocoding API — Python

This skill covers Python-specific usage of the OpenCage Geocoding API.
For general, language-agnostic concepts (endpoint, parameters, response structure, error codes, confidence scores, annotations, rate limits, test keys), refer to **opencage-geocoding-api.md** first.

## Installation

```bash
pip install opencage
```

The `opencage` package is available on PyPI and wraps the REST API with a Pythonic interface. It requires `requests` as a dependency (installed automatically).

## Basic Usage

```python
from opencage.geocoder import OpenCageGeocode

geocoder = OpenCageGeocode('YOUR-API-KEY')

# Forward geocoding (address → coordinates)
results = geocoder.geocode('Berlin, Germany')
if results:
    lat = results[0]['geometry']['lat']
    lng = results[0]['geometry']['lng']
    print(f"{lat}, {lng}")  # 52.5170365, 13.3888599

# Reverse geocoding (coordinates → address)
results = geocoder.reverse_geocode(51.5074, -0.1278)
if results:
    print(results[0]['formatted'])
```

Both methods return a list of result dicts matching the structure described in **opencage-geocoding-api.md**. Always check `if results:` before accessing `results[0]` — an empty list is returned when nothing is found.

## Passing Optional Parameters

All optional API parameters (documented in opencage-geocoding-api.md) can be passed as keyword arguments:

```python
results = geocoder.geocode(
    'Brandenburg Gate',
    countrycode='de',
    language='de',
    limit=1,
    no_annotations=1,
)
```

Parameter names match the API parameter names exactly (e.g. `no_annotations`, `countrycode`, `proximity`).

**Always pass `no_annotations=1`** when you only need coordinates or `formatted` — it reduces response size and latency.

## Error Handling

The library raises exceptions for API errors rather than returning error status codes. Import and catch them explicitly:

```python
from opencage.geocoder import (
    OpenCageGeocode,
    InvalidInputError,
    NotAuthorizedError,
    ForbiddenError,
    RateLimitExceededError,
    UnknownError,
)

geocoder = OpenCageGeocode('YOUR-API-KEY')

try:
    results = geocoder.geocode('Berlin, Germany', no_annotations=1)
    if results:
        lat = results[0]['geometry']['lat']
        lng = results[0]['geometry']['lng']
    else:
        print('No results found')

except InvalidInputError as e:
    # Bad query — fix the input
    print(f'Invalid input: {e}')

except NotAuthorizedError as e:
    # API key missing or invalid (HTTP 401)
    print(f'Invalid API key: {e}')

except ForbiddenError as e:
    # API key disabled or suspended (HTTP 403)
    print(f'Key forbidden: {e}')

except RateLimitExceededError as e:
    # Quota exceeded (HTTP 402) or rate limit hit (HTTP 429)
    print(f'Rate limit exceeded: {e}')

except UnknownError as e:
    # Unexpected server error
    print(f'Unknown error: {e}')
```

## Accessing Components Safely

`components` fields are not guaranteed to be present for every location (see opencage-geocoding-api.md — "Results Reflect the Real World"). Always use `.get()`:

```python
result = results[0]
components = result.get('components', {})

country = components.get('country')
city = components.get('city') or components.get('town') or components.get('village')
postcode = components.get('postcode')
place_type = components.get('_type')
```

The pattern `components.get('city') or components.get('town') or components.get('village')` is useful when you want the most local settlement name regardless of its administrative level.

## Batch Geocoding

For processing a list of addresses, add a short delay between requests to respect the 1 req/s rate limit on free-trial accounts:

```python
import time
from opencage.geocoder import OpenCageGeocode, RateLimitExceededError

geocoder = OpenCageGeocode('YOUR-API-KEY')

addresses = ['Berlin, Germany', 'Paris, France', 'London, UK']
results_out = []

for address in addresses:
    try:
        results = geocoder.geocode(address, no_annotations=1, limit=1)
        if results:
            results_out.append({
                'input': address,
                'lat': results[0]['geometry']['lat'],
                'lng': results[0]['geometry']['lng'],
                'formatted': results[0]['formatted'],
            })
        else:
            results_out.append({'input': address, 'error': 'no results'})
    except RateLimitExceededError:
        print('Rate limit hit — pausing')
        time.sleep(2)
    time.sleep(1)  # 1 req/s for free-trial accounts; remove for paid subscriptions
```

## Async Support

The library also provides an async geocoder for use with `asyncio`:

```python
import asyncio
from opencage.geocoder import OpenCageGeocode

async def main():
    geocoder = OpenCageGeocode('YOUR-API-KEY')
    results = await geocoder.geocode_async('Berlin, Germany', no_annotations=1)
    if results:
        print(results[0]['geometry'])
    await geocoder.close_async()

asyncio.run(main())
```

Use the async geocoder when integrating into an existing `asyncio`-based application to avoid blocking the event loop.

## Environment Variables

Never hard-code your API key. Store it in an environment variable and read it at runtime:

```python
import os
from opencage.geocoder import OpenCageGeocode

geocoder = OpenCageGeocode(os.environ['OPENCAGE_API_KEY'])
```

Or using `python-dotenv` in development:

```python
from dotenv import load_dotenv
load_dotenv()

import os
from opencage.geocoder import OpenCageGeocode

geocoder = OpenCageGeocode(os.environ['OPENCAGE_API_KEY'])
```

## Further Reading

- OpenCage Python tutorial: https://opencagedata.com/tutorials/geocode-in-python
- `opencage` package on PyPI: https://pypi.org/project/opencage/
- General API reference: **opencage-geocoding-api.md**
