# Excerpts

The instructions the ten-slot findings actually rest on. Everything else is described in prose rather than reproduced.

Addresses are load-time virtual addresses with a `__TEXT` base of `0x100000000`; subtract that base for a file offset. Feral ATTILA 1.6.1, build `480285.103778`, arm64, executable SHA-256 `13f5d523…`.

## 1. The six-slot store (patch site 1)

At `0x103457244`, inside the 2432-byte function at `0x103456db8` that builds the settlement panel. This is the only one of the two sites that executes.

```
103457238: f9412ec0    ldr   x0, [x22, #0x258]
10345723c: 52800028    mov   w8, #0x1
103457240: 3901c008    strb  w8, [x0, #0x70]
103457244: 528000c8    mov   w8, #0x6          <-- the patched instruction
103457248: b9007408    str   w8, [x0, #0x74]
10345724c: aa1503e1    mov   x1, x21
```

The object is fetched from `+0x258` of whatever `x22` holds, a flag at `+0x70` is set to 1, and the slot count at `+0x74` is set to 6. The patch rewrites `0x103457244` to `mov w8, #0xa`, four bytes, `48 01 80 52`.

Immediately after this, the function calls the 3404-byte routine at `0x103457800` that fills in the slot widgets.

The same three-instruction idiom occurs exactly once more in the 100 MB binary, at `0x1034577f4`. That copy is in [`fn_site2_1034577ec.s`](fn_site2_1034577ec.s) — a 20-byte function nothing calls, branches to, or takes the address of, and which appears in no vtable. It is an out-of-line copy of a method that was also inlined at site 1. Writing to it changes no behaviour, which is what made it useful as a control.

## 2. First loop bound: the widget fill

In the function at `0x103457800`, which formats `building_slot_%d` names and populates the panel. `x20` holds the object.

```
103457f8c: b9407688    ldr   w8, [x20, #0x74]
103457f90: aa1503fc    mov   x28, x21
103457f94: eb0802bf    cmp   x21, x8
103457f98: 54000e82    b.hs  0x103458168      <-- exit loop when index >= count
103457f9c: f9400699    ldr   x25, [x20, #0x8]
```

`+0x74` is reloaded each iteration and compared against the loop index; the loop exits once the index reaches it. Thirty-two instructions earlier, at `0x103457f50`, the same field is loaded and a `cbz` skips the whole block when it is zero.

## 3. Second loop bound: the widget refresh

In the function at `0x10345b200`, which walks the same widgets and calls a method on each. `x19` holds the object.

```
10345b214: b9407408    ldr   w8, [x0, #0x74]
10345b218: 34000568    cbz   w8, 0x10345b2c4  <-- nothing to do when zero
10345b21c: aa0003f3    mov   x19, x0
10345b220: 52800016    mov   w22, #0x0        <-- index = 0
10345b224: f000a3b4    adrp  x20, 0x1048d2000 <-- "building_slot_%d"
10345b228: 912f9694    add   x20, x20, #0xbe5
10345b22c: 14000004    b     0x10345b23c
10345b230: b9407668    ldr   w8, [x19, #0x74]
10345b234: 6b0802df    cmp   w22, w8
10345b238: 54000462    b.hs  0x10345b2c4      <-- exit loop when index >= count
```

The same shape: guard on zero, then index against `+0x74` each time round.

In both loops `+0x74` is read only as a bound. It is never scaled into an allocation size, never used as an array index, and never reaches an allocator. Both loops index the widget list held at `+0x8` of the object, which the object does not own or size.

A third routine, at `0x10345d1c0`, also formats `building_slot_%d`, but it loops over the settlement's real slot list rather than over `+0x74`. So the interface limit and the game's actual slot count are separate quantities that can disagree.

## 4. The constructor, and the default of 4

`EMPIRECAMPAIGN::CampaignSettlementCallback`, 136 bytes at `0x1034aac8c`, also reachable as slot 56 of its own primary vtable.

```
1034aac9c: 52801400    mov   w0, #0xa0        <-- operator new(160)
1034aaca0: 943274b8    bl    0x104147f80
...
1034aacc4: f000bf69    adrp  x9, 0x104c99000  <-- primary vtable
1034aacc8: 91302129    add   x9, x9, #0xc08
1034aaccc: f9000269    str   x9, [x19]        <-- vptr
1034aacd0: 9107c129    add   x9, x9, #0x1f0   <-- secondary vtable
1034aacd4: a904a269    stp   x9, x8, [x19, #0x48]
1034aacd8: 3901c27f    strb  wzr, [x19, #0x70] <-- flag = 0
1034aacdc: d0008468    adrp  x8, 0x104538000
1034aace0: fd454900    ldr   d0, [x8, #0xa90]
1034aace4: fc074260    stur  d0, [x19, #0x74]  <-- slot count = 4, and +0x78 = 0
1034aace8: 7901327f    strh  wzr, [x19, #0x98]
```

The eight bytes at `0x104538a90` are `04 00 00 00 00 00 00 00`, so the single `stur` writes **4** to `+0x74` and 0 to `+0x78`.

This is what identifies the class. Two vptrs are stored, the second at `+0x48` with an `offset_to_top` of −72, so the object is polymorphic with multiple inheritance. The primary vtable base is `0x104c99bf8` and the secondary `0x104c99de8`; the values stored are those plus 16, as the ABI requires. The allocation size of 160 bytes, the flag at `+0x70`, the count at `+0x74`, and the halfword at `+0x98` all match the fields the two loops above use.

The constructor defaults the count to **4**, and site 1 raises it to **6** — matching vanilla ATTILA, where minor settlements show four building slots and provincial capitals six. The patch changes only the capital figure.

The `bl 0x104146c00` at the tail of this function is the exception landing pad calling `_Unwind_Resume`; that stub is in [`stub_104146c00.s`](stub_104146c00.s).

## What follows from this

`+0x74` is the number of `building_slot_N` widgets the capital's settlement panel fills in and refreshes. It is a loop bound over interface widgets and nothing else — which is why the ten-slot patch has no memory cost, never appears near a crash, and could not plausibly cause the input degradation by way of the value it writes.

None of the 17 distinct implementations in the class's vtables reads `+0x70`–`+0x77`, including through the secondary-base view. Every reader is non-virtual. The vtable addresses and sizes are in [`CampaignSettlementCallback_vtable_methods.txt`](CampaignSettlementCallback_vtable_methods.txt).
