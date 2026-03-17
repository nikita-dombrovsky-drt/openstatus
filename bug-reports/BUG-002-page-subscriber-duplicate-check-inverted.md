# BUG-002: Page Subscriber Duplicate Check Has Inverted Logic

## Severity: MAJOR

## Description

The page subscriber endpoint (`POST /v1/page_subscriber/{id}/update`) has an inverted duplicate-detection query that causes **two critical failures**:

1. **Active subscribers are NOT detected as duplicates** → duplicate subscriptions can be created (caught only by a DB constraint, producing a 500 error with leaked SQL)
2. **Unsubscribed users are blocked from re-subscribing** → they receive a `409 CONFLICT` when trying to subscribe again

## Root Cause

In `apps/server/src/routes/v1/pageSubscribers/post.ts` (lines 65-73), the "already subscribed" check queries:

```typescript
const alreadySubscribed = await db
  .select()
  .from(pageSubscriber)
  .where(
    and(
      eq(pageSubscriber.email, input.email),
      eq(pageSubscriber.pageId, Number(id)),
      isNotNull(pageSubscriber.acceptedAt),    // ✅ correct
      isNotNull(pageSubscriber.unsubscribedAt), // ❌ WRONG — should be isNull()
    ),
  )
  .get();
```

The last condition `isNotNull(pageSubscriber.unsubscribedAt)` matches subscribers who **have been unsubscribed** (i.e., former subscribers). It should be `isNull(pageSubscriber.unsubscribedAt)` to match **currently active** subscribers.

## Steps to Reproduce

### Scenario A: Active subscriber can create duplicates (should be blocked)

1. Subscribe an email:
```bash
curl -X POST http://localhost:3000/v1/page_subscriber/1/update \
  -H "x-openstatus-key: 1" -H "Content-Type: application/json" \
  -d '{"email":"test@bugtest.com"}'
# Returns 200 OK
```

2. Verify the subscription in the database (simulate email verification):
```sql
UPDATE page_subscriber SET accepted_at = strftime('%s', 'now')
WHERE email = 'test@bugtest.com' AND page_id = 1;
```

3. Try subscribing the same email again:
```bash
curl -X POST http://localhost:3000/v1/page_subscriber/1/update \
  -H "x-openstatus-key: 1" -H "Content-Type: application/json" \
  -d '{"email":"test@bugtest.com"}'
# ACTUAL: Returns 500 with full SQL query leaked in response body
# EXPECTED: Returns 409 CONFLICT "Email already subscribed"
```

### Scenario B: Unsubscribed user blocked from re-subscribing (should be allowed)

1. Follow steps 1-2 from Scenario A
2. Unsubscribe the user:
```sql
UPDATE page_subscriber SET unsubscribed_at = strftime('%s', 'now')
WHERE email = 'test@bugtest.com' AND page_id = 1;
```

3. Try re-subscribing:
```bash
curl -X POST http://localhost:3000/v1/page_subscriber/1/update \
  -H "x-openstatus-key: 1" -H "Content-Type: application/json" \
  -d '{"email":"test@bugtest.com"}'
# ACTUAL: Returns 409 CONFLICT "Email test@bugtest.com already subscribed"
# EXPECTED: Returns 200 OK (user should be able to re-subscribe)
```

## Expected Behavior

- Active subscribers (acceptedAt set, unsubscribedAt NULL) should be detected and return `409 CONFLICT`
- Unsubscribed users (acceptedAt set, unsubscribedAt set) should be allowed to re-subscribe

## Actual Behavior

- Active subscribers are NOT detected → 500 error from DB unique constraint with **SQL query leaked** in response
- Unsubscribed users ARE detected (incorrectly) → blocked with 409 CONFLICT

## Security Concern

When an active subscriber tries to subscribe again, the DB unique index triggers a 500 error that **leaks the full SQL INSERT statement** in the response body, including table/column names and parameter values.

## Impact

- Users cannot re-subscribe after unsubscribing
- Duplicate subscription attempts produce 500 errors instead of clean 409 responses
- SQL query details are exposed in error responses (information disclosure)

## Suggested Fix

```diff
--- a/apps/server/src/routes/v1/pageSubscribers/post.ts
+++ b/apps/server/src/routes/v1/pageSubscribers/post.ts
     const alreadySubscribed = await db
       .select()
       .from(pageSubscriber)
       .where(
         and(
           eq(pageSubscriber.email, input.email),
           eq(pageSubscriber.pageId, Number(id)),
           isNotNull(pageSubscriber.acceptedAt),
-          isNotNull(pageSubscriber.unsubscribedAt),
+          isNull(pageSubscriber.unsubscribedAt),
         ),
       )
       .get();
```

Note: Also need to import `isNull` from `@openstatus/db` (currently only `isNotNull` is imported).
