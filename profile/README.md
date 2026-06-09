# go-virtio

**Pure-Go, transport-agnostic virtio drivers.**

`go-virtio` is a small family of Go modules that implement
[virtio](https://docs.oasis-open.org/virtio/virtio/v1.1/virtio-v1.1.html)
device-class drivers in pure Go — no cgo, no kernel — and route every
transport-level operation through a narrow interface so the same driver
code works under UEFI, bare-metal MMIO, virtio-mmio, and anything else
that can satisfy the contract.

The point: replace the fragmented in-tree virtio code that lives inside
each Go project with one reusable, well-tested, transport-pluggable set
of drivers. Extracted from the [cloud-boot](https://github.com/cloud-boot)
TamaGo + UEFI loader, but designed from day one to run anywhere virtio
runs.

## Repositories

| Repo                                                          | Role                                                                                                                |
| ------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| [`common`](https://github.com/go-virtio/common)               | Transport-agnostic infrastructure: PCI capability walker, modern-config layout, split-virtqueue, transport interfaces. |
| [`net`](https://github.com/go-virtio/net)                     | Pure-Go virtio-net driver wrapping `common`. Implements the modern init sequence and split-virtqueue TX/RX path.   |
| [`blk`](https://github.com/go-virtio/blk)                     | Pure-Go virtio-blk driver. Placeholder today; activates when a concrete pre-EBS caller needs it.                    |

## How the pieces fit

```
  ┌─────────────────────────────────────────────────────────────┐
  │  go-virtio/net          go-virtio/blk        (future: gpu)  │   spec-level drivers
  └──────────────┬─────────────────┬──────────────────────────────┘
                 │                 │
  ┌──────────────▼─────────────────▼──────────────────────────────┐
  │                       go-virtio/common                         │   transport-agnostic infra
  │   PCI cap walker · ModernConfig · split-virtqueue · iface     │
  └──────────────┬─────────────────────────────────────────────────┘
                 │  Transport interface (PCIConfigReader,
                 │   BARMemoryAccessor, PageAllocator)
  ┌──────────────▼─────────────────────────────────────────────────┐
  │   EFI_PCI_IO_PROTOCOL · bare-metal MMIO · virtio-mmio · …     │   host backplane
  └────────────────────────────────────────────────────────────────┘
```

Every driver in `go-virtio/*` consumes `common.Transport` and nothing
else. Swap the backplane, keep the driver.

## Project standards

- **Pure Go.** No cgo, no syscalls into a host driver, no kernel
  dependency.
- **100% test coverage** is the bar on every module. `common` is at
  ~96% en route to 100%; `net` is at 81% climbing to 100%.
- **BSD-3-Clause** on all source files.
- **Spec-traceable.** Every register, descriptor field, and feature bit
  carries a citation back to the Virtio 1.1 spec section it implements.
- **Multi-arch.** Pure-Go code with no architecture-specific assembly,
  so the same drivers work on amd64, arm64, riscv64, loongarch64.

## Who uses it

The first consumer is [cloud-boot](https://github.com/cloud-boot) —
specifically the pure-UEFI TamaGo loader that drives the guest's
virtio-net NIC during PXE / HTTP boot under Apple
`Virtualization.framework` and KVM/OVMF. The intent is that any Go
project running close to the metal (microVM runtimes, bootloaders,
unikernels, test harnesses) can pull in `go-virtio/*` instead of
re-implementing the same `struct virtio_pci_cap` walk.

## Landing page

Project landing page: <https://go-virtio.github.io>.
