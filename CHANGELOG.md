# Changelog

## 0.1.0 — 2026-10-13

The implementation landed. Every `todo()` under `src/` is a body, the
three shipped suites and two additions to them run green, and the publish
gate reports 99% of 413 instrumented lines under `src/` covered over five
suites, with one region excused.

- 51 public functions and one trait with thirteen members, all
  implemented. `[dependencies]` is still empty: the absences are the
  design, and nothing landed here needed one.
- Five suites, 60 tests, no `the run leaked` line and no `W####`.
  `resource_tests`, `command_tests` and `shader_tests` are the shipped
  oracle; `gpudriver_edge_tests` and `gpu_nv_edge_tests` are additions,
  the second written against the coverage report and named for the lines
  it reaches.
- 53 doc examples, all compiled.
- One region is excused by `cov: skip`: the `at < 0` arm of
  `gpushader.type_name`, which is unreachable through the module's only
  caller — `declared_binding` answers `None` to a declaration with no
  `var` before it ever calls `kind_in`.
- Stability is `experimental`, not `stable`, and the one thing that
  talks to hardware is `gpudriver.GpuDriver`, whose evidence is a
  recording double. See `### Known`.

### Added

- Every body: the descriptor builders, the row arithmetic, the frame
  validator, the limit checks, the WGSL text scans and the trait pumps in
  `gpudriver`.
- `tests/gpu_nv_edge_tests.nv` — 27 tests, one per reachable region the
  shipped suites left uncovered: the fault `message` arms of every enum
  in the package, the limit axes, the dispatch-limit axis order, the
  label balance, `work_count`'s arms, the scan's refusals, the shader-
  declares-what-the-layout-does-not-bind direction, and the workgroup
  size reader. Two of them pin a known gap deliberately (below), and
  two more were written by the mutation battery: a row stride of zero is
  a fault, and a pass the frame never ends is named where it opened.
- `tests/gpudriver_edge_tests.nv` — 8 tests over a `Recorder`
  implementation of `GpuDriver` that records the calls it was given.

### Changed

- The three shipped suites were left as the oracle in every assertion.
  Two classes of sentence changed: the release-stamp sentences that said
  the suites were red on purpose, and the ten `x == false` sites the
  `redundant-bool` lint rewrites to `not x`. No assertion's meaning
  changed and the counts at `HEAD` were checked against the counts now.
- `novo.toml`: version to `0.1.0`, stability to `experimental`, and the
  `novo` floor from `>= 0.8.9` to `>= 0.20.2`, which is the toolchain
  every verdict below was measured with.
- `README.md`: the status banner, the "interface only" paragraphs and
  the 27-row implementation-status table are gone; the section on how to
  run the tests now says what the suites assert against.

### Known

- **A `Result`-returning trait method reached through `dyn` leaks the
  `Result` it returns.** Filed as
  `memory-perceus/a-trait-method-reached-through-dyn-leaks-the-result-it-returns`
  (open, 0.20.2). Every pump in `gpudriver` would be written as a
  `match` on a `dyn GpuDriver` call, and that match leaks; each is
  written instead as a one-line tail-returning static wrapper around the
  call, with the `Result` unpacked at a static caller that holds the
  concrete type. The workaround costs a wrapper per member and is
  commented at each one. It can be removed when the filing closes.
- **`min_bytes` on a buffer binding is carried and not compared.** A
  layout that asks for 64 bytes agrees with a shader whose uniform buffer
  is one float. The size a shader wants is in the struct its declaration
  names or in the array length inside it, and reading that is a parse of
  the language rather than a scan of it, so the scan answers 0 for every
  buffer and the comparison would be 0 against 0. The disagreement is
  left to the driver's own layout validation, which reports it with the
  numbers in it. Pinned by
  `test_a_binding_s_minimum_size_is_not_compared_and_that_is_a_known_gap`.
- **The scan reads a `fn` inside a comment as a declaration.** It is a
  text search and not a lexer: a source whose entry point is commented
  out still answers `has_entry_point`. The typo this catches is a typo in
  a real declaration, and a source a driver refuses it refuses in its own
  words. Pinned by
  `test_a_fn_inside_a_comment_is_read_as_a_declaration_and_that_is_a_known_gap`.
- **Nothing here has been run against a GPU.** This machine has none, so
  the seam's evidence is a double that records calls and a `NoDriver`
  that answers `GpuNoAdapter` to everything — which is what a build
  machine would answer. `layer = "host"` and the package claims no
  device, so there is no emulated target to run either; the pure half
  carries the suite.

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
