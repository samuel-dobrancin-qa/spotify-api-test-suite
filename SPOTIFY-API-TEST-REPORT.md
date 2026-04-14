# Spotify API — QA Test Suite Report

**Tester:** Samuel Dobrančin — Quality Engineer  
**Date:** April 2026  
**Target:** Spotify Web API (api.spotify.com)  
**Tool:** Postman  
**Categories:** Authentication · Search · Track Data  

---

## Why I tested this

Most API testing portfolios use practice APIs built specifically for learning — they're safe, predictable, and don't reflect what real production APIs actually do. I wanted to test something real.

Spotify's Web API is used by thousands of developers building music applications worldwide. It has authentication flows, pagination, market restrictions, and edge cases that only surface when you actually probe the boundaries. Testing it seriously means finding things that matter to real developers, not just confirming that happy paths work.

This collection covers three endpoint categories across 12 test cases — authentication, search, and track data. Every test was designed to probe a specific behaviour, not just verify the obvious.

---

## Environment setup

All tests run against a Postman environment with four variables:

- `client_id` and `client_secret` — Spotify developer app credentials
- `base_url` — `https://api.spotify.com/v1`
- `access_token` — populated automatically by AUTH-01 and reused across all subsequent requests

The token automation means the entire collection can be run in sequence without manual intervention. AUTH-01 fires first, saves the token, and every other request picks it up from the environment variable.

---

## Category 1 — Authentication

Spotify uses OAuth 2.0 Client Credentials flow for server-to-server API access. These four tests cover the happy path, two distinct failure modes, and invalid token behaviour against a protected endpoint.

---

### AUTH-01 — Valid credentials

**Endpoint:** `POST https://accounts.spotify.com/api/token`  
**Result:** 200 — 4/4 tests passed

Confirmed: valid credentials return an access token with `token_type: Bearer` and `expires_in: 3600`. Token saved automatically to environment for downstream use.

**Observation:** Tokens expire after exactly one hour. Any test collection running longer than 60 minutes needs to re-authenticate mid-run. Worth building a pre-request script to handle token refresh automatically in a production test suite.

---

### AUTH-02 — Invalid credentials

**Endpoint:** `POST https://accounts.spotify.com/api/token`  
**Result:** 400 — 3/3 tests passed

Invalid client ID and secret return `"error": "invalid_client"` with `"error_description": "Invalid client"`. The error code is machine-readable and the description is human-readable — two separate fields serving two different audiences. Good API design.

---

### AUTH-03 — Missing grant type

**Endpoint:** `POST https://accounts.spotify.com/api/token`  
**Result:** 400 — 3/3 tests passed

Removing the `grant_type` parameter entirely returns `"error": "unsupported_grant_type"` with `"error_description": "grant_type parameter is missing"`. 

**Notable finding:** Spotify returns a different error code for a missing required field (`unsupported_grant_type`) than for wrong credentials (`invalid_client`). The error description even names the missing parameter. This level of specificity means a developer can fix the problem without guessing — they know exactly what's wrong and where.

---

### AUTH-04 — Invalid token against protected endpoint

**Endpoint:** `GET https://api.spotify.com/v1/search` with `Authorization: Bearer fake_token_12345`  
**Result:** 401 — 4/4 tests passed

**Notable finding:** The accounts endpoint (`accounts.spotify.com`) and the API endpoint (`api.spotify.com`) return different error response structures. The accounts endpoint returns flat JSON:

```json
{
  "error": "invalid_client",
  "error_description": "Invalid client"
}
```

The API endpoint returns nested JSON:

```json
{
  "error": {
    "status": 401,
    "message": "Invalid access token"
  }
}
```

These are two different services with two different error schemas. A developer consuming both endpoints needs to handle two different error formats in their error handling code — something easy to miss if you only test one endpoint type.

---

## Category 2 — Search

The search endpoint is Spotify's most used API endpoint. These four tests cover the happy path, an empty query, URL encoding behaviour, and a genuinely surprising finding about Spotify's search algorithm.

---

### SEARCH-01 — Valid track search

**Endpoint:** `GET /search?q=Radiohead&type=track&limit=5`  
**Result:** 200 — 5/5 tests passed

Returned 5 tracks as requested. Pagination metadata present and correct — `total: 8`, `next` URL pointing to offset 5, `previous: null` on the first page.

**Observation:** Every track in the response had `"preview_url": null`. Spotify removed 30-second preview clips from most markets in 2023. The field still exists in the response schema and is still documented, but returns null consistently. A developer building a music preview feature would get no error and no warning — just silent null values across all results. This is a silent deprecation that could cause subtle bugs in applications that assume `preview_url` is populated.

**Automatic chaining:** The first track ID (`Let Down` by Radiohead) was saved to the `track_id` environment variable for use in the Track Data category.

---

### SEARCH-02 — Empty query

**Endpoint:** `GET /search?q=&type=track&limit=5`  
**Result:** 400 — 3/3 tests passed

Empty `q` parameter returns `"error": {"status": 400, "message": "No search query"}`. Specific, actionable error message. Correctly distinguished from a valid query that returns zero results — those should be 200, this is 400 because the request itself is malformed.

---

### SEARCH-03 — Unencoded special characters break URL structure

**Endpoint:** `GET /search?q=@#$%^&type=track&limit=5`  
**Result:** 400 — 4/4 tests passed

**Finding:** This test produced an unexpected result. The `&` character in the query string `@#$%^&type=track` was interpreted as a URL parameter separator rather than a literal character. Spotify's server received the request as `q=@#$%^` with the `type` parameter missing — returning `"error": "missing parameter type"` rather than anything related to special characters.

This is not a Spotify bug — it's a URL encoding requirement. Special characters in query parameters must be percent-encoded before sending (`@` → `%40`, `#` → `%23`, `&` → `%26`). The test name was updated to reflect what actually happened: unencoded special characters break URL structure before the API even processes them. Any developer building a search interface needs URL encoding on the client side.

---

### SEARCH-04 — Spotify returns results for gibberish queries

**Endpoint:** `GET /search?q=xzqjfkwpvbnmxzqjfkwp&type=track&limit=5`  
**Result:** 200 — 4/4 tests passed

**Finding:** A completely random 20-character string returned 841 matching tracks including Gummy Bear songs and Bluey theme music. Spotify's search algorithm is fuzzy enough that near-zero-match queries still return results.

This has a real implication for developers — you cannot assume an empty result set means "no match" because Spotify will almost always return something. If you're building a "no results found" state in your application, you can't rely on `total: 0` to trigger it. You may need to implement your own relevance threshold or use Spotify's search more carefully.

---

## Category 3 — Track Data

These four tests probe how Spotify handles track retrieval with valid IDs, invalid IDs, non-existent IDs, and market restrictions. The distinction between the last three is one of the most interesting findings in the collection.

---

### TRACK-01 — Valid track by ID

**Endpoint:** `GET /tracks/{{track_id}}`  
**Result:** 200 — 4/4 tests passed

Retrieved "Let Down" by Radiohead (OK Computer, 1997) using the ID saved automatically from SEARCH-01. Track data complete — name, duration, artist, album all present.

**Observation:** `preview_url` was completely absent from this response — not present as null, simply missing from the schema entirely. In the search results the same track had `"preview_url": null`. The same field behaves differently across two endpoints — present but null in search, absent entirely in track lookup. Inconsistent response structure for the same data type across related endpoints is worth flagging for any developer building applications that need to handle both.

---

### TRACK-02 — Invalid base62 ID

**Endpoint:** `GET /tracks/INVALID_TRACK_ID_12345`  
**Result:** 400 — 3/3 tests passed

Returns `"error": {"status": 400, "message": "Invalid base62 id"}`.

Spotify track IDs use base62 encoding — alphanumeric characters only, 22 characters in length. When the ID format is structurally wrong, Spotify identifies this as a format error before even checking the database. The error message names the encoding standard, which is directly useful to a developer debugging the issue.

---

### TRACK-03 — Well-formed but non-existent ID

**Endpoint:** `GET /tracks/0000000000000000000000`  
**Result:** 404 — 3/3 tests passed

Returns `"error": {"status": 404, "message": "Resource not found"}`.

**Key finding:** This is the important distinction from TRACK-02. A structurally invalid ID returns 400. A structurally valid ID that doesn't exist in the database returns 404. Spotify differentiates between two genuinely different problems — a malformed request versus a valid request for something that doesn't exist. A developer handling errors in their application needs to handle these separately: 400 means fix your code, 404 means the content is gone and you should handle that gracefully for the user.

---

### TRACK-04 — Market restriction

**Endpoint:** `GET /tracks/{{track_id}}?market=CN`  
**Result:** 200 — 5/5 tests passed

**Best finding in the collection.** Requesting a track with a market parameter for a region where it's unavailable returns 200 — not 403, not 404. The full track data is returned but two things change:

```json
{
  "is_playable": false,
  "restrictions": {
    "reason": "market"
  }
}
```

Spotify tells you the track exists, that it can't be played, and exactly why. This is deliberate API design — the track is in the catalogue but licensed only for certain territories.

A developer building a music application needs to actively check `is_playable` on every track response when market parameters are involved. Checking only for 200 and assuming all returned tracks are playable will produce a broken experience for users in restricted markets — the UI would show tracks that silently fail to play. The `restrictions.reason` field gives enough information to show a meaningful message to the user.

---

## Summary

| ID | Test | Expected | Actual | Result |
|---|---|---|---|---|
| AUTH-01 | Valid credentials | 200 + token | 200 + token | Pass |
| AUTH-02 | Invalid credentials | 400 | 400 invalid_client | Pass |
| AUTH-03 | Missing grant type | 400 | 400 unsupported_grant_type | Pass |
| AUTH-04 | Invalid token | 401 | 401 Invalid access token | Pass |
| SEARCH-01 | Valid search | 200 + results | 200 + 5 tracks | Pass |
| SEARCH-02 | Empty query | 400 | 400 No search query | Pass |
| SEARCH-03 | Special characters | 400 or 200 | 400 missing parameter | Pass |
| SEARCH-04 | Gibberish query | 200 + empty | 200 + 841 results | Pass |
| TRACK-01 | Valid track ID | 200 + data | 200 + full track | Pass |
| TRACK-02 | Invalid base62 ID | 400 | 400 Invalid base62 id | Pass |
| TRACK-03 | Non-existent ID | 404 | 404 Resource not found | Pass |
| TRACK-04 | Market restriction | 200 | 200 + is_playable false | Pass |

12/12 tests passing.

---

## Key findings for developers

**Error code specificity is strong.** Spotify consistently returns different error codes for different problems — `invalid_client` vs `unsupported_grant_type`, 400 vs 404 for different ID failure modes. This makes debugging straightforward.

**Two services, two error schemas.** The accounts endpoint and the API endpoint return errors in different JSON structures. Handle both separately in your error handling code.

**`preview_url` is silently deprecated.** The field exists in responses but returns null or is absent. Don't build features that depend on it without a fallback.

**Fuzzy search always returns something.** Don't rely on empty results to trigger "no results found" UI states. Spotify will return something for almost any query.

**Market restrictions return 200, not an error.** Always check `is_playable` when using market parameters. A 200 response does not mean the track can be played.

---

## About

Samuel Dobrančin — Quality Engineer  
4+ years enterprise QA experience, currently deepening API testing skills.

[GitHub](https://github.com/samuel-dobrancin-qa) ·
[LinkedIn](https://linkedin.com/in/samuel-dobrancin-8a203a273)
