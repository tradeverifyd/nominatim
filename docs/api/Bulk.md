## `/bulk` Search Endpoint

The `/bulk` endpoint allows you to submit multiple geocoding queries in a single HTTP request. It accepts both free-form text queries and structured queries, and processes them efficiently by reusing a single database connection and query analyzer.

### HTTP Method

**POST** only. The request body must be JSON.

### Request Format

```json
{
  "queries": [ ... ],
  "limit": 1,
  "addressdetails": false,
  "countrycodes": "us,de",
  "accept-language": "en"
}
```

#### Parameters

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `queries` | array | yes | — | List of queries (strings or structured objects). Max default: 1000. |
| `limit` | integer | no | `1` | Results per query, clamped to 1–50. |
| `addressdetails` | boolean | no | `false` | Include broken-out address fields in results. |
| `countrycodes` | string | no | — | Comma-separated country codes to restrict results. |
| `accept-language` | string | no | server default | Preferred language for display names. |

### Query Types

Each entry in the `queries` array can be either a **free-form string** or a **structured object**.

#### Free-form string

A plain text query, equivalent to a normal `/search?q=...` call:

```json
"1600 Pennsylvania Ave NW, Washington, DC"
```

#### Structured query object

A dictionary with one or more of these fields:

| Field | Description | Example |
|---|---|---|
| `street` | House number and street name | `"1600 Pennsylvania Ave NW"` |
| `city` | City or town name | `"Washington"` |
| `county` | County name | `"Arlington County"` |
| `state` | State or province | `"DC"` |
| `postalcode` | Postal / ZIP code | `"20500"` |
| `country` | Country name or code | `"US"` |

At least one field must be provided. The most specific field present determines what rank of results are returned:

- `street` present → address-level results (ranks 26–30)
- `city` (no street) → city-level results (ranks 13–25)
- `county` (no city/street) → county-level results (ranks 10–12)
- `state` (no county/city/street) → state-level results (ranks 5–9)
- `postalcode` only → postal code results (ranks 5–11)
- `country` only → country-level results (rank 4)

### Example Request

```bash
curl -X POST http://localhost:8088/bulk \
  -H "Content-Type: application/json" \
  -d '{
    "queries": [
      "Eiffel Tower, Paris",
      {"street": "1600 Pennsylvania Ave NW", "city": "Washington", "state": "DC", "country": "US"},
      {"city": "Berlin", "country": "Germany"},
      {"postalcode": "10115", "country": "DE"}
    ],
    "limit": 3,
    "addressdetails": true
  }'
```

### Response Format

Returns a JSON array with one entry per query, in the same order as the input:

```json
[
  {
    "query": "Eiffel Tower, Paris",
    "results": [
      {
        "place_id": 12345,
        "osm_type": "W",
        "osm_id": 5013364,
        "display_name": "Tour Eiffel, Avenue Anatole France, Paris, France",
        "lat": "48.8583701",
        "lon": "2.2944813",
        "boundingbox": ["48.8574...", "48.8593...", "2.2933...", "2.2956..."],
        "address": {
          "tourism": "Tour Eiffel",
          "road": "Avenue Anatole France",
          "city": "Paris",
          "country": "France"
        },
        "importance": 0.78
      }
    ]
  },
  {
    "query": {"street": "1600 Pennsylvania Ave NW", "city": "Washington", "state": "DC", "country": "US"},
    "results": [ "..." ]
  }
]
```

Each result object contains:

| Field | Type | Description |
|---|---|---|
| `place_id` | integer | Internal Nominatim place ID |
| `osm_type` | string | OSM object type: `N` (node), `W` (way), or `R` (relation) |
| `osm_id` | integer | OpenStreetMap object ID |
| `display_name` | string | Full formatted address |
| `lat` / `lon` | string | Centroid coordinates |
| `boundingbox` | array | `[minlat, maxlat, minlon, maxlon]` |
| `address` | object | Address components (only when `addressdetails: true`) |
| `importance` | float | Relevance/importance score |

### Error Handling

- **Missing/empty `queries`**: returns an error stating `queries` must be a non-empty list
- **Too many queries**: returns an error if the list exceeds the server's `BULK_MAX_QUERIES` limit (default 1000)
- **Invalid structured query**: returns an error if a dict has none of the valid keys
- **Timeout**: if the server times out mid-processing, completed queries return their results and remaining queries return empty `results` arrays

### Tips

- **Mix query types freely** — you can combine free-form strings and structured objects in the same request.
- **Use structured queries for precision** — they apply automatic rank restrictions so you get results at the right geographic level.
- **Keep `limit` low** for batch workloads — the default of `1` is ideal for geocoding workflows where you only need the best match.
- **Use `countrycodes`** when you know the target country to speed up queries and reduce false matches.
