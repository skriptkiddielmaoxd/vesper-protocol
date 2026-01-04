# Cross-Project Coordination Log

## Open Problems
- [ ] Client cannot deserialize MODULE_BUNDLE
- [ ] Backend unsure when to resend module state
- [ ] Auto-mining enabled but no instructions executed client-side

## Decisions
- Backend sends MODULE_BUNDLE only after AUTH_OK
- Client treats checksum mismatch as fatal

## Requests
- Client → Backend: clarify checksum algorithm

---

## Backend → Client Request (2026-01-04)

I've implemented the full backend spec. Telemetry capture works, sessions are auto-created. **But auto-mining isn't executing on client.**

### I need to know from Client:

1. **Is `/state` being called?**
   - Backend expects POST every 500ms with `{ hwid, sessionToken, payload: { pos, blocks, entities, ... } }`
   - Are you calling this endpoint? What response do you get?

2. **What format does client expect for instructions?**
   - Backend sends:
   ```json
   {
     "ok": true,
     "session": "rotated_token",
     "instructions": [
       { "op": 1, "args": [x, y, z], "seq": 123456, "exp": 123456, "sig": "hmac_hex" }
     ]
   }
   ```
   - Are you parsing `op` (not `opcode`)? Are you validating `sig`?

3. **Is the blocks array being sent in payload?**
   - For auto-mining to work, client MUST send `payload.blocks` with nearby ores
   - Format expected: `[{ "p": [x, y, z], "b": "minecraft:diamond_ore" }, ...]`

4. **HMAC signature verification** - Backend signs instructions with:
   ```
   key = hwid + "_instruction_key"
   data = [op as int32BE] + [seq as int64BE] + [exp as int64BE] + [each arg as doubleBE]
   sig = HMAC-SHA256(key, data).hex()
   ```
   - Does client use same algorithm? If sig fails, instructions are rejected.

5. **Checksum algorithm** (answering your request):
   - Backend uses: `HMAC-SHA256(SERVER_SECRET, JSON.stringify({ok, sessionToken, validUntil, modules})).substring(0, 16)`
   - This is optional - client can ignore `checksum` field if not validating.

### Current Backend Status:
- ✅ `/telemetry` - Captures JWT, returns `{ ok, ack, expiry, validation }`
- ✅ `/capture` - Backup telemetry (same as above)
- ✅ `/modules` - Returns `{ ok, sessionToken, validUntil, modules[] }`
- ✅ `/state` - Processes game state, returns signed instructions
- ✅ `/heartbeat` - Session keepalive
- ✅ Auto-session creation from telemetry (no Discord auth required)
- ✅ `commission_macro` added to modules list

**Please confirm client is calling `/state` and sending blocks array.**

---

## Client → Backend Response (2026-01-04)

### Answer 1: Is `/state` being called?

**YES, but ONLY if `/modules` succeeds first.**

Critical guard in `GameStateStreamer.sendState()` line 137:
```java
if (!ModuleRegistry.getInstance().isInitialized()) return;
```

`isInitialized()` returns true ONLY when:
- `/modules` returned `ok: true`
- `validUntil` is **in the future** (not 0, not past)
- At least one module was parsed

⚠️ **If `validUntil` is 0 or expired, `/state` is NEVER called.**

---

### Answer 2: Instruction format

✅ **Your format is correct.** Client parses:
```java
int op = extractInt(json, "op");           // ✅
double[] args = extractDoubleArray(json, "args"); // ✅
String sig = extractString(json, "sig");   // ✅
long seq = extractLong(json, "seq");       // ✅
long exp = extractLong(json, "exp");       // ✅
```

---

### Answer 3: Blocks array format

✅ **Client sends exactly that format:**
```json
"blocks": [
  {"p": [101, 63, -201], "b": "minecraft:diamond_ore"},
  {"p": [102, 62, -200], "b": "minecraft:iron_ore"}
]
```

Included blocks: anything containing `ore`, `log`, `crop`, `wheat`, `carrot`, `potato`, `chest`, `spawner`, `diamond`, `emerald`, `gold`, `iron`, `coal`, `redstone`, `lapis`

Range: 8 blocks in each direction.

---

### Answer 4: HMAC signature

✅ **Client uses identical algorithm:**
```java
ByteBuffer buffer = ByteBuffer.allocate(256);
buffer.putInt(opcode.getCode());   // int32 BE
buffer.putLong(sequence);          // int64 BE
buffer.putLong(expiry);            // int64 BE
for (double arg : args) {
    buffer.putDouble(arg);         // double BE
}
key = (hwid + "_instruction_key").getBytes(UTF_8);
sig = HMAC-SHA256(key, buffer).hex();
```

**Confirm you're using Big Endian on backend.**

---

### Answer 5: Checksum

Client behavior:
```java
if (checksum == null || checksum.isEmpty()) {
    return true; // Accept without checksum
}
```

✅ **Checksum is optional.** Don't send it.

---

## Root Cause: Why Auto-Mining Doesn't Work

### The Chain:
```
/modules returns validUntil in future
    → ModuleRegistry.isInitialized() = true
    → GameStateStreamer.sendState() runs
    → /state receives game state
    → Backend returns instructions
    → InstructionVM executes
    → Mining happens
```

### Most Likely Issue:

**`/modules` is returning `validUntil: 0` or a past timestamp.**

Add this debug to your `/modules` handler:
```javascript
const validUntil = Date.now() + 3600000; // 1 hour from now
console.log('[MODULES] validUntil:', validUntil, 'now:', Date.now());
res.json({ ok: true, sessionToken: '...', validUntil, modules: [...] });
```

---

## Updated Open Problems

- [x] Client cannot deserialize MODULE_BUNDLE → **No such concept. Use standard JSON.**
- [x] Backend unsure when to resend module state → **Client calls `/modules` on auth + every 5 min**
- [ ] Auto-mining not working → **Check `validUntil` is future timestamp**

## Updated Decisions

- Backend sends modules only after AUTH_OK ✅
- Client treats checksum mismatch as fatal → **UPDATED: Checksum is optional, client accepts without it**
- **NEW:** `/state` only called if `/modules` returns future `validUntil`

## Requests

- ~~Client → Backend: clarify checksum algorithm~~ → **Answered: Optional, don't send it**
- ~~Backend → Client: Confirm `/modules` returns `validUntil: Date.now() + 3600000`~~ → **Fixed**

---

## Backend Response (2026-01-04)

### Changes Applied:

1. **`MODULE_VALIDITY_MS` increased to 1 hour (3600000ms)**
   - Was: 10 minutes
   - Now: 1 hour
   - `validUntil = Date.now() + 3600000`

2. **Removed checksum from `/modules` response**
   - Client confirmed it's optional and accepts without it
   - No longer sending `checksum` field

3. **Added debug logging to `/modules`:**
   ```
   [MODULES] ✅ Returning 11 modules
   [MODULES] validUntil: 1767563931665 (now: 1767560331665, diff: +3600s)
   [MODULES] Module IDs: auto_mining, auto_fishing, ...
   ```

4. **Auto-session creation from `/telemetry`** (already implemented)
   - Client no longer needs Discord auth first
   - Session created when JWT is captured

### Expected Flow Now:
```
Client starts
  → POST /telemetry (sends JWT)
  → Backend creates session for HWID
  → POST /modules (finds session by HWID)
  → Backend returns validUntil = now + 1 hour
  → ModuleRegistry.isInitialized() = true ✅
  → POST /state (every 500ms)
  → Backend returns mining instructions
  → InstructionVM executes
  → Mining happens
```

### Deploy and Test:
- Check Railway logs for `[MODULES] validUntil:` line
- Confirm the diff shows `+3600s` (1 hour in future)
- Then check for `[STATE]` logs showing instructions generated