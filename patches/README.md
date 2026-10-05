# Patch

`mk1212-mac-fixes.patch` modifies [ausxen's MK1212 macOS Launcher](https://github.com/ausxen/MK1212_macOS), specifically **upstream commit `c897ffe`**, which carried an MIT licence. A copy of that licence sits beside this file as [`LICENSE`](LICENSE): it is upstream's `LICENSE` as it stood at `c897ffe`, it covers the upstream code this patch applies to, and the patch is offered under the same terms.

**Upstream has since removed its `LICENSE` file**, in commit `5386ac8` ("Version 0.8.0", 3 October 2026). As of that release the project carries no licence file and no licence statement in its README. The MIT terms recorded here therefore describe `c897ffe`, not upstream's current state — check with upstream before reusing anything from a later version.

The patch is not a launcher and does nothing on its own. It applies on top of a checkout of the upstream project.

## Applying it

It is a plain `git diff` against upstream commit `c897ffe`, and **it targets that commit specifically**.

```bash
git clone https://github.com/ausxen/MK1212_macOS
cd MK1212_macOS
git checkout c897ffe
git apply /path/to/mk1212-mac-fixes.patch
```

**It no longer applies to upstream's current `main`.** Upstream 0.8.0 rewrote the runtime-patch source and reworked the launcher, so `git apply --check` fails on two counts:

```
error: scripts/build-runtime-patch: already exists in working directory
error: patch failed: src/mk1212_mac_tool.py:33
error: src/mk1212_mac_tool.py: patch does not apply
```

`git apply --3way` gets further but leaves conflicts in both files. Checking out `c897ffe` first, as above, is the only clean route.

That is expected rather than a problem: 0.8.0 replaced the code-page write with a heap-field patch that writes no executable memory, which is the mechanism this investigation's evidence pointed towards. The ten-slot parts of this patch are superseded by it and are kept here as the record of how the conclusion was reached.

## What it changes

| File | Change |
|---|---|
| `src/mk1212_mac_tool.py` | Lowercased shim filenames, isolation accepting a real `VFS/Local` directory, a widened running-game check, four local Lua repairs, three-layer Lua validation before install, and the ten-slot runtime patch plumbing |
| `src/mk1212_slot_runtime_patch.c` | The injected library that rewrites the slot instruction in memory at load time |
| `src/mk1212_slot_runtime_patch_inject_only.c` | The control library: same guards, writes nothing |
| `scripts/build-runtime-patch` | Builds and ad-hoc signs either library |

Two of these changes are diagnostic rather than fixes. The Lua tracing in `mechanics_buffer_states.lua` and `mk1212_common.lua` exists to investigate buffer-state release, and `mk1212_slot_runtime_patch_inject_only.c` exists to isolate the cause of the input degradation. Both are described in the [investigation record](../README.md).

## The runtime patch

The ten-slot runtime patch in this diff is the cause of the input degradation documented in the investigation record, so it is recorded here as a finding rather than recommended for play. On this setup, six slots was the stable configuration.

Upstream 0.8.0 supersedes it. That release changes the same slot count without writing to a signed code page: it interposes `operator new`, identifies the settlement-callback object by its allocation size and both vtable pointers, and compare-exchanges the `+0x74` field from 6 to 10 on the heap. It also hashes the code page at startup and exits if anything modifies it. For ten slots, use upstream 0.8.0 rather than this patch.
