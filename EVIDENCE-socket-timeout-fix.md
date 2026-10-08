# Socket Timeout Fix — Evidence Pack

## Bug Summary

Node.js `http.ClientRequest` emits a `'timeout'` event when the socket times out, but **does not automatically destroy the request**. Without an explicit handler, the timeout fires silently and the request hangs forever — creating a "zombie socket" that prevents retry logic from triggering.

## The Fix

In `tus-js-client/lib/node/NodeHttpStack.ts`, a single event handler was added:

```typescript
req.on('timeout', () => {
  req.destroy(new Error('socket timeout'))
})
```

A secondary fix in `tus-js-client/lib/upload.ts` aborts orphaned partial uploads before creating new ones on retry:

```typescript
if (this._parallelUploads != null) {
  for (const upload of this._parallelUploads) {
    await upload.abort()
  }
}
this._parallelUploads = []
```

## Test Environment

- **Test**: `npm run test:matrix` (Playwright, `EXTREME_DROP_OUT` scenario)
- **Scenario**: 25MB file upload through cyclic `geostationary_satellite` / `connection_loss` phases (120s per phase)
- **Socket timeout**: 60s (`SENDER_SOCKET_TIMEOUT=60`)
- **Services**: broker + file-watcher + data-manager + tusd + postgres + fakesat emulator
- **Patched version**: local `npm link` of `tus-js-client-stall-detection` with the fix applied
- **Unpatched version**: published `tus-js-client-stall-detection@5.0.0-alpha.13` from npm

## Results Summary

| Run | Version | Test File | Result | Duration | Socket Timeout Retries |
|-----|---------|-----------|--------|----------|------------------------|
| 1 | **Patched** | `matrix-25MB-ExtremeDropOut-1776236835561.dat` | **PASSED** | ~10 min | Multiple |
| 2 | **Unpatched** | `matrix-25MB-ExtremeDropOut-1776238595265.dat` | **STUCK** | 23+ min (killed) | 0 |
| 3 | **Patched** | `matrix-25MB-ExtremeDropOut-1776240584532.dat` | **PASSED** | 11m 54s | 80 |
| 4 | **Unpatched** | `matrix-25MB-ExtremeDropOut-1776241442598.dat` | **STUCK** | 23+ min (killed) | 0 |
| 5 | **Patched** | `matrix-25MB-ExtremeDropOut-1776242969784.dat` | **PASSED** | 11m 6s | 30 |

**Patched: 3/3 PASSED — Unpatched: 0/2 PASSED (both stuck indefinitely)**

## Detailed Evidence

### Patched Behaviour

Broker logs show the fix working as expected during `connection_loss` phases:

```
2026-04-15T08:18:53.120Z HEAD retry#1 socket timeout 35df37b3
2026-04-15T08:18:53.120Z HEAD retry#1 socket timeout 71ddff6e
2026-04-15T08:18:53.121Z HEAD retry#1 socket timeout cca22d08
2026-04-15T08:18:55.604Z HEAD retry#1 socket timeout e3f00bd3
2026-04-15T08:19:16.232Z HEAD retry#3 socket timeout e52d8673
2026-04-15T08:19:16.232Z HEAD retry#3 socket timeout 1617d11e
2026-04-15T08:19:16.233Z HEAD retry#3 socket timeout af18147e
2026-04-15T08:19:56.130Z HEAD retry#2 socket timeout 35df37b3
2026-04-15T08:19:58.608Z HEAD retry#2 socket timeout e3f00bd3
2026-04-15T08:20:49.981Z PATCH retry#0 socket timeout d9c8454f
2026-04-15T08:20:50.965Z PATCH retry#0 socket timeout 1617d11e
2026-04-15T08:20:55.848Z PATCH retry#0 socket timeout 71ddff6e
2026-04-15T08:21:02.112Z PATCH retry#0 socket timeout cca22d08
2026-04-15T08:21:45.645Z PATCH retry#0 socket timeout e3f00bd3
```

Key details from the request objects in broker logs:

- Error origin: `Error: socket timeout` at `NodeHttpStack.ts:122` — our new handler
- `"_eventsCount": 3` — response + error + **timeout** handler registered
- `"destroyed": true` — socket properly cleaned up by the timeout handler
- Retries cycle every 60s during `connection_loss`, keeping the upload alive
- Once `geostationary_satellite` returns, retries succeed and the transfer completes

### Unpatched Behaviour

Broker logs show the transfer going silent after hitting `connection_loss`:

```
2026-04-15T07:39:32.438Z PATCH retry#0 socket hang up 287c39e2   ← ECONNRESET from emulator
2026-04-15T07:41:49.699Z HEAD  retry#1 socket hang up 287c39e2   ← ECONNRESET from emulator
                                                                   ← SILENCE — no more retries
```

Key details from the request objects in broker logs:

- Error type: `socket hang up` / `ECONNRESET` — never `socket timeout`
- `"_eventsCount": 2` — only response + error handlers (no timeout handler)
- `"destroyed": false` — socket NOT cleaned up, zombie socket hangs forever
- Transfer stuck at `uploading` in the database — `updated_at` never changes
- Even after `geostationary_satellite` returns (network healthy), the transfer stays dead
- Only way to recover: restart the broker service

### Database Evidence

**Patched run** — transfer completes:

```
id                                   | status    | updated_at
-------------------------------------+-----------+----------------------------
e85351bd-7e09-47eb-8512-3bd3ec8dee87 | completed | 2026-04-15 08:21:52.044+00
```

**Unpatched run** — transfer stuck indefinitely:

```
id                                   | status    | updated_at
-------------------------------------+-----------+----------------------------
224c4a64-ca45-4c8d-adf0-5c971df8f4fd | uploading | 2026-04-15 08:24:17.413+00
```

(Checked at 08:48 — 24 minutes later, still `uploading`, `updated_at` unchanged, geostationary satellite active)

### `_eventsCount` Comparison

The single most telling detail — the number of event handlers registered on the `http.ClientRequest`:

| Version | `_eventsCount` | Handlers | Socket Destroyed on Timeout |
|---------|---------------|----------|-----------------------------|
| **Unpatched** | 2 | `response`, `error` | No — zombie socket |
| **Patched** | 3 | `response`, `error`, **`timeout`** | Yes — `req.destroy()` called |
