<div align="center">

<img src="https://raw.githubusercontent.com/Hartwell-Labs/.github/main/profile/assets/hartwell-logo.svg" width="72" alt="Hartwell Labs" />

## Fortis

Chain-of-trust attestation for embedded systems — Rust on RISC-V, verifiable boot.

[![Rust](https://img.shields.io/badge/Rust-RISC--V-F15A24?style=flat-square&logo=rust)](.)
[![License](https://img.shields.io/badge/license-MIT-F15A24?style=flat-square)](LICENSE) [![Website](https://img.shields.io/badge/site-hartwell--labs.github.io-4f46e5?style=flat-square)](https://hartwell-labs.github.io)

[Website](https://hartwell-labs.github.io) · [All products](https://hartwell-labs.github.io/products/) · [Security](https://hartwell-labs.github.io/security/) · [Hack the Lab](https://github.com/Hartwell-Labs/hack-the-lab)

</div>

**Measured boot, zero trust.** — Bare-metal RISC-V chain-of-trust with real
SHA-256 and ML-KEM-768 post-quantum crypto on QEMU virt.

## What it demonstrates

| Feature | Implementation |
|---------|---------------|
| SHA-256 | `sha2` crate (FIPS 180-4) — real SHA-256, not a placeholder |
| ML-KEM-768 | `ml-kem` crate (FIPS 203) — post-quantum key encapsulation |
| MMIO UART | ns16550a driver for QEMU virt (`0x10000000`) |
| Measured boot | SHA-256 PCR registers over **real binary sections** |
| Chain of trust | 3-stage measurement with attestation report |
| Binary size | Under 64 KiB code (stripped, LTO, `opt-level=z`) |
| Section proofs | Prints real linker addresses + sizes for `.text` and `.rodata` |
| Host verification | External verifier reads ELF, computes expected PCR, compares |

## How it works

1. **Firmware** runs in QEMU, measures its own `.text` and `.rodata` sections
   using linker-defined addresses, prints PCR values.
2. **Verifier** (`scripts/verify/`) reads the ELF binary independently,
   computes expected PCR values from the actual section bytes, runs the
   firmware in QEMU, parses the output, and compares.
3. **External root of trust**: the verifier is the trust anchor — it doesn't
   need to trust the firmware's self-report.

## Limitations

| Limitation | Detail |
|------------|--------|
| **Simulated TPM** | PCR registers are `[u8; 32]` in RAM, not hardware TPM registers. The verifier acts as external root of trust. A production system would use a real TPM chip (PCR 16-23 for application use). |
| **No secure boot** | QEMU loads the binary directly from `-kernel`. There is no hardware root of trust, no signature verification, and no firmware signing. |
| **Key rotation** | Keys are generated with real OS randomness via `scripts/keygen`, but re-rotation requires re-running the keygen tool manually. |

## Why `no_std`

This project runs on **bare-metal RISC-V** without any OS, bootloader, or
heap allocator:

- **No standard library** — there is no OS to provide it
- **No heap** — `Vec`, `Box`, `String` etc. are unavailable
- **Full memory control** — linker script, BSS clearing, stack layout
- **Consistent with the ecosystem** — eBPF (talus) also uses `no_std`

## Architecture

```
┌─────────────────────────────────────────────────────┐
│ Stage 0: Reset vector → global_asm! _start          │
│   • Mask interrupts (supervisor mode)               │
│   • Set up 16 KiB stack                             │
│   • Clear BSS section                               │
│   • Jump to rust_main()                              │
├─────────────────────────────────────────────────────┤
│ Stage 1: Measure .text (real machine code)          │
│   • SHA-256(_text_start.._text_end) → PCR[0]       │
│   • Prints real address + size from linker          │
├─────────────────────────────────────────────────────┤
│ Stage 2: Measure .rodata (real const data)          │
│   • SHA-256(PCR[0] || _rodata) → PCR[1]            │
│   • Chained measurement (PCR[0] feeds into PCR[1])  │
├─────────────────────────────────────────────────────┤
│ Stage 3: ML-KEM-768 post-quantum verification       │
│   • Decapsulate ciphertext → shared secret           │
│   • Verify shared secret matches expected            │
│   • Extend PCR[2] with shared secret                 │
├─────────────────────────────────────────────────────┤
│ Stage 4: Attestation report                         │
│   • Print PCR bank (PCR[0], PCR[1], PCR[2])         │
│   • Chain of trust verdict: PASSED / FAILED          │
└─────────────────────────────────────────────────────┘
```

### Verification flow

```
Build firmware
    ↓
scripts/verify/
    ├─ 1. Read ELF → extract .text, .rodata bytes (by section name)
    ├─ 2. Compute SHA-256 chain → expected PCR[0], PCR[1]
    ├─ 3. ML-KEM-768 decapsulate (deterministic from keys.rs) → PCR[2]
    ├─ 4. Run firmware in QEMU → parse "PCR[i] = 0x..." output
    ├─ 5. Compare host-computed vs firmware-printed
    └─ 6. PASSED or FAILED
```

## Crypto details

### SHA-256 (FIPS 180-4)

```rust
// PCR extend: PCR = SHA-256(PCR_old || data)
let mut hasher = Sha256::new();
hasher.update(pcr);
hasher.update(data);
pcr.copy_from_slice(&hasher.finalize());
```

### ML-KEM-768 (FIPS 203)

```rust
// Deterministic key derivation from seed
let dk = DecapsulationKey::<MlKem768>::from_seed(seed);
let ct = Array::try_from(CT.as_slice()).unwrap();

// Decapsulate ciphertext → shared secret (deterministic)
let ss = dk.decapsulate(&ct);
```

Keypair and ciphertext are generated on the host using real OS randomness
(`getrandom` crate) via `scripts/keygen/`.

## Connections to the portfolio

```
pqguard (post-quantum crypto)  ←→  Fortis (firmware verification)
        ↓                                    ↓
   ML-KEM-768                         Chain of trust
        ↓                                    ↓
talus (eBPF monitoring)        ←→  Runtime attestation
```

## Running

### Prerequisites

```bash
# Install Rust target
rustup target add riscv64gc-unknown-none-elf
rustup component add rust-src

# Install QEMU (Ubuntu/Debian)
sudo apt install qemu-system-misc
```

### Build & verify

```bash
# Build firmware
cargo build --target riscv64gc-unknown-none-elf --release

# Run host-side verifier (reads ELF, runs QEMU, compares)
cargo run --manifest-path scripts/verify/Cargo.toml --release
```

### Run in QEMU manually

```bash
qemu-system-riscv64 \
    -machine virt \
    -bios default \
    -nographic \
    -kernel target/riscv64gc-unknown-none-elf/release/fortis
```

### Verifier output

```
=== Fortis Verifier ===
ELF: target/riscv64gc-unknown-none-elf/release/fortis

ELF sections:
  .text:   addr=0x80200000  size=44878
  .rodata: addr=0x8020b000  size=7456

Expected PCR values (host-computed):
  PCR[0] = 0xd146f17d84c41e3c8fb888acecb534d8...
  PCR[1] = 0xb3dcab5d6e0b885687e09cb1d5bd1886...
  PCR[2] = 0x230fa3d24255c839356e2dc5b34772fa...

Running firmware in QEMU...
Firmware PCR values (from QEMU):
  PCR[0] = 0xd146f17d84c41e3c8fb888acecb534d8...
  PCR[1] = 0xb3dcab5d6e0b885687e09cb1d5bd1886...
  PCR[2] = 0x230fa3d24255c839356e2dc5b34772fa...

Verification:
  PCR[0] ✓ match
  PCR[1] ✓ match
  PCR[2] ✓ match

=== VERIFICATION: PASSED ===
```

## Project structure

```
fortis/
├── src/
│   ├── main.rs          # Entry point + chain of trust (measures real sections)
│   ├── uart.rs          # MMIO UART driver (ns16550a)
│   └── keys.rs          # ML-KEM-768 keypair (generated by scripts/keygen)
├── link.ld              # Linker script (QEMU virt, 0x80200000)
├── build.rs             # Build script (linker flags)
├── Cargo.toml
├── scripts/
│   ├── keygen/          # Generate ML-KEM-768 keys with real randomness
│   └── verify/          # Host-side verifier (reads ELF, runs QEMU, compares)
├── .github/workflows/
│   └── ci.yml           # Build + verify + size check + clippy
└── README.md
```

## Key generation

```bash
cd scripts/keygen && cargo run --release
# Output: Rust source → paste into src/keys.rs
```

## License

MIT

## Deep Dives

Extended dossiers (architecture, verification, benchmarks, error codex) ship in this repo:
- [SECURITY.md](SECURITY.md)
---

<div align="center">

**[Hartwell Labs](https://github.com/Hartwell-Labs)** — security systems, languages and tools, built in the open.

[Website](https://hartwell-labs.github.io) · [All products](https://hartwell-labs.github.io/products/) · [Security policy](https://hartwell-labs.github.io/security/) · [Report a vulnerability](https://hartwell-labs.github.io/security/)

<sub>MIT License · © 2026 Hartwell Labs</sub>

</div>
