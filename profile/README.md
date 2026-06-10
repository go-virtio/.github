# go-virtio

**Pure-Go, transport-agnostic virtio drivers.**

`go-virtio` is a family of Go modules that implement
[virtio](https://docs.oasis-open.org/virtio/virtio/v1.1/virtio-v1.1.html)
device-class drivers in pure Go — no cgo, no kernel — and route every
transport-level operation through a narrow interface so the same driver
code works under UEFI, bare-metal MMIO, virtio-mmio, and anything else
that can satisfy the contract.

The point: replace the fragmented in-tree virtio code that lives inside
each Go project with one reusable, well-tested, transport-pluggable set
of drivers. Extracted from the [cloud-boot](https://github.com/cloud-boot)
TamaGo + UEFI loader, but designed from day one to run anywhere virtio
runs — and now driving a real `virtio-gpu` device end-to-end (see
[Validation](#validation)).

## Repositories

| Repo | Latest | Role |
| --- | --- | --- |
| [`common`](https://github.com/go-virtio/common) | v0.1.5 | Transport-agnostic infrastructure: PCI capability walker, modern-config registers, split-virtqueue + **descriptor chaining**, device-class IDs (now also centralizing the virtio-fs IDs), the `Transport` interface. |
| [`net`](https://github.com/go-virtio/net) | v0.1.1 | virtio-net (DID 0x1041). Frame-level `TransmitFrame` / `ReceiveFrame` over a TX/RX queue pair. |
| [`rng`](https://github.com/go-virtio/rng) | v0.1.1 | virtio-rng (DID 0x1044). Single-queue entropy `Read`. The minimal device class. |
| [`vsock`](https://github.com/go-virtio/vsock) | v0.1.1 | virtio-vsock (DID 0x1053). Three queues; packet-level `SendPacket` / `ReceivePacket` with `virtio_vsock_hdr`. |
| [`blk`](https://github.com/go-virtio/blk) | v0.2.0 | virtio-blk (DID 0x1042). `ReadBlocks` / `WriteBlocks` / `Flush`; requests are header + data + status descriptor chains. |
| [`console`](https://github.com/go-virtio/console) | v0.1.0 | virtio-console (DID 0x1043). Raw byte-stream `Write` / `Read` over an rx/tx pair. |
| [`balloon`](https://github.com/go-virtio/balloon) | v0.1.0 | virtio-balloon (DID 0x1045). `Inflate` / `Deflate` via le32 PFN arrays. |
| [`gpu`](https://github.com/go-virtio/gpu) | v0.6.0 | virtio-gpu (DID 0x1050). **2D framebuffer** + **virgl 3D** (host-GPU `ClearScreen` / `DrawTriangle` / `DrawTexturedTriangle`) + a pure-Go **software 3D rasterizer** (`gpu/soft3d`). |
| [`fs`](https://github.com/go-virtio/fs) | v0.2.1 | virtio-fs (DID 0x105A). FUSE-over-virtio **read-write** mount: Init/Lookup/Open/Read + Write/Create/Mkdir/SetAttr/Unlink/Rename/Fsync/… (each `fuse.h`-cited). v0.2.1 fixes a `fuse_attr` struct-size bug found by real-virtiofsd validation. |
| [`venus`](https://github.com/go-virtio/venus) | v0.5.1 | **Vulkan-over-virtio** (Venus), full: a `vk.xml`→Go serializer/**generator** (offline byte-verified) **plus** a working shared-memory ring transport. A clear-image runs end-to-end on a real renderer — the guest submits the full Vulkan sequence (instance→device→image→clear→submit) over the ring to `virgl_test_server --venus` + lavapipe, the host creates the image and executes the clear, **and on a Linux render-node host the guest maps the cleared image's `HOST3D\|MAPPABLE` blob (gbm dma_buf) and reads the texels back as RED** — guest-side pixel readback included. |
| [`validate`](https://github.com/go-virtio/validate) | v0.1.0 | Multi-driver real-hardware validation harness (tamago+QEMU) + a pure-Go virglrenderer/Venus vtest client. Validates all 8 drivers + virgl 3D + the full Venus clear-image-with-readback (on macOS QEMU + Linux hosts). |

## How the pieces fit

```
  net   rng   vsock   blk   console   balloon   gpu (2D + virgl 3D)   fs   spec-level drivers
   └─────┴──────┴──────┴───────┴────────┴─────────┴──────┘
                            │
              ┌─────────────▼──────────────┐
              │       go-virtio/common      │                          transport-agnostic infra
              │  PCI cap walker · ModernConfig · split-virtqueue ·     │
              │  descriptor chaining · Transport interface             │
              └─────────────┬──────────────┘
                            │  Transport (PCIConfigReader,
                            │   BARMemoryAccessor, PageAllocator)
   ┌────────────────────────▼───────────────────────────────────────┐
   │  EFI_PCI_IO_PROTOCOL · bare-metal MMIO · virtio-mmio · TamaGo …  │  host backplane
   └─────────────────────────────────────────────────────────────────┘
```

Every driver in `go-virtio/*` consumes `common.Transport` and nothing
else. Swap the backplane, keep the driver.

## The 3D story

`go-virtio/gpu` makes the honest split explicit — "pure-Go 3D" is three
different things:

- **Software (CPU).** `gpu/soft3d` is a dependency-free, z-buffered
  triangle rasterizer that renders into the virtio-gpu framebuffer. Works
  on any host, no GPU. **Validated end-to-end** (it draws the cube below).
- **virgl (host GPU).** `gpu` hand-encodes the virgl command stream
  (shaders shipped as TGSI *text*) so a host `virglrenderer` does the
  drawing — real hardware acceleration, still CGO=0. **Clear, triangle and
  textured triangle are all validated against a real virglrenderer**
  (software llvmpipe, via the validate harness) — the textured one samples a
  2×2 texture into a smooth gradient across the primitive.
- **Vulkan / Venus.** `venus` is full — a `vk.xml`→Go
  serializer/generator (offline byte-verified) **plus** a working
  shared-memory ring transport, and a **clear-image runs end-to-end on a
  real renderer**: the guest submits the full Vulkan command sequence
  (instance→device→image→clear→submit) over the ring to
  `virgl_test_server --venus` + lavapipe, and the host creates the image
  and executes the clear. **Guest-side pixel readback is closed too**: on a
  Linux render-node host the guest maps the cleared image's
  `HOST3D|MAPPABLE` blob (gbm dma_buf) and reads the texels back as RED. A
  separate subproject, not an incremental step.

What is *not* feasible in pure Go: a full OpenGL/Vulkan API — that needs a
Mesa-class GL→TGSI stack. Everything below that line is buildable, and
much of it is built.

## Validation

`go-virtio/validate` boots a bare-metal [TamaGo](https://github.com/usbarmory/tamago)
guest under QEMU, binds a `common.Transport` to a real `virtio-gpu-pci`
device, runs `OpenVirtioGPU → SetupFramebuffer → soft3d.RenderCube →
Flush`, and a host `screendump` confirms a shaded 3D cube at 320×240. The
2D framebuffer + software rasterizer are proven on a real device model —
not just against a fake. (Bringing it up surfaced two bugs a fake can't:
the PCI cap-walk needs dword-granular config reads, and a board-init
omission — the legacy 8259 PIC must be masked after switching to the I/O
APIC.)

The harness's `vtest/` half then validated all three **virgl 3D** paths
against a real `virglrenderer` (software llvmpipe, no GPU): the clear
stream reads back red, `DrawTriangle` rasterizes, and
`DrawTexturedTriangle` samples a 2×2 texture into a smooth gradient. That
flushed out two more bugs only a real renderer shows — `BIND_SHADER` was
32 (= `SET_TESS_STATE`; fixed to 31), and the textured fragment shader's
texcoord input defaulted to flat `CONSTANT` interpolation until declared
`PERSPECTIVE`.

The harness has since broadened well past gpu, and now **all 8 drivers are
validated on real devices**. Six run on the macOS QEMU host — **gpu, blk,
console, net, rng, balloon** — with byte-level round-trips: blk write then
read-back, console echo, net frame both directions, rng live entropy, and
balloon used-ring inflate/deflate. The last two need facilities macOS QEMU
cannot provide, so they run on real Linux hosts (Debian arm64 VMs with
`/dev/vhost-vsock`, a DRM render node, and KVM): **vsock** round-trips a
real vsock packet between the guest driver and an `AF_VSOCK` host process
via `vhost-vsock-pci`, and **fs** does read+write against a real
`virtiofsd` over QEMU `vhost-user-fs` — the guest reads a file back
byte-for-byte and writes one reflected on the host share. On the 3D side,
the virgl paths **and** the full Venus clear-image-with-readback are
validated against a real `virglrenderer`/lavapipe via the `vtest/` half.

The running theme: real-renderer / real-device validation has caught **9
encoding bugs invisible to fake-device unit tests** — `SURFACE`=8,
`BIND_SHADER`=31, the TGSI `PERSPECTIVE` interpolation, `SUBMIT_CMD2`
framing, an `idleTimeout`, an object-id-0, the `VkDeviceCreateInfo`
`pEnabledFeatures` field, `fuse_attr` being 88 bytes not 92 (fs, found by
real virtiofsd), and virglrenderer 1.1.0 requiring `vkGetDeviceQueue2`
(Venus). Real hardware sees what a fake cannot.

## Project standards

- **Pure Go.** No cgo, no syscalls into a host driver, no kernel
  dependency.
- **100% test coverage** is the bar, met on `rng`, `vsock`, `blk`,
  `console`, `balloon`, `gpu`, `fs` and the `venus`/`validate` Go;
  `common` (~99%) and `net` (~99%) carry a few defensively-unreachable
  branches in code derived before the rule. Coverage is measured, not
  asserted.
- **BSD-3-Clause** on all source files.
- **Spec-traceable.** Every register, descriptor field, and feature bit
  carries a citation back to the Virtio 1.1 spec section it implements.
- **Multi-arch.** Pure-Go code with no architecture-specific assembly, so
  the same drivers work on amd64, arm64, riscv64, loongarch64.

## Landing page

Project landing page: <https://go-virtio.github.io>.
