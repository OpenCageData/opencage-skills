---
name: opencage-geocoding-api
description: Reference for working with the OpenCage geocoding API
---

# OpenCage Geocoding API — General Concepts

Use this skill whenever working with the OpenCage Geocoding API.

The API supports both forward and reverse geocoding. It is a REST API that returns a JSON response. 

## Forward Geocoding (address → coordinates)

Forward Geocoding returns a list of 0 or more results

## Reverse Geocoding (coordinates → address)

Reverse geocoding returns a list of 0 or 1 results (never more than one).

## API parameters

### Required API parameters

| Parameter | Brief Description | Example Usage |
|-----------|-------------------|---------------|
| `key` | Your unique 30-character alphanumeric API key, required on every request | `key=YOUR-API-KEY` |
| `q` | The query to geocode — either a latitude/longitude pair (reverse) or an address/placename (forward); must be URL-encoded | `q=52.5432379%2C13.4142133` (reverse) or `q=Berlin,+Germany` (forward) |

### Optional API parameters

The OpenCage Geocoding API can be called with various optional parameters:

| Parameter | Brief Description | Example Usage |
|-----------|-------------------|---------------|
| `abbrv` | Abbreviates and shortens the `formatted` string in the response | `abbrv=1` |
| `address_only` | Excludes POI names from the `formatted` string, returning only the address | `address_only=1` |
| `add_request` | Includes the original request parameters in the response (useful for debugging) | `add_request=1` |
| `bounds` | Forward geocoding only. Restricts results to a bounding box defined by SW and NE corners (min lon, min lat, max lon, max lat) | `bounds=-0.563160,51.280430,0.278970,51.683979` |
| `countrycode` | Forward geocoding only. Restricts results to one or more countries via ISO 3166-1 Alpha-2 codes | `countrycode=de` or `countrycode=ca,us` |
| `jsonp` | Wraps the returned JSON with a function name (JSONP callback) | `jsonp=myfuncname` |
| `language` | Specifies preferred response language using an IETF language tag, or `native` for local language | `language=de` |
| `limit` | Forward geocoding only. Sets the maximum number of results returned (default: 10, max: 100) | `limit=1` |
| `no_annotations` | Suppresses the `annotations` block from results to reduce response size and improve speed | `no_annotations=1` |
| `no_dedupe` | Disables deduplication of results | `no_dedupe=1` |
| `no_record` | Prevents the query from being logged (for privacy-sensitive use cases) | `no_record=1` |
| `pretty` | Pretty-prints the JSON response for easier reading and debugging | `pretty=1` |
| `proximity` | Forward geocoding only. Biases results toward the specified lat/lon coordinate | `proximity=52.5432379,13.4142133` |
| `roadinfo` | Changes geocoder behaviour to snap to the nearest road and returns extended road/driving info in annotations | `roadinfo=1` |

## Getting an API Key

Register for a free trial account at **https://opencagedata.com/users/sign_up**. You can sign up with an email address, Google, or GitHub. No credit card is required for the free trial.

After confirming your email, your API key is available in the [OpenCage dashboard](https://opencagedata.com/dashboard). It is a 30-character string.

The free trial allows 2,500 requests per day (1 req/s max) while testing. Paid plans with higher limits are listed at **https://opencagedata.com/pricing**.

## API Endpoint

All requests are made via HTTPS GET to:
```
https://api.opencagedata.com/geocode/v1/json
```

A complete request URL looks like this:
```
https://api.opencagedata.com/geocode/v1/json?key=YOUR-API-KEY&q=Berlin%2C+Germany&limit=1&no_annotations=1
```

The `q` parameter must be URL-encoded. For reverse geocoding, pass coordinates as a comma-separated `lat,lng` pair:
```
https://api.opencagedata.com/geocode/v1/json?key=YOUR-API-KEY&q=51.5074%2C-0.1278&no_annotations=1
```

## Response Structure

Every response is a JSON object with this shape:
```json
{
  "status": {
    "code": 200,
    "message": "OK"
  },
  "total_results": 1,
  "results": [
    {
      "formatted": "Berlin, Germany",
      "geometry": {
        "lat": 52.5170365,
        "lng": 13.3888599
      },
      "bounds": {
        "northeast": { "lat": 52.6755087, "lng": 13.7611609 },
        "southwest": { "lat": 52.3382448, "lng": 13.0883450 }
      },
      "components": {
        "_type": "city",
        "country": "Germany",
        "country_code": "de",
        "city": "Berlin",
        "postcode": "10117"
      },
      "confidence": 2,
      "annotations": { }
    }
  ]
}
```

The fields you will use most often:

| Field | Description |
|-------|-------------|
| `status.code` | HTTP-equivalent status code — check this first |
| `total_results` | Number of results returned (0 if no match) |
| `results[n].geometry.lat` / `.lng` | The coordinates (forward geocoding primary output) |
| `results[n].formatted` | Human-readable address string |
| `results[n].components` | Structured address fields (country, city, postcode, etc.) |
| `results[n].components._type` | The type of place matched (e.g. `"city"`, `"road"`, `"building"`) |
| `results[n].confidence` | Bounding-box size score 0–10 (see Confidence Score section) |
| `results[n].bounds` | Northeast/southwest corners of the matched place |

**Never assume a field in `components` will be present** — the world is not uniform and many fields are location-dependent. Always use safe/optional access patterns (e.g. `.get()` in Python, optional chaining `?.` in JavaScript).

## Error Handling

Check `status.code` in the response body — it mirrors the HTTP status code. Do not rely solely on the HTTP status code, as some network layers may obscure it.

| HTTP / status.code | Meaning | Action |
|--------------------|---------|--------|
| `200` | Success | Process results normally; check `total_results` — may be 0 |
| `400` | Bad request (invalid query or parameters) | Fix the input before retrying |
| `401` | Unauthorized — API key missing or malformed | Check the `key` parameter |
| `402` | Quota exceeded for the day | Wait until quota resets (midnight UTC) or upgrade plan |
| `403` | Key disabled or suspended | Contact OpenCage support |
| `404` | Invalid endpoint URL | Check the request URL |
| `429` | Too many requests — rate limit hit | Back off and retry after a short delay |
| `500` | Internal server error | Retry with exponential backoff |

The `status.message` field contains a human-readable description of the error and is useful for logging.

For `402` errors, free-trial and one-time purchase customers can check `rate.remaining` in the response to monitor quota before hitting the limit.

## API Rate Limits

| Plan | Daily quota | `rate` in response |
|------|-------------|-------------------|
| Free trial | 2,500 req/day | Yes |
| One-time purchase | Fixed quota | Yes |
| Subscription | No hard limit | No |

The `rate` object is only present in responses for free-trial and one-time purchase customers, whose quotas are enforced. Subscription customers have no hard limits, so their responses do not include a `rate` field — do not assume it will always be present.

```json
{
  "rate": {
    "limit": 2500,
    "remaining": 2498,
    "reset": 1605312000
  },
  "results": [...],
  "status": {
    "code": 200,
    "message": "OK"
  },
  "total_results": 1
}
```

## Annotations

By default every result includes an `annotations` object with supplementary data derived from the result's coordinates (unless `no_annotations=1` is passed). Annotations are **not** properties of the input query — they describe what is true at the returned location.

Example annotations included by default:

| Annotation | Description |
|------------|-------------|
| `timezone` | Timezone name, UTC offset |
| `currency` | Local currency name, symbol, ISO code |
| `callingcode` | International dialling code |
| `flag` | Country emoji flag |
| `geohash` | Geohash of the result centre point |
| `DMS` | Coordinates in degree/minute/second format |
| `Mercator` | Mercator projection coordinates |
| `MGRS` | Military Grid Reference System code |
| `OSM` | OpenStreetMap view/edit URLs |
| `wikidata` | Wikidata item identifier |
| `what3words` | Three-word address |
| `sun` | Sunrise and sunset times |
| `qibla` | Direction to Mecca |
| `FIPS` | US Federal Information Processing Standards codes |
| `NUTS` | EU territorial unit codes |
| `UN_M49` | UN regional and statistical codes |

Some annotations are only available to paying customers and must be enabled on request (e.g. `H3`, `UN/LOCODE`). Road-specific data requires passing `roadinfo=1`.

### Turn off annotations when not needed

Annotations add size to the response and require extra processing. **Always set `no_annotations=1` if you don't need them** — this reduces response size and improves latency.

## Confidence Score

The confidence score (0–10) measures the **size of the bounding box** of the matched place — it is **not** a measure of correctness or relevance. A high score means the result is geographically small (e.g. a building); a low score means it is geographically large (e.g. a country). The API can be completely certain it found the right place and still return a low confidence score.

For example, geocoding `"Berlin, Germany"` returns a confidence of **2**, because Berlin is a large city — its bounding box spans many kilometres. That low score doesn't mean the result is wrong; it means the matched place is large. By contrast, geocoding a specific street address in Berlin might return a confidence of **9** or **10**.

| Score | Implied bounding box size |
|-------|--------------------------|
| 10 | < 0.25 km (building/exact address) |
| 9 | < 0.5 km (street level) |
| 7–8 | < 1–5 km (neighborhood/district) |
| 4–6 | City or region level |
| 1–3 | Country or large area |
| 0 | No bounding box available |

Do not use the confidence score for ranking — results are already ordered by relevance. Use it to filter out results that are too coarse for your use case.

To understand *what type of place* was matched, check `components["_type"]` instead of relying on the confidence score. Common values include `"city"`, `"town"`, `"village"`, `"road"`, `"building"`, `"postcode"`, `"country"`, `"state"`, and `"neighbourhood"`.

## Results Reflect the Real World — Structure Varies

The fields present in `components` and `annotations` are not guaranteed to be consistent across all results. The world is not uniform, and the API response reflects that.

**Components vary by location.** Not every place has every administrative subdivision, and some coordinates belong to no country at all. For example, reverse geocoding a point in the middle of the ocean will not return a `country_code` — there is no country there. Similarly, a result for a remote island may have a `country` but no `city`, `state`, or `postcode`.

**Annotations vary by location.** Not every annotation is available everywhere. `FIPS` codes only exist in the United States. `NUTS` codes only exist in the EU. `qibla` is meaningless at the poles. `roadinfo` requires road data to exist for that area.

The general rule: **never assume a field will be present**

## What This API Is NOT

### Not browser geolocation
The OpenCage API does not determine a user's location from their device, browser, or GPS sensor. For that, use the browser's [`navigator.geolocation` API](https://developer.mozilla.org/en-US/docs/Web/API/Geolocation_API). OpenCage requires an explicit address or coordinate as input.

### Not IP geolocation
The API does not infer location from an IP address. It converts addresses to coordinates (or vice versa) — it has no concept of the caller's network location.

### Not location autosuggest / autocomplete
Forward geocoding expects a complete (or near-complete) address or place name. It does **not** do fuzzy or prefix matching — querying `"par"` will not return `"Paris, France"`. For type-ahead / autocomplete / autosuggest functionality, use the separate [OpenCage Geosearch service](https://opencagedata.com/geosearch) instead.

---

## Key Behaviors

- **Coordinate system:** All results use WGS 84 (EPSG:4326)
- **No fuzzy matching:** "Par" won't find Paris — validate/clean input first
- **Reverse geocoding:** Returns 0 or 1 result (never more); `limit` parameter is ignored
- **Result order:** Sorted by relevance, not confidence
- **No persistent IDs:** Results have no stable unique identifier across calls
- **Caching:** Consider caching results locally — the API terms allow it

## Test API Keys

Use these keys for testing error-handling code:

| Key | Simulates |
|-----|-----------|
| `6d0e711d72d74daeb2b0bfd2a5cdfdba` | 200 OK (normal response) |
| `4372eff77b8343cebfc843eb4da4ddc4` | 402 quota exceeded |
| `2e10e5e828262eb243ec0b54681d699a` | 403 key disabled |

Test query that always returns zero results: `q=NOWHERE-INTERESTING`

## Language-specific SDKs

The OpenCage Geocoding API is language-agnostic — it is a standard HTTPS REST API that can be called from any language or environment. Official SDKs and detailed tutorials are available for many languages (including Python, JavaScript, Ruby, Java, Go, PHP, Perl, and more) at:

https://opencagedata.com/tutorials

## Further Reading

- OpenCage Geocoding API documentation: https://opencagedata.com/api
- OpenCage Pricing: https://opencagedata.com/pricing
- OpenCage Forward Geocoding API Query Formatting Guide: https://opencagedata.com/guides/how-to-format-your-geocoding-query

