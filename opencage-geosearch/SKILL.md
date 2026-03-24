---
name: opencage-geosearch
description: >-
  Use when the user needs geographic place autosuggest or autocomplete on
  a web page, or mentions "geosearch", "place autocomplete", "OpenCage
  widget", "oc_gs_ key", or "geographic autosuggest". Also use when
  integrating a location search widget with Leaflet, OpenLayers, or
  MapLibre.
---

# OpenCage Geosearch — General Concepts

OpenCage Geosearch is a **JavaScript widget** that provides geographic autosuggest/autocomplete functionality for forms. It converts partial text input into place names — countries, states, regions, cities, towns, villages, and neighbourhoods. It is built on top of Algolia's Autocomplete library.

## Quick Reference

```
Type:      Front-end JavaScript widget (not a REST API)
Key:       oc_gs_... (different from Geocoding API key)
Package:   @opencage/geosearch-bundle (CDN or npm)
Setup:     opencage.algoliaAutocomplete({ container, plugins: [OpenCageGeoSearchPlugin({ key })] })
Coords:    params.item.geometry.lat / .lng (in onSelect handler)
CORS:      MUST register domain in OpenCage dashboard before use
Covers:    Countries, cities, towns, villages, neighbourhoods — NOT street addresses or postcodes
```

## What Geosearch Is NOT

**Geosearch is not the same as the OpenCage Geocoding API.** Key differences:

- Geosearch is a front-end JavaScript widget; the Geocoding API is a back-end REST API.
- Geosearch does type-ahead / prefix matching (e.g. "Par" → "Paris, France"); the Geocoding API does not.
- Geosearch does **not** cover house addresses, postcodes, or individual roads.
- Geosearch uses a **different API key** (format: `oc_gs_...`); a Geocoding API key will not work here.

If the task requires converting a complete address or coordinates server-side, use the `opencage-geocoding-api` skill instead.

## API Key

Geosearch uses its own API key, distinct from the Geocoding API key. The key format is `oc_gs_...`.

Obtain a key from the [OpenCage dashboard](https://opencagedata.com/dashboard) after signing up at **https://opencagedata.com/users/sign_up**.

### CORS Domain Configuration — Required Step

Before the widget will work in a browser, you **must** add the domain(s) it will be called from to the allowed list in your OpenCage account dashboard. Requests from unlisted domains will be blocked. This is a common integration mistake — do not skip it.

## Installation

### Option 1: CDN (simplest, no build step)

Add both the CSS and the JS bundle to your HTML:

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@opencage/geosearch-bundle/dist/css/autocomplete-theme-classic.min.css" />
<script src="https://cdn.jsdelivr.net/npm/@opencage/geosearch-bundle"></script>
```

### Option 2: npm

| Package | Use case |
|---------|----------|
| `@opencage/geosearch-bundle` | Bundled version including Algolia Autocomplete — easiest to use |
| `@opencage/geosearch-core` | Core plugin only; bring your own Algolia Autocomplete dependency |
| `@opencage/leaflet-opencage-geosearch` | Leaflet map integration |
| `@opencage/ol-opencage-geosearch` | OpenLayers map integration |

```bash
npm install @opencage/geosearch-bundle
```

## Basic Setup

You need:
1. A container element in your HTML (e.g. `<div id="place"></div>`)
2. A Geosearch API key (`oc_gs_...`)
3. The domain registered in your OpenCage dashboard (see CORS note above)

## Configuration Options

### Required Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `container` | string | CSS selector for the container element |
| `key` | string | Your Geosearch API key (`oc_gs_...`) |

### Optional Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `bounds` | string | — | Restrict results to a bounding box: `"min_lon,min_lat,max_lon,max_lat"` |
| `countrycode` | string | — | ISO 3166-1 Alpha-2 country code to restrict results (e.g. `'de'`, `'gb'`) |
| `proximity` | string | — | Bias results toward a coordinate: `"lat,lng"` |
| `language` | string | `'en'` | Two-letter language code for results: `de`, `en`, `es`, `fr`, `it`, `pt` |
| `limit` | number | `5` | Maximum number of results shown (max: `10`) |
| `debounce` | number | `300` | Milliseconds to wait after user stops typing before sending a request |
| `noResults` | string | `"No results."` | Message displayed when the API returns no matches |

## Event Handlers

Pass event handlers as the second argument to `OpenCageGeoSearchPlugin()`:

| Handler | When it fires |
|---------|---------------|
| `onSelect(params)` | User clicks or keyboards to a result |
| `onActive(params)` | A result becomes the active/highlighted item |
| `onSubmit(params)` | The form is submitted |

Each handler receives a `params` object containing `state`, `event`, `item`, and setter functions. The selected place's coordinates are at `params.item.geometry.lat` and `params.item.geometry.lng`.

## Code Examples

### Minimal example (CDN)

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@opencage/geosearch-bundle/dist/css/autocomplete-theme-classic.min.css" />
<script src="https://cdn.jsdelivr.net/npm/@opencage/geosearch-bundle"></script>

<div id="place"></div>

<script>
  opencage.algoliaAutocomplete({
    container: '#place',
    plugins: [
      opencage.OpenCageGeoSearchPlugin({
        key: 'oc_gs_YOUR-KEY-HERE'
      })
    ]
  });
</script>
```

### With country restriction and result handler

```js
opencage.algoliaAutocomplete({
  container: '#place',
  plugins: [
    opencage.OpenCageGeoSearchPlugin({
      key: 'oc_gs_YOUR-KEY-HERE',
      countrycode: 'de',
      language: 'de',
      limit: 3
    }, {
      onSelect: (params) => {
        const { lat, lng } = params.item.geometry;
        console.log('Selected coordinates:', lat, lng);
        // e.g. populate hidden form fields, pan a map, etc.
      }
    })
  ]
});
```

### Accessing coordinates on selection

The `params.item` object returned in `onSelect` contains:
- `params.item.geometry.lat` — latitude
- `params.item.geometry.lng` — longitude
- `params.item.formatted` — human-readable place name string

## Map Integration

The widget can be paired with mapping libraries. Dedicated packages and tutorials are available for:

- **Leaflet** — use `@opencage/leaflet-opencage-geosearch`
- **OpenLayers** — use `@opencage/ol-opencage-geosearch`
- **MapLibre** — use the core plugin with MapLibre GL JS

## Browser Support

Modern browsers are supported. For legacy browser support, include a `fetch` polyfill (e.g. `unfetch`) since the widget uses the Fetch API internally.

## CSS Customisation

The widget ships with Algolia's `autocomplete-theme-classic` stylesheet. Appearance can be customised by overriding the CSS custom properties (variables) defined in that theme.

## What the Service Covers

Geosearch returns results for: countries, states/provinces, regions, cities, towns, villages, suburbs/neighbourhoods, and some points of interest.

It does **not** return: individual street addresses, postcodes, or road names.

## Further Reading

- OpenCage Geosearch overview (feature list, live demo, getting started): https://opencagedata.com/geosearch
- OpenCage Geosearch GitHub (full README, changelog, and framework-specific examples): https://github.com/OpenCageData/geosearch/blob/master/README.md
- OpenCage Pricing (free trial limits, paid plan comparison): https://opencagedata.com/pricing
