# BUG-001: DNS Monitor Update Endpoint Checks Wrong JobType

## Severity: MAJOR

## Description

The DNS monitor update endpoint (`PUT /v1/monitor/dns/{id}`) uses an incorrect job type check, making it **impossible to update any DNS monitor** via the API.

## Root Cause

In `apps/server/src/routes/v1/monitors/put_dns.ts` (line ~105), the code checks:

```typescript
if (_monitor.jobType !== "tcp") {
  throw new OpenStatusApiError({
    code: "NOT_FOUND",
    message: `Monitor ${id} not found`,
  });
}
```

This should be:
```typescript
if (_monitor.jobType !== "dns") {
```

The check was likely copy-pasted from `put_tcp.ts` and the job type string was never updated.

## Steps to Reproduce

1. Start the server in development mode
2. Create a DNS monitor:
```bash
curl -X POST http://localhost:3000/v1/monitor/dns \
  -H "x-openstatus-key: 1" \
  -H "Content-Type: application/json" \
  -d '{"name":"DNS Test","frequency":"10m","regions":["ams"],"request":{"uri":"openstatus.dev"}}'
# Returns 200 OK with id (e.g., 69), jobType: "dns"
```

3. Verify the DNS monitor exists:
```bash
curl http://localhost:3000/v1/monitor/69 -H "x-openstatus-key: 1"
# Returns 200 OK — monitor exists with jobType: "dns"
```

4. Try to update the DNS monitor:
```bash
curl -X PUT http://localhost:3000/v1/monitor/dns/69 \
  -H "x-openstatus-key: 1" \
  -H "Content-Type: application/json" \
  -d '{"name":"Updated DNS","frequency":"10m","regions":["ams"],"request":{"uri":"openstatus.dev"}}'
# Returns 404: {"code":"NOT_FOUND","message":"Monitor 69 not found"}
```

## Expected Behavior

`PUT /v1/monitor/dns/{id}` should return `200` and update the DNS monitor successfully.

## Actual Behavior

Returns `404 NOT_FOUND "Monitor {id} not found"` because the code checks `jobType !== "tcp"` (which is always `true` for DNS monitors since their jobType is `"dns"`).

## Impact

- **All DNS monitors are completely uneditable** via the v1 API
- Users must delete and recreate DNS monitors to change any setting
- TCP monitors could potentially be incorrectly updated through the DNS endpoint

## Suggested Fix

```diff
--- a/apps/server/src/routes/v1/monitors/put_dns.ts
+++ b/apps/server/src/routes/v1/monitors/put_dns.ts
-    if (_monitor.jobType !== "tcp") {
+    if (_monitor.jobType !== "dns") {
```
