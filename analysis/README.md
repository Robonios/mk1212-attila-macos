# Analysis

Static analysis of the Feral macOS build of Total War: ATTILA, used to work out what the six-slot limit actually controls. Nothing here was produced by running the game: it is disassembly and symbol data read out of the executable on disk.

The subject is Feral's ATTILA 1.6.1, build `480285.103778`, arm64, executable SHA-256 `13f5d523…`, Mach-O UUID `392D6F66-9183-329E-8A98-8C90FC828326`. Addresses are load-time virtual addresses with a `__TEXT` base of `0x100000000`, so a file offset is the address minus that base. The executable itself is not included here and is not redistributable.

| File | What it is |
|---|---|
| `excerpts.md` | The instructions the findings rest on, with the reasoning around them |
| `fn_site2_1034577ec.s` | The 20-byte function containing patch site 2 — dead code, nothing references it |
| `stub_104146c00.s` | The 3-instruction stub called from the constructor's landing pad: a jump to the imported `_Unwind_Resume` |
| `CampaignSettlementCallback_vtable_methods.txt` | Addresses and sizes of the distinct implementations in `EMPIRECAMPAIGN::CampaignSettlementCallback`'s vtables, as `address<TAB>size<TAB>slots`. No code. |

This is deliberately a small set. Earlier drafts included full disassembly of the settlement-panel function at `0x103456db8`, the widget-fill routine at `0x103457800`, the second `building_slot_%d` formatter at `0x10345d1c0`, and the 1168 matching C++ typeinfo names. None of that is needed to follow the argument, and reproducing that much of a commercial binary is not something this repository needs to do. `excerpts.md` quotes the roughly twenty instructions that carry the findings and describes the rest in prose.

## What it establishes

The six-slot limit is two instructions storing `6` into offset `+0x74` of an `EMPIRECAMPAIGN::CampaignSettlementCallback`. That field is the number of `building_slot_N` widgets the capital's settlement panel fills in and refreshes. It is read only as a loop bound; it never sizes memory, indexes an array, or reaches an allocator. The class's constructor defaults it to 4, and site 1 raises it to 6 for capitals.

Site 2 is unreferenced — no call, no branch, no address taken, no vtable entry — so writing to it changes no behaviour. That is what made it a useful control: a session patched only at site 2 ran with six slots and still degraded, which pointed at the act of writing to a signed code page rather than at the slot value.

The full reasoning is in the [investigation record](../README.md#ten-slot-settlements).

## Reproducing it

Everything here came from a read-only disassembly of the executable, for example:

```bash
objdump -d --start-address=0x103457238 --stop-address=0x103457250 "Total War ATTILA"
```

## Copyright

These excerpts are derived from a commercial, copyrighted executable and are published as a record of interoperability research into a specific interface limit. They are deliberately limited to the instructions the findings depend on. No part of the game is redistributed here.
