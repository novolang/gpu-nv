# gpu-nv

`std.gpu` as a package: the device, the buffers and the pipelines as
values, a frame as a list a host submits, and the driver as a trait.

**Status: NOT IMPLEMENTED — interface only.**  Every `pub fn` body is a
`todo()`, so the signatures, the effect rows and the tests are published
and nothing is implemented.  The first implementation is the `0.1.0`
published over this.

## What this is

The standard library binds wgpu-native through about sixty `@ffi`
externs, and a program using them holds raw handle integers, calls
`novo_gpu_encoder_begin`, mutates an encoder and submits it.  That
surface works — the acceptance probes render offscreen and read pixels
back — and it cannot be tested, cannot be sent between cells, and cannot
be implemented by anything but wgpu-native.

This package is the same model as values, with one trait at the
boundary.

| surface | module | reach for it when |
| --- | --- | --- |
| the **resources** | `gpures` | you are allocating anything |
| the **frame** | `gpucmd` | you are recording work |
| the **shader** | `gpushader` | you are writing WGSL or a layout |
| the **boundary** | `gpudriver` | you are actually talking to a GPU |

51 public functions, one trait with thirteen members, every body a
`todo()`.

## The load-bearing interface

```novo norun:pseudo
pub enum GpuCommand
    GpuBeginRenderPass(target: GpuViewId, r: Float, g: Float, b: Float, a: Float)
    GpuSetPipeline(pipeline: GpuPipelineId)
    GpuDraw(vertex_count: Int, instance_count: Int)
    GpuDispatch(x: Int, y: Int, z: Int)
    // … eleven more

pub struct GpuCommandList
    label: Str
    commands: [GpuCommand]
```

**Recording and submitting are different acts.**  Every graphics API
models a frame as an encoder a caller mutates — begin a pass, set a
pipeline, draw, end, finish, submit — and every one of those calls
performs something.  Here the whole recording is a value: built with no
device present, checked with no device present, and handed to a driver
in one call.

Four things follow, and none of them is available to an encoder:

- **A frame can be built on one cell and submitted on another**, because
  a list is data and an encoder is a handle.
- **A test can assert what would have been submitted**, on a machine
  with no GPU.  `tests/command_tests.nv` is the whole of a renderer's
  correctness checked on a build machine, which is the difference
  between a render path with tests and one without.
- **`gpucmd.validate` is a function**, so a pass with no pipeline set, a
  dispatch past the adapter's workgroup limit and a texture copy with an
  unaligned row stride are named faults at a position in a list, rather
  than driver errors at run time with a handle number in the message.
- **The driver trait shrinks to `submit`**, which is what makes two very
  different backends plausible implementations of it.

**The cost is that a list cannot branch on a result.**  An encoder can
read an occlusion query and decide what to draw next; a list is decided
before it is submitted.  That is real, it is what GPU-driven rendering
needs, and the answer here is two submissions with a readback between
them — which costs a round trip.  Named rather than discovered.

## The one example that will work

```novo
use gpucmd
use gpures

// A frame, recorded with no device in the room and checked against the
// limits every backend must meet.
fn clear_frame(target: GpuViewId) -> Int
    let l = gpucmd.extend(gpucmd.list("frame"),
                          gpucmd.render_pass(target, 0.0, 0.0, 0.0, 1.0, []))
    list.len(gpucmd.validate(l, gpures.baseline_limits()))

fn main() [io]
    println("a frame, checked before anything is sent")
```

## Adding it, and checking it

```console
$ novo pkg add gpu-nv
$ novo pkg build
$ novo test --isolate tests/command_tests.nv
```

The suites are **red on purpose**: every body is a `todo()`, so every
assertion reaches `not implemented: gpu-nv.<module>.<fn>`.  Twenty-five
tests across three suites, all red, every failure that message.

`tests/shader_tests.nv` also **implements `GpuDriver`** — a `NoDriver`
that answers `GpuNoAdapter` to everything — and that is an assertion in
itself: a trait a test cannot implement in one screen is one a second
backend will not implement either.

## The layer, and the effect row

`host`, from the plan.  `gpures`, `gpucmd` and `gpushader` are `[]`
throughout; `gpudriver` declares `[io]`, and `[io, time]` where the call
blocks until the device answers.

**Those are the rows `std.gpu` declares, minus `[ffi]`.**  The standard
library's externs read `[io, ffi]` and `[io, time, ffi]` — the `[time]`
on exactly the calls that block: opening a device, writing a buffer,
submitting, reading back.  `[ffi]` is `sys`-only under the layer rules,
and that is what the trait is for: the C library stays on the shelf
behind an impl, and a `sys` impl declaring `[io, time, ffi]` against an
`[io, time]` member is legal, because a trait with no effect parameter
does not pin its impls' rows.

**The plan row said `[hw]`/`[gpu]`.**  `gpu` is not one of SPEC § 5.1's
eleven labels, so it cannot be an effect row at all — the same finding
pty-nv made about `proc`.  And `[hw]` is a claim the standard library
does not make about these same calls; a hardware abstraction declares
it, and this is a driver boundary over one.  The rows here are the ones
the runtime already declares, which is the answer that cannot be wrong
about what the code does.

## What `std.gpu` keeps, and what this takes

`std.gpu` **stays exactly where it is.**  It is the runtime boundary —
the `@ffi` shims in `bin/novo_rt_gpu.c` against `libwgpu_native.so`, the
conditional link that keeps a hello-world free of wgpu, and the
interpreter's `failwith` arm — and none of that can move into a package,
because a package may not declare `[ffi]` outside `sys`.  The two
coexist: a program takes both, and an impl of `GpuDriver` over
`std.gpu`'s externs is how they meet.  That impl is about sixty
one-line adapters and belongs beside the runtime rather than here.

| `std.gpu` keeps | this package takes |
| --- | --- |
| the `@ffi` externs and the `[ffi]` row | nothing: it never names a C library |
| the conditional link on `novo_gpu_` | nothing |
| the process-singleton `WGPUInstance` | nothing |
| raw `Int` handles | `GpuBufferId`, `GpuTextureId`, … as distinct types |
| `Adapter`, `Device`, `GpuBuffer`, `Texture`, `Pipeline`, `Encoder`, `RenderPass` as thin structs over handles | the same model, as descriptors that can be checked with no device |
| `novo_gpu_encoder_begin` … `finish_submit` — a mutated encoder | `GpuCommandList`, a value |
| three prebuilt pipeline shapes (`quad`, `quad_mvp`, `instanced_quad`) | `GpuRenderPipelineDesc` with an explicit vertex layout, so a fourth shape is a value and not a new extern |
| `ptr.alloc_word` out-parameters for the layouts a pipeline creates | `GpuOpenDevice` and `GpuTextureView`, which answer the pair |
| the 256-byte row alignment, in a comment | `gpures.padded_row_bytes`, and `GpuBadRowAlignment` |

**Window, surface and present stay out of both.**  `std.gpu`'s own
documentation already draws that line and this package keeps it: this
renders to a texture, and presenting one is `std.window`'s. An
offscreen render is the whole of what a test, a thumbnailer and a
headless renderer need, and it is the half with no platform in it.

## What ml's device dispatch would take, and what it would not

`orbit/ml` selects a backend with **a runtime device tag rather than a
trait**, and its own header says why: one process can hold handles from
two vendors at once, and a per-type impl would still have to carry the
tag.  That is a real disagreement with this package's shape, and it is
worth being exact about where it lands.

**What ml would take**: the values.  `GpuBufferId` and `GpuCommandList`
are exactly what its quantized weights and its matvec dispatch already
are — opaque device blobs behind handles, and a sequence of kernels to
run — and a recorded list is what lets it build a layer's work on the
CPU and submit it once.  `gpucmd.workgroups_for` is the ceiling division
its kernels need, and `gpures.GpuLimits.max_compute_workgroup_size_x` is
the wavefront-width hazard its README describes, read from the adapter
instead of derived from a build script.

**What it would not take**: `GpuDriver` as a `dyn`.  ml binds one device
per process and dispatches on a tag; taking the trait would mean either
one impl per vendor chosen at startup — which is what it already does,
spelled differently — or a `dyn` whose effect union every ml call pays.
Its CUDA and ROCm kernels are also not WGSL, and a WGSL-only shader
surface has nothing to say about them.

So the honest answer is: **ml would take the value half and leave the
trait.**  That is a report for the milestone review rather than a
redesign here, and it is the one consumer whose shape this package does
not fit.

## Where the design wanted to widen

**`trait GpuDriver[e]` would make two thirds of this package `core`.**
Every function in `gpudriver` is a pump over a trait: `open_best`,
`validate_and_submit`, `submit_and_read` and the rest perform nothing of
their own, and with the effect-parameterised form — btree-nv's `PageIo`
shape, `docs/publishing.md` § How a `core` package takes a stream from
its host — each would be charged what the caller's impl costs and this
package would have no effects at all.

It is not done here because the plan row says `host` and because the
concrete form is what the standard library's rows already look like.  It
is the same widening postgres-nv and mysql-nv named on `std.sql`'s
`Database`: three rows now want an effect parameter on a trait whose
impls differ in what they cost.

## Five places the GPU bites, and where each one has a name

- **A row stride that is not a multiple of 256.**  The readback
  SUCCEEDS and the image is sheared, one row further out of place per
  row, with no error attached.  `gpures.padded_row_bytes`, and
  `GpuBadRowAlignment` as a validation fault.
- **`Storage` and `MapRead` are mutually exclusive.**  A result a
  compute shader wrote cannot be read by the host without a copy step,
  and a program that tried gets a driver error about a buffer rather
  than about the rule.  `gpures.usage_is_legal`,
  `gpucmd.compute_round_trip`, `gpudriver.create_readback_pair`.
- **An unbound texture reads as transparent black.**  So a bind group
  with a slot unfilled draws nothing and looks exactly like a correct
  draw of nothing.  `gpushader.missing_bindings`, and
  `gpushader.layouts_agree` for the layout that disagrees with its
  shader.
- **`n / workgroup_size` drops the last partial group.**  The tail of
  the data is untouched, with no error, on exactly the inputs whose
  length is not a multiple of the size.  `gpucmd.workgroups_for`.
- **A vertex stride that does not hold its attributes.**  Wrong by four
  in every renderer that has added an attribute and forgotten it, and
  the symptom is geometry that shears rather than an error.
  `gpushader.stride_fits`.

And a sixth that is not a bug but a lie: **a software rasteriser runs
everything**, a thousand times slower, and a program that silently used
one has a benchmark that means nothing and a user who thinks their
machine is broken.  `gpures.is_software`, and `preferred_adapter`
refuses one unless asked.

## What is deliberately absent

- **The window, the surface and the swapchain.**  `std.window`'s, and
  the line `std.gpu` already draws.
- **A C binding.**  `libvulkan-sys` and the rest are `sys` and a `host`
  package may not depend on one — which is the rule that keeps a C
  toolchain from arriving through a dependency.  The trait is how one is
  reached.
- **Depth and stencil.**  A 2D renderer and a compute dispatch need
  neither, and a depth attachment is a row for the version that has a
  3D consumer.
- **Queries, timestamps and occlusion.**  All three need a list that can
  branch on a result, which is the trade-off named above.
- **Indirect draws.**  The same reason.
- **A WGSL front end.**  The backend has one; a second would disagree
  with it in the corners, and the disagreement would show up as a shader
  this package accepts and the driver refuses.  The scans here read what
  a source DECLARES and say so when they cannot.

## Reference

wgpu is the reference implementation and its API is the shape this
ports; WebGPU's specification is the normative document for the limits,
the formats and the 256-byte copy alignment, and its "required limits"
table is what `gpures.baseline_limits` answers.  `std.gpu` and
`bin/novo_rt_gpu.c` are the existing binding this package is measured
against, and `docs/stdlib/gpu.md` is the surface it keeps compatible in
behaviour if not in spelling.

The implementation lane's gate is a real adapter: an offscreen render
with pixel-asserted readback, a compute round trip, and the same two on
a software rasteriser so CI can run them.

## Licence

Apache-2.0.
