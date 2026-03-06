# Hummingbird Bug Report

---

## Ticket 1: Missing Default for APP_PORT

**File:** `server.js`, line 35

### Bug

```javascript
const port = process.env.APP_PORT;
```

The port comes entirely from the `APP_PORT` environment variable. There is no fallback or guard — no default value, no validation.

If `APP_PORT` is not set, `process.env.APP_PORT` is `undefined`, and `app.listen(undefined)` tells Node.js to bind to a random available port (port 0 behavior). The server would start successfully but on an unpredictable port — `curl` requests would fail to connect unless you happened to guess the right port.

### Code Diff

**Before (broken):**
```javascript
const port = process.env.APP_PORT;
```

**After (fixed):**
```javascript
const port = process.env.APP_PORT || 9000;
```

### Why It Was Broken

Without a default or validation, if `APP_PORT` is missing from the environment, the server silently binds to a random port instead of failing with a clear error. This makes the issue very hard to debug since the server appears to start normally with no crash or error message.

---

## Ticket 2: Width Missing from Metadata Response

**File:** `clients/dynamodb.js`, line 78–84

### Bug

When calling `GET /v1/media/:id`, the response never includes a `width` field — even though `width` was correctly saved to DynamoDB during upload.

### Code Diff

**Before (broken):**
```javascript
return {
  mediaId,
  size: Number(Item.size.N),
  name: Item.name.S,
  mimetype: Item.mimetype.S,
  status: Item.status.S,
};
```

**After (fixed):**
```javascript
return {
  mediaId,
  size: Number(Item.size.N),
  name: Item.name.S,
  mimetype: Item.mimetype.S,
  status: Item.status.S,
  width: Number(Item.width.N),
};
```

### Why It Was Broken

`createMedia` correctly saves `width` to DynamoDB (line 34: `width: { N: String(width) }`). However, `getMedia` reads the item back but forgot to include `width` in the returned object. As a result, `GET /v1/media/:id` always returned metadata without the `width` field regardless of what value was passed in `?width=`.

---

## Ticket 3: Location Header Missing `http://`

**File:** `controllers/media.js`, line 111

### Bug

When the media is not yet ready and the API returns a `202 Accepted`, the `Location` header is missing the `http://` scheme prefix:

```
Location: hummingbird-production-alb-xxx.us-west-2.elb.amazonaws.com/v1/media/<id>/status
```

Clients cannot follow this as a redirect because it is not a valid absolute URI.

### Code Diff

**Before (broken):**
```javascript
res.set('Location', `${req.hostname}/v1/media/${mediaId}/status`);
```

**After (fixed):**
```javascript
res.set('Location', `${req.protocol}://${req.get('host')}/v1/media/${mediaId}/status`);
```

### Why It Was Broken

Two problems with the original code:

1. **Missing scheme** — `req.hostname` only returns the hostname (e.g. `hummingbird-alb-xxx.elb.amazonaws.com`) with no `http://` prefix, so the resulting `Location` header was not a valid absolute URI and clients could not follow it.
2. **`req.hostname` drops the port** — `req.hostname` strips the port number, so non-standard ports (e.g. `:9000`) would be lost. `req.get('host')` preserves both the hostname and port (e.g. `host:9000`), producing a correct absolute URI.

---

## Bonus: Status Never Changes (Two Bugs)

### Bug 1 — DynamoDB Key Casing Mismatch in `setMediaStatus`

**File:** `clients/dynamodb.js`, line 153

#### Bug

`setMediaStatus` was using `SK: { S: 'metadata' }` (lowercase) while both `createMedia` and `getMedia` used `SK: { S: 'METADATA' }` (uppercase). DynamoDB keys are case-sensitive, so `setMediaStatus` was targeting a non-existent item and silently creating a new record instead of updating the existing one.

#### Code Diff

**Before (broken):**
```javascript
Key: {
  PK: { S: `MEDIA#${mediaId}` },
  SK: { S: 'metadata' },
},
```

**After (fixed):**
```javascript
Key: {
  PK: { S: `MEDIA#${mediaId}` },
  SK: { S: 'METADATA' },
},
```

#### Why It Was Broken

DynamoDB `UpdateItem` with a non-existent key and no `ConditionExpression` silently creates a new item. So every call to `setMediaStatus` was writing to a phantom record (`SK: 'metadata'`) instead of the real one (`SK: 'METADATA'`), leaving the actual media record's status unchanged and stuck at `PENDING` forever.

---

### Bug 2 — Event Type String Mismatch in Worker

**File:** `worker/processor.js`, line 82
**Related:** `core/constants.js`, line 18

#### Bug

Event type string mismatch — the worker was checking for `'media.v1.resize'` but the API was publishing `'media.v1.resize-media'`.

#### Code Diff

**Before (broken):**
```javascript
if (parsed.type === 'media.v1.resize') {
```

**After (fixed):**
```javascript
if (parsed.type === 'media.v1.resize-media') {
```

#### Why It Was Broken

The correct event type `'media.v1.resize-media'` is defined in `core/constants.js` line 18 as `EVENTS.RESIZE_MEDIA.type`. The API uses this constant when publishing to SNS, so every message sent had `type: 'media.v1.resize-media'`.

However, the worker hardcoded a different string `'media.v1.resize'` (missing the `-media` suffix). Since no `if` branch matched, the worker hit the fallback `Skipping unknown event type` warning and discarded every resize message without processing it — leaving all media records stuck in `PENDING` status forever.
