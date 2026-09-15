# Changelog

## 0.0.2 — 2026-09-15

- README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

## 0.0.1 — interface

The interface, published before anything is implemented: every `pub fn`
body is a `todo()`, and the signatures, the effect rows and the tests are
the design.

- Four modules.  `gpures`, `gpucmd` and `gpushader` are `[]` throughout
  — descriptors, a recorded frame and a binding layout are values a
  caller builds and checks with no device present; `gpudriver` declares
  `[io]` and `[io, time]`, which is what `std.gpu` declares minus the
  `[ffi]` a `host` package may not have.
- 51 public functions and one trait with thirteen members, every body a
  `todo("gpu-nv.<module>.<fn>")`.
- Three test suites, 25 tests, red on purpose — and one of them
  implements `GpuDriver`, because a trait a test cannot implement in a
  screen is one a second backend will not implement either.
- `std.gpu` is neither wrapped nor replaced: it is the runtime boundary
  and stays there, and the README carries the table of what it keeps.

### Design notes

Two findings recorded here rather than in the README, which states what the
package does rather than how it was decided.

- **An effect-parameterised `trait GpuDriver[e]` would make `gpures`, `gpucmd`
  and `gpushader` reachable from a `core` package.** Every function in
  `gpudriver` is a pump over the trait and performs nothing of its own, so
  with the parameterised form each would be charged what the caller's
  implementation costs and this package would declare no effects at all. The
  concrete form is used instead, because it is what the standard library's own
  rows already look like.
- **A consumer that binds one device per process would take the values and
  leave the trait.** `GpuBufferId` and `GpuCommandList` fit a workload that
  holds opaque device blobs behind handles and runs a sequence of kernels, and
  `gpucmd.workgroups_for` and `GpuLimits.max_compute_workgroup_size_x` are the
  two numbers such a workload needs. Taking `GpuDriver` as a `dyn` would mean
  paying its effect union on every call, and kernels written in a language
  other than WGSL have nothing to gain from the shader surface.
