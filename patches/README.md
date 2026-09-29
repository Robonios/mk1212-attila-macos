# Patch

`mk1212-mac-fixes.patch` modifies [ausxen's MK1212 macOS Launcher](https://github.com/ausxen/MK1212_macOS), which is MIT-licensed. A copy of that licence sits beside this file as [`LICENSE`](LICENSE); it covers the upstream code the patch applies to, and the patch is offered under the same terms.

The patch is not a launcher and does nothing on its own. It applies on top of a checkout of the upstream project.

## Applying it

It is a plain `git diff` against upstream commit `c897ffe`.

```bash
git clone https://github.com/ausxen/MK1212_macOS
cd MK1212_macOS
git checkout c897ffe
git apply /path/to/mk1212-mac-fixes.patch
```

Later upstream commits may move the surrounding code, in which case the hunks need rebasing by hand.

## What it changes

| File | Change |
|---|---|
| `src/mk1212_mac_tool.py` | Lowercased shim filenames, isolation accepting a real `VFS/Local` directory, a widened running-game check, six Lua repairs, three-layer Lua validation before install, and the ten-slot runtime patch plumbing |
| `src/mk1212_slot_runtime_patch.c` | The injected library that rewrites the slot instruction in memory at load time |
| `src/mk1212_slot_runtime_patch_inject_only.c` | The control library: same guards, writes nothing |
| `scripts/build-runtime-patch` | Builds and ad-hoc signs either library |

Two of these changes are diagnostic rather than fixes. The Lua tracing in `mechanics_buffer_states.lua` and `mk1212_common.lua` exists to investigate buffer-state release, and `mk1212_slot_runtime_patch_inject_only.c` exists to isolate the cause of the input degradation. Both are described in the [investigation record](../README.md).

## The runtime patch

The ten-slot runtime patch in this diff is the cause of the input degradation documented in the investigation record, so it is recorded here as a finding rather than recommended for play. Six slots is the stable configuration.
