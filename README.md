# arceos-loadapp

A standalone filesystem-based application loader running on [ArceOS](https://github.com/arceos-org/arceos) unikernel, with all dependencies sourced from [crates.io](https://crates.io). Demonstrates **FAT filesystem initialization, file I/O, and VirtIO block device driver** across multiple architectures.

## What It Does

This application demonstrates the full I/O stack from filesystem down to block device:

1. **VirtIO-blk driver**: Automatically discovered and initialized via PCI bus probing.
2. **FAT filesystem**: Mounted on the VirtIO block device during ArceOS runtime startup.
3. **File read**: Opens `/sbin/origin.bin` from the FAT filesystem and reads its first 64 bytes.
4. **Child task**: Spawns a worker thread that prints the first 8 bytes of the file as hex values.
5. **CFS scheduling**: Uses preemptive CFS scheduler for task management.

### I/O Stack

```
Application (std::fs::File)
    └── axfs (FAT filesystem)
        └── axdriver (VirtIO-blk)
            └── virtio-drivers (PCI transport)
                └── QEMU VirtIO block device
```

## Supported Architectures

| Architecture | Rust Target | QEMU Machine | Platform |
|---|---|---|---|
| riscv64 | `riscv64gc-unknown-none-elf` | `qemu-system-riscv64 -machine virt` | riscv64-qemu-virt |
| aarch64 | `aarch64-unknown-none-softfloat` | `qemu-system-aarch64 -machine virt` | aarch64-qemu-virt |
| x86_64 | `x86_64-unknown-none` | `qemu-system-x86_64 -machine q35` | x86-pc |
| loongarch64 | `loongarch64-unknown-none` | `qemu-system-loongarch64 -machine virt` | loongarch64-qemu-virt |

## Prerequisites

- **Rust nightly toolchain** (edition 2024)
- **QEMU** for target architectures
- **rust-objcopy** (`cargo install cargo-binutils`)

## Quick Start

```bash
cargo install cargo-clone
cargo clone arceos-loadapp
cd arceos-loadapp

# Build and run on RISC-V 64 QEMU (default)
cargo xtask run

# Other architectures
cargo xtask run --arch aarch64
cargo xtask run --arch x86_64
cargo xtask run --arch loongarch64
```

Expected output:

```
Load app from fat-fs ...
fname: /sbin/origin.bin
Wait for workers to exit ...
worker1 checks code:
0x10 0x21 0x32 0x43 0x54 0x65 0x76 0x87
worker1 ok!
Load app from disk ok!
```

## Project Structure

```
app-loadapp/
├── .cargo/
│   └── config.toml       # cargo xtask alias & AX_CONFIG_PATH
├── xtask/
│   └── src/
│       └── main.rs       # build/run tool (FAT32 disk image + QEMU)
├── configs/
│   ├── riscv64.toml
│   ├── aarch64.toml
│   ├── x86_64.toml
│   └── loongarch64.toml
├── src/
│   └── main.rs           # File open/read + worker thread
├── build.rs
├── Cargo.toml
└── README.md
```

## Key Components

| Component | Role |
|---|---|
| `axstd` | ArceOS standard library (`std::fs::File`, `std::io`, `std::thread`) |
| `axfs` | Filesystem module — mounts FAT32 on the VirtIO block device |
| `axdriver` | Device driver framework — VirtIO-blk via PCI bus |
| `axtask` | Task scheduler with CFS algorithm |
| `fatfs` (xtask) | Creates the FAT32 disk image with `/sbin/origin.bin` at build time |

## How the Disk Image Works

The `xtask` tool uses the `fatfs` Rust crate to create a 64MB FAT32 disk image (`target/disk.img`):

1. Allocates a 64MB raw file
2. Formats it as FAT32 using `fatfs::format_volume()`
3. Creates `/sbin/` directory
4. Writes `/sbin/origin.bin` with 64 bytes of sample binary data
5. Attaches the image to QEMU as `-device virtio-blk-pci`

No external tools (`mkfs.fat`, `mtools`) are required.

## License

GPL-3.0-or-later OR Apache-2.0 OR MulanPSL-2.0
