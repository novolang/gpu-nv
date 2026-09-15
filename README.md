# gpu-nv

WebGPU is a graphics and compute API for the GPU, specified by the W3C in
[the WebGPU specification](https://www.w3.org/TR/webgpu/) with
[WGSL](https://www.w3.org/TR/WGSL/) as its shading language. This package is
that model in novo-lang, as values: the adapter, the device, the buffers, the
textures, the pipelines and the bind groups are descriptions a program builds
and checks with no device present, a frame is a list of commands, and the one
thing that talks to hardware is a trait a program supplies an implementation
of.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared with
its full signature, but every body is a `todo()` that panics when called. The
package is published so its design can be reviewed and depended on before it
is implemented. Version 0.1.0 will be the first working release.

## What it is

An **adapter** is one GPU the machine can see. A **device** is an open
connection to one, and it comes with a **queue**, which is what work is
submitted to. Every adapter has **limits**: the largest buffer, the largest
texture, how many bind groups a pipeline may have, and how large a compute
dispatch may be. Limits differ between machines, so a program reads them
rather than assuming them.

A **buffer** is a block of memory on the device. A **texture** is an image on
the device, and a **view** is a window onto part of one. A **sampler** says how
a shader reads a texture. A **shader module** is compiled WGSL source. A
**pipeline** is a shader plus everything fixed about how it runs.

A shader reads its inputs through **bindings**, numbered slots grouped into
**bind groups**. A **bind group layout** is the shape of one group: which slot
holds a uniform buffer, which holds a texture, which holds a sampler. A bind
group is that shape filled in with actual resources.

In this package none of those is the thing itself. Each is an **id** the driver
allocated, and the memory lives on the other side of the boundary. Every id is
its own type, so passing a texture where a buffer belongs is a type error
rather than a driver message about a handle number.

Work is recorded rather than performed. A **command list** is a flat list of
commands with a label: begin a pass, set a pipeline, set a bind group, draw,
dispatch, copy, end the pass. Recording it needs no device. Submitting it is
one call on the driver.

| Quantity | Value |
| --- | --- |
| Modules that perform anything | 1 of 4 |
| Members on the `GpuDriver` trait | 13 |
| Commands `GpuCommand` can hold | 15 kinds |
| Faults `gpucmd.validate` can report | 9 kinds |
| Copy row alignment required by WebGPU | 256 bytes |
| Smallest `maxTextureDimension2D` any backend must support | 8192 |
| Smallest `maxBindGroups` any backend must support | 4 |

## Install

```
novo pkg add gpu-nv
```

## Example

```novo
use gpucmd
use gpures

fn main() [io]
    // A view onto the texture this frame draws into. A driver answers
    // one from `create_texture`; the id stands in for it here.
    let target = GpuViewId { handle: 1 }

    // One frame, recorded with no device present: a pass that clears to
    // opaque black, and the command that ends it.
    let frame = gpucmd.extend(gpucmd.list("frame"),
                              gpucmd.render_pass(target, 0.0, 0.0, 0.0, 1.0, []))

    // Every reason a driver would refuse this list, found before it is
    // sent, against the limits every backend is required to support.
    for f in gpucmd.validate(frame, gpures.baseline_limits())
        println("this frame would be refused")

    // How many commands the frame holds.
    println("${gpucmd.length(frame)}")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test` fails
on purpose: every test reaches a `not implemented: gpu-nv.<module>.<fn>` panic.
The tests are the specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `gpures` | The adapter and its limits, the ten id types, the buffer, texture and sampler descriptions, the row and format arithmetic, the checks against an adapter's limits, and the fault type every call answers with. |
| `gpucmd` | The fifteen commands, the command list, the nine validation faults, the helpers that record a whole pass or a whole compute round trip, and the arithmetic for how many workgroups a dispatch needs. |
| `gpushader` | The shader source as a value, the binding layouts, the vertex layouts, the two pipeline descriptions, and the scans that read what a WGSL source declares so a layout can be checked against it. |
| `gpudriver` | The `GpuDriver` trait, the values its calls answer, and the functions that combine several of its calls into one: open the best device, validate then submit, submit then read back, release in the right order. |

## How to choose an entry point

**`gpucmd` alone is enough to build and check a frame.** Nothing in it needs a
device, so a test on a machine with no graphics hardware can assert exactly
what would have been submitted.

**`gpushader.layouts_agree` and `missing_bindings` check a shader against its
layout.** Both answer before a device exists, which is where a binding
mismatch is cheapest to find.

**`gpudriver.open_best` is the way in to real hardware.** It lists the
adapters, picks one and opens a device with its queue and its limits.

**`gpudriver.validate_and_submit` is the ordinary submit.** It checks the list
against this device's limits first, so a fault is named before the driver
refuses one.

**`gpudriver.submit_and_read` is a compute round trip in one call.** Submit,
then read one buffer back.

**`gpures.baseline_limits` is what to check against when no device is open.**
It answers the limits every backend is required to support, so a build-time
check has a floor to use.

## The rules a user needs

1. **A command list is decided before it is submitted.** An encoder in other
   APIs can read a query and choose what to draw next. A list cannot. The
   answer is two submissions with a readback between them, which costs a round
   trip. Occlusion queries, timestamps and indirect draws all need the thing a
   list cannot do.
2. **A pass is a range in the flat list, not a nested value.**
   `GpuBeginRenderPass` and `GpuEndPass` are two commands. An unbalanced list
   is therefore expressible, and `gpucmd.validate` is what makes it a named
   fault at a position.
3. **`gpucmd.validate` answers a list of faults, and an empty list means the
   submission is legal.** It catches a pass with no pipeline set, a command
   outside a pass, a pass left open, a dispatch past the adapter's workgroup
   limit, a copy with an unaligned row stride, and unbalanced labels.
4. **A copy's bytes per row must be a multiple of 256.** WebGPU's
   `COPY_BYTES_PER_ROW_ALIGNMENT` is 256, and it is the same on Vulkan, Metal
   and Direct3D 12. A readback with an unaligned stride *succeeds* and the
   image is sheared, one row further out of place per row, with nothing
   reported. `gpures.padded_row_bytes` is the stride to use and
   `GpuBadRowAlignment` is the fault.
5. **`storage` and `map_read` cannot be set on the same buffer.** A result a
   compute shader wrote cannot be read by the host without a copy into a second
   buffer. `gpures.usage_is_legal` is the check, `gpudriver.create_readback_pair`
   allocates the pair, and `gpucmd.compute_round_trip` records the copy.
6. **An unbound texture reads as transparent black.** A bind group with a slot
   unfilled draws nothing, and that looks exactly like a correct draw of
   nothing. `gpushader.missing_bindings` names the empty slots and
   `gpushader.layouts_agree` names the disagreements between a layout and what
   its shader declares.
7. **`n / workgroup_size` drops the last partial group.** The tail of the data
   is left untouched, with nothing reported, on exactly the inputs whose length
   is not a multiple of the size. `gpucmd.workgroups_for` is the ceiling
   division.
8. **A vertex buffer's stride must hold its attributes.** A stride that is
   short by one attribute shears the geometry rather than raising an error.
   `gpushader.stride_fits` is the check and `gpushader.required_stride` is the
   number.
9. **A software rasteriser runs everything, a thousand times slower.** A
   program that picked one by accident has a benchmark that means nothing.
   `gpures.is_software` is the question, and `gpures.preferred_adapter` refuses
   one unless the caller asks for it.
10. **Limits are per adapter and must be read.** A workgroup size, a bind group
    count, a buffer size and a texture dimension are all numbers a program
    silently exceeds on the smaller of two machines. `gpures.fits_limits` and
    `texture_fits_limits` make the check before the allocation rather than
    after the failure.
11. **A device can be lost, and the type says so.** Every driver call answers a
    `Result`, and `GpuDeviceLost` is a variant rather than a panic. A laptop
    that suspends, a driver that updates and a shader that hangs the GPU all
    produce it. `gpudriver.needs_recreate` says whether every resource has to
    be made again.
12. **Resources are released in an order the backend requires.**
    `gpudriver.release_all` applies it and `gpures` — through
    `gpudriver.release_rank` — is where the order is written down.
13. **The WGSL source is the caller's and this package does not compile it.**
    The driver compiles it and the diagnostic comes back whole.
    `gpushader.bindings_of_wgsl`, `has_entry_point` and `workgroup_size_of`
    read what a source *declares* and say so when they cannot.
14. **A pipeline names its entry points as strings.** A typo produces a missing
    function on some backends and a silently unbound stage on others.
    `gpudriver.create_checked_shader` checks the names before the driver does.
15. **`gpures`, `gpucmd` and `gpushader` perform nothing.** Every function in
    them declares no effects. `gpudriver` declares `[io]`, and `[io, time]` on
    the four calls that block until the device answers: opening a device,
    writing a buffer, submitting, and reading back.
16. **A driver implementation may declare more effects than the trait
    member.** An implementation over a C library declares `[io, time, ffi]`
    against an `[io, time]` member, which is legal because a trait with no
    effect parameter does not pin its implementations' effects.

## What is not included

- **The window, the surface and the swapchain.** This package renders to a
  texture. Presenting one belongs to a windowing package. An offscreen render
  is the whole of what a test, a thumbnailer and a headless renderer need, and
  it is the half with no platform in it.
- **A C binding.** A binding over a C library is a `sys` package, and this one
  may not depend on one, which is what keeps a C toolchain from arriving
  through a dependency. `GpuDriver` is how a binding is reached: a program
  takes both and supplies the implementation.
- **A WGSL front end.** The backend has one. A second would disagree with it
  in the corners, and the disagreement would appear as a shader this package
  accepts and the driver refuses.
- **Depth and stencil.** A two-dimensional renderer and a compute dispatch need
  neither.
- **Queries, timestamps, occlusion and indirect draws.** All four need a list
  that can branch on a result. See rule 1.
- **Linear algebra.** A transform matrix is sixteen floats a caller writes into
  a uniform buffer, and a driver has no business choosing which package
  produced them.
- **A device build.** There is no `tests/embedded_probe.nv` and no claim that
  any module runs on a microcontroller.

## Related packages

- `std.gpu` in the standard library is the runtime boundary: the foreign
  function declarations against the native WebGPU library, and the conditional
  link that keeps a program that does not use the GPU free of it. It stays
  where it is, because a package outside the `sys` layer may not declare
  foreign function effects. An implementation of `GpuDriver` over it is how a
  program uses both, and that implementation is about sixty one-line adapters.
- [color-nv](https://novo-lang.org/packages/color-nv) is colour as a value. A
  clear colour and a pixel read back are its `Srgb8`.
- [raster-nv](https://novo-lang.org/packages/raster-nv) draws shapes into a
  surface on the CPU. It is the other way to get a picture, and it needs no
  hardware.

## Tests

```bash
novo test tests/resource_tests.nv   # the descriptors, the limits, the two silent numbers
novo test tests/command_tests.nv    # a frame built and validated with no device
novo test tests/shader_tests.nv     # layouts against shaders, and the driver seam
```

The reference implementation is [wgpu](https://wgpu.rs/), whose API is the
shape this package follows. The normative document for the limits, the formats
and the copy alignment is the WebGPU specification, and its required limits
table is what `gpures.baseline_limits` answers. Asserting 256, 8192 and 4 is
therefore a check against that table rather than a restatement of this
package's own code.

Every assertion in `command_tests.nv` is about a recorded list rather than
about a device, which is what lets the render half of a program be tested on a
build machine. `shader_tests.nv` implements `GpuDriver` in about thirty lines
as a `NoDriver` that answers `GpuNoAdapter` to everything, which is itself an
assertion: a trait a test cannot implement in one screen is one a second
backend will not implement either.

The tests compile today and fail at run, each on the
`not implemented: gpu-nv.<module>.<fn>` panic that is its body. That is the
expected state of an interface release. They turn green one at a time as
bodies land. `novo test --isolate tests/<file>` prints one verdict per test.

## Implementation status

| Item | Implemented |
| --- | --- |
| `gpures.GpuBackend`, `.GpuAdapterKind`, `.GpuAdapterInfo`, `.GpuLimits`, `.GpuFault` | the types are declared; nothing constructs one |
| `gpures.GpuDeviceId`, `.GpuQueueId`, `.GpuBufferId`, `.GpuTextureId`, `.GpuViewId` | the types are declared |
| `gpures.GpuSamplerId`, `.GpuShaderId`, `.GpuPipelineId`, `.GpuLayoutId`, `.GpuBindGroupId` | the types are declared |
| `gpures.GpuBufferUsage`, `.GpuBufferDesc`, `.GpuTextureFormat`, `.GpuTextureDesc`, `.GpuSamplerDesc` | the types are declared |
| `gpures.no_usage`, `.storage_usage`, `.read_usage`, `.usage_is_legal` | no |
| `gpures.buffer`, `.texture_2d`, `.sampler_nearest_clamp` | no |
| `gpures.row_bytes`, `.padded_row_bytes`, `.format_bytes` | no |
| `gpures.fits_limits`, `.texture_fits_limits`, `.baseline_limits` | no |
| `gpures.is_software`, `.preferred_adapter`, `.backend_name` | no |
| `gpucmd.GpuCommand`, `.GpuCommandList`, `.GpuListFault`, `.GpuWorkCount` | the types are declared; nothing constructs one |
| `gpucmd.list`, `.push`, `.extend`, `.length`, `.uses_buffer`, `.work_count` | no |
| `gpucmd.validate` | no |
| `gpucmd.render_pass`, `.compute_pass`, `.compute_round_trip`, `.workgroups_for` | no |
| `gpushader.GpuShaderDesc`, `.GpuStages`, `.GpuBindingKind`, `.GpuBindingDesc`, `.GpuLayoutDesc` | the types are declared; nothing constructs one |
| `gpushader.GpuBindingValue`, `.GpuBindingSlot`, `.GpuBindGroupDesc`, `.GpuWorkgroupSize` | the types are declared |
| `gpushader.GpuVertexAttribute`, `.GpuVertexBufferDesc`, `.GpuRenderPipelineDesc`, `.GpuComputePipelineDesc` | the types are declared |
| `gpushader.shader`, `.stages`, `.layout`, `.sampler_texture_layout` | no |
| `gpushader.bindings_of_wgsl`, `.layouts_agree`, `.has_entry_point`, `.workgroup_size_of` | no |
| `gpushader.vertex_buffer`, `.attribute`, `.stride_fits`, `.required_stride` | no |
| `gpushader.render_pipeline`, `.compute_pipeline`, `.bind_group`, `.missing_bindings` | no |
| `gpudriver.GpuDriver`, and all thirteen of its members | the trait is declared; no implementation is shipped |
| `gpudriver.GpuOpenDevice`, `.GpuTextureView`, `.GpuSubmission`, `.GpuResource`, `.GpuReadbackPair` | the types are declared |
| `gpudriver.open_best`, `.validate_and_submit`, `.submit_and_read` | no |
| `gpudriver.create_checked_shader`, `.create_readback_pair` | no |
| `gpudriver.release_all`, `.release_rank`, `.needs_recreate` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
