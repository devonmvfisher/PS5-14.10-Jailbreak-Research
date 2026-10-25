# P4 Device Run Order — addrof via lineDash read window

**For NeonStride. DO NOT POST ON ISSUE #18 OR ANY PUBLIC ISSUE.**

## What it does

Turns the getLineDash read primitive into a full addrof (get the raw heap address of any JS object). This is the Stage 1.5 → Stage 2 bridge — the single piece blocking arbitrary R/W.

**Method:** Overwrites the lineDash Vector's internal m_buffer pointer to aim at a JSFinalObject's inline property area where we've placed the target object. getLineDash() reads back the raw JSValue bits as a float64, whose IEEE 754 bit pattern IS the 48-bit heap address (JSC NaN-boxing: object cells have 0x0000 in the top 16 bits).

## Prerequisites

- V720 same-type alias working (ctx_a=stale, ctx_b=reclaimed) — DEVICE CONFIRMED
- read64 and write64 primitives
- State base address from a prior step
- lineDash offset: **0x0a8** (16 bytes: m_buffer ptr + size/cap) — SOURCE CONFIRMED by cobrajet302gt from WK-1400 drop

## How to run it

Load P4-addrof-linedash.js after buildScribbleRW returns. Three steps:

```js
// Step 1: Bootstrap — confirm the alias and load the probe
P4_Addrof.addrof(targetObj, ctx_a, ctx_b, read64, write64, 0x0a8);

// Step 2: Scan header sizes to find inline property offset
// (pass the State base address you got from your alias setup)
P4_Addrof.scanHeaderSize(ctx_a, read64, write64,
    {hi: STATE_HI, lo: STATE_LO},  // State base from your reclaim
    0x0a8,                           // lineDash offset
    HOLDER_HI, HOLDER_LO);          // holder address from your leak

// Step 3: Extract with the header size that showed OBJECT POINTER
var result = P4_Addrof.extract(ctx_a, read64, write64,
    {hi: STATE_HI, lo: STATE_LO},
    0x0a8,
    HOLDER_HI, HOLDER_LO,
    32);  // or whichever header_size the scan flagged
```

**Getting the holder address:** The probe creates a `holder = {}` object internally. You need its heap address from your existing leak. Options:
1. If your read64 can scan the heap near State, the holder will be in a nearby libpas page (same general heap — Q116r2 confirmed)
2. If you have a spray-based address prediction, use that
3. If you have addrof for other object types already, use it to get holder's address

## What to report back

1. **scanHeaderSize output** — which header_size value(s) showed `<-- OBJECT POINTER`
2. **extract result** — the `target_addr` field and whether `match` is true (cross-check: direct read64 vs lineDash read should be identical)
3. **Any crash** — if getLineDash crashes, note whether it crashes on the read or on restore. The probe restores the original m_buffer after reading, so a crash on restore means the original was already corrupted
4. **Console log** — full `result.log` array

## What success means

If `result.match === true` and `result.target_addr` is a valid pointer (top 16 bits = 0x0000, non-zero):

**We have addrof.** Combined with the dash forge read primitive and FontProxy dispatch (Q114r), this completes the path to arbitrary R/W. The chain becomes:

1. addrof (P4) → get address of any JS object
2. Dash forge → read at any address
3. write64 → write at any address
4. FontProxy vtable dispatch → code execution
5. JIT-ROP → syscall access
6. Kernel exploit (procdesc UAF)

## What failure means

- **All header sizes show non-pointer values:** JSFinalObject layout differs from stock JSC. Report the full scan — we'll derive the right offset from the bit patterns
- **match is false (direct read vs lineDash mismatch):** Endianness or alignment issue. Report both values
- **getLineDash returns 0 elements:** size/capacity write didn't land. Check if the State base address is correct
- **Crash on getLineDash:** m_buffer redirect didn't land cleanly. The lineDash offset (0x0a8) may need device verification — run P5 first if unsure

## Sizing note (Kickazzman correction, issue #43)

The spray target is the **context object at 768B**, not the State at 272B. State rides inline (Vector<State, 1>). This doesn't change P4's operation (it works on State's lineDash field directly) but matters if you're doing heap layout setup before running this.

## Files

- `probes/P4-addrof-linedash.js` — the probe (already in this repo)
- `docs/Q116r2_ALLOCATOR_ARCHITECTURE.md` — allocator routing (libpas, same general heap)
- `docs/Q114r_FONTPROXY_VTABLE_REGISTRATION.md` — what addrof unlocks (the dispatch chain)
