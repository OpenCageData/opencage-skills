# OpenCage Geocoding API — Command Line

This skill covers command-line usage of the OpenCage Geocoding API via the `opencage` CLI tool.
For general, language-agnostic concepts (endpoint, parameters, response structure, error codes, confidence scores, annotations, rate limits, test keys), refer to **opencage-geocoding-api/SKILL.md** first.

## Installation

```bash
pip install opencage
```

Requires Python 3.8 or newer. The same `opencage` package provides both the Python library and the CLI tool. Source is on GitHub at https://github.com/opencagedata/python-opencage-geocoder.

## Basic Usage

Every command requires four arguments:

```bash
opencage <command> --api-key YOUR-API-KEY --input <input-file> --output <output-file>
```

- `<command>`: either `forward` (address → coordinates) or `reverse` (coordinates → address)
- `--api-key`: your OpenCage API key
- `--input`: path to the input CSV file
- `--output`: path where results will be written

Use the built-in help for a full option listing:

```bash
opencage forward --help
opencage reverse --help
```

## Forward Geocoding (Addresses → Coordinates)

Input file — one address per line:

```
Madrid, Spain
Milan, Italy
Berlin, Germany
München, Deutschland
Pappelallee 78/79, 10437 Berlin, Germany
```

Command:

```bash
opencage forward --api-key YOUR-API-KEY --input addresses.csv --output coordinates.csv
```

## Reverse Geocoding (Coordinates → Addresses)

Input file — one `latitude,longitude` pair per line:

```
44.8303087,-0.5761911
2.345,10.2423
9.781652,124.391118
65.21345,50.234556
```

Command:

```bash
opencage reverse --api-key YOUR-API-KEY --input coordinates.csv --output addresses.csv
```

## Optional Parameters

**Pass API parameters** (language, country code, etc.) via `--optional-api-params`:

```bash
opencage forward --api-key YOUR-API-KEY \
  --input paris.csv \
  --output paris-usa.out.csv \
  --optional-api-params 'countrycode=us,language=fr'
```

**Parallel processing** — up to 20 workers; paying customers only (free trial is 1 req/s):

```bash
opencage reverse --api-key YOUR-API-KEY \
  --input coordinates.csv \
  --output results.csv \
  --workers 4
```

**Skip header row** in input files:

```bash
opencage forward --api-key YOUR-API-KEY \
  --input addresses.csv \
  --output results.csv \
  --headers
```

**Specify input columns** (1-indexed) when the file has multiple columns and lat/lng are not in columns 1 and 2:

```bash
opencage reverse --api-key YOUR-API-KEY \
  --input data.csv \
  --output results.csv \
  --input-columns 2,3
```

**Select output columns** — control which fields appear in the output file:

```bash
opencage reverse --api-key YOUR-API-KEY \
  --input coordinates.csv \
  --output results.csv \
  --add-columns formatted,country,postcode
```

Default output columns include: `lat`, `lng`, `_type`, `_category`, `country_code`, `country`, `state`, `county`, `_normalized_city`, `postcode`, `road`, `house_number`, `confidence`, `formatted`.

## Debugging and Testing

**Dry run** — validate input format without making any API calls:

```bash
opencage reverse --api-key YOUR-API-KEY --input coordinates.csv --dry-run
```

**Limit rows** — process only the first N lines (useful for testing on a sample):

```bash
opencage forward --api-key YOUR-API-KEY --input addresses.csv --output results.csv --limit 50
```

**Verbose output** — print the full request URL and API response for each row:

```bash
opencage reverse --api-key YOUR-API-KEY --input coordinates.csv --output results.csv --verbose
```

## API Key via Environment Variable

Never hard-code your API key in scripts. Store it in an environment variable and pass it at runtime:

```bash
export OPENCAGE_API_KEY="YOUR-API-KEY"

opencage forward --api-key "$OPENCAGE_API_KEY" --input addresses.csv --output results.csv
```

Or add the export to your shell profile (`~/.bashrc`, `~/.zshrc`) so it persists across sessions.

## Further Reading

- OpenCage command-line tutorial: https://opencagedata.com/tutorials/geocode-commandline
- `opencage` package on PyPI: https://pypi.org/project/opencage/
- General API reference: **opencage-geocoding-api/SKILL.md**
