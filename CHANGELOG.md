# Changelog

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
