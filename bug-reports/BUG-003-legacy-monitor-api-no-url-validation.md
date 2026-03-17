# BUG-003: Legacy Monitor API Accepts Empty and Invalid URLs

## Severity: MEDIUM

## Description

The legacy monitor creation endpoint (`POST /v1/monitor`) does not validate the URL format, allowing creation of monitors with:
- Empty strings (`""`)
- Whitespace-only strings (`"   "`)
- Non-HTTP protocol URLs (`"javascript:alert(1)"`)

The newer v2 endpoint (`POST /v1/monitor/http`) properly validates URLs using `z.url()`.

## Root Cause

In `apps/server/src/routes/v1/monitors/schema.ts`, the `MonitorSchema` defines the URL field as:

```typescript
url: z.string().openapi({
  example: "https://www.documenso.co",
  description: "The url to monitor",
}),
```

This uses `z.string()` which accepts any string value, including empty strings and non-URL values.

In contrast, the v2 `HTTPMonitorSchema` uses:
```typescript
url: z.url().openapi({...})
```

which properly validates URL format.

## Steps to Reproduce

1. Create monitor with empty URL:
```bash
curl -X POST http://localhost:3000/v1/monitor \
  -H "x-openstatus-key: 1" -H "Content-Type: application/json" \
  -d '{"periodicity":"10m","url":"","name":"Empty URL","regions":["ams"],"method":"GET"}'
# Returns 200 OK with url: ""
```

2. Create monitor with javascript: protocol:
```bash
curl -X POST http://localhost:3000/v1/monitor \
  -H "x-openstatus-key: 1" -H "Content-Type: application/json" \
  -d '{"periodicity":"10m","url":"javascript:alert(1)","name":"XSS URL","regions":["ams"],"method":"GET"}'
# Returns 200 OK with url: "javascript:alert(1)"
```

3. Compare with v2 endpoint:
```bash
curl -X POST http://localhost:3000/v1/monitor/http \
  -H "x-openstatus-key: 1" -H "Content-Type: application/json" \
  -d '{"name":"Invalid URL","frequency":"10m","regions":["ams"],"request":{"url":"not-a-url","method":"GET"}}'
# Returns 400 BAD_REQUEST: "Invalid URL"
```

## Expected Behavior

The legacy endpoint should reject empty and invalid URLs with a `400 BAD_REQUEST` response.

## Actual Behavior

The legacy endpoint accepts any string as a URL, including empty strings and non-HTTP protocol URLs, returning `200 OK`.

## Impact

- Monitors with invalid URLs will fail silently when the checker tries to execute them
- Empty URL monitors waste resources (scheduler processes them but they can never succeed)
- `javascript:` protocol URLs could pose an XSS risk if the URL is rendered in the dashboard without sanitization
- Inconsistent validation between v1 (legacy) and v2 endpoints confuses API consumers

## Suggested Fix

Add URL validation to the legacy `MonitorSchema`:

```diff
--- a/apps/server/src/routes/v1/monitors/schema.ts
+++ b/apps/server/src/routes/v1/monitors/schema.ts
     url: z.string().openapi({
+    url: z.string().min(1).url().openapi({
       example: "https://www.documenso.co",
       description: "The url to monitor",
     }),
```

Or at minimum, add a `.min(1)` to prevent empty strings:
```diff
-    url: z.string().openapi({
+    url: z.string().min(1).openapi({
```
