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