# OpenCage Geocoding API — Node.js

This skill covers Node.js-specific usage of the OpenCage Geocoding API.
For general, language-agnostic concepts (endpoint, parameters, response structure, error codes, confidence scores, annotations, rate limits, test keys), refer to **opencage-geocoding-api/SKILL.md** first.

## Installation

```bash
npm install opencage-api-client
```

The `opencage-api-client` package is the recommended Node.js client for the OpenCage API. It supports CommonJS, ES modules, and TypeScript, and returns Promises for all geocoding calls.

## Basic Usage

### CommonJS

```javascript
require('dotenv').config();
const opencage = require('opencage-api-client');

// Forward geocoding (address → coordinates)
opencage.geocode({ q: 'Berlin, Germany', no_annotations: 1 })
  .then((data) => {
    if (data.results.length > 0) {
      const { lat, lng } = data.results[0].geometry;
      const formatted = data.results[0].formatted;
      console.log(`${lat}, ${lng} — ${formatted}`);
    } else {
      console.log('No results found');
    }
  })
  .catch((error) => {
    console.error('Error:', error.message);
  });

// Reverse geocoding (coordinates → address) — pass lat,lng as a string
opencage.geocode({ q: '51.5074,-0.1278', no_annotations: 1 })
  .then((data) => {
    if (data.results.length > 0) {
      console.log(data.results[0].formatted);
    }
  });
```

### ES Modules

```javascript
import 'dotenv/config';
import opencage from 'opencage-api-client';

const data = await opencage.geocode({ q: 'Berlin, Germany', no_annotations: 1 });
if (data.results.length > 0) {
  const { lat, lng } = data.results[0].geometry;
  console.log(`${lat}, ${lng}`);
}
```

The library reads the API key from the `OPENCAGE_API_KEY` environment variable automatically — no need to pass it explicitly unless you want to override it.

## Passing Optional Parameters

All optional API parameters (documented in opencage-geocoding-api/SKILL.md) are passed as properties of the options object:

```javascript
const data = await opencage.geocode({
  q: 'Brandenburg Gate',
  countrycode: 'de',
  language: 'de',
  limit: 1,
  no_annotations: 1,
});
```

**Always pass `no_annotations: 1`** when you only need coordinates or the formatted address — it reduces response size and latency.

## Checking the Response Status

The resolved value is the full API response object. Check `status.code` before trusting the results:

```javascript
const data = await opencage.geocode({ q: 'Berlin, Germany', no_annotations: 1 });

if (data.status.code !== 200) {
  console.error(`API error ${data.status.code}: ${data.status.message}`);
} else if (data.results.length === 0) {
  console.log('No results found');
} else {
  console.log(data.results[0].formatted);
}
```

Common status codes and their meanings are documented in **opencage-geocoding-api/SKILL.md**. Key ones to handle:

- `200` — OK
- `401` — Invalid or missing API key
- `402` — Quota exceeded
- `403` — Key suspended
- `429` — Too many requests (per-second rate limit hit)

## Error Handling

Network errors and API errors (4xx/5xx) are thrown as JavaScript `Error` objects and must be caught:

```javascript
import opencage from 'opencage-api-client';

async function geocodeAddress(address) {
  try {
    const data = await opencage.geocode({ q: address, no_annotations: 1, limit: 1 });

    if (data.status.code === 200 && data.results.length > 0) {
      return data.results[0];
    } else if (data.status.code === 429) {
      throw new Error('Rate limit exceeded');
    } else {
      console.warn(`No results for "${address}": ${data.status.message}`);
      return null;
    }
  } catch (error) {
    console.error('Geocoding failed:', error.message);
    throw error;
  }
}
```

## TypeScript

The package ships with TypeScript type definitions:

```typescript
import { geocode } from 'opencage-api-client';
import type { GeocodingRequest, GeocodingResponse } from 'opencage-api-client';

const request: GeocodingRequest = {
  q: 'Berlin, Germany',
  no_annotations: 1,
  limit: 1,
};

const data: GeocodingResponse = await geocode(request);
if (data.results.length > 0) {
  const { lat, lng } = data.results[0].geometry;
  console.log(lat, lng);
}
```

## Accessing Components Safely

`components` fields are not guaranteed to be present for every location (see opencage-geocoding-api/SKILL.md — "Results Reflect the Real World"). Use optional chaining and nullish coalescing:

```javascript
const components = data.results[0]?.components ?? {};

const country  = components.country ?? null;
const city     = components.city ?? components.town ?? components.village ?? null;
const postcode = components.postcode ?? null;
const type     = components._type ?? null;
```

## Batch Geocoding

For processing a list of addresses, add a delay between requests to respect the 1 req/s rate limit on free-trial accounts:

```javascript
import opencage from 'opencage-api-client';

const sleep = (ms) => new Promise((resolve) => setTimeout(resolve, ms));

async function batchGeocode(addresses) {
  const results = [];

  for (const address of addresses) {
    try {
      const data = await opencage.geocode({ q: address, no_annotations: 1, limit: 1 });

      if (data.status.code === 200 && data.results.length > 0) {
        const { lat, lng } = data.results[0].geometry;
        results.push({ input: address, lat, lng, formatted: data.results[0].formatted });
      } else if (data.status.code === 429) {
        console.warn('Rate limit hit — pausing');
        await sleep(2000);
      } else {
        results.push({ input: address, error: data.status.message });
      }
    } catch (error) {
      results.push({ input: address, error: error.message });
    }

    await sleep(1000);  // 1 req/s for free-trial accounts; remove for paid subscriptions
  }

  return results;
}

const addresses = ['Berlin, Germany', 'Paris, France', 'London, UK'];
const output = await batchGeocode(addresses);
console.log(output);
```

## Environment Variables

The library automatically reads `OPENCAGE_API_KEY` from `process.env`. Store the key in a `.env` file during development and load it with `dotenv`:

```bash
# .env
OPENCAGE_API_KEY=your-api-key-here
```

```javascript
// At the top of your entry file
import 'dotenv/config';

// Then use opencage-api-client normally — it picks up the key automatically
import opencage from 'opencage-api-client';
```

Never commit `.env` files to version control. In production, set the environment variable through your platform's secrets management.

## Further Reading

- OpenCage Node.js tutorial: https://opencagedata.com/tutorials/geocode-in-nodejs
- `opencage-api-client` on npm: https://www.npmjs.com/package/opencage-api-client
- Source code: https://github.com/tsamaya/opencage-api-client
- General API reference: **opencage-geocoding-api/SKILL.md**
