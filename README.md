# jeth

Jolt proving Ethereum: stateless execution of a full, recent Ethereum mainnet block inside a Jolt RISC-V guest (pre-state witness → execute all txs → assert post-state root against the header), traced — not proved — to measure RISC-V cycle count and cycles/gas.

Status: **optimization campaign in progress**. Current: the ten-block mainnet set 25905781–25905790 validates at **13.65 cycles/gas** (gas-weighted, fully self-verifying, 2026-09-10) — from 513 c/g stock. See [RESULTS.md](RESULTS.md) for the full ladder, row-exact profiles, quartering plan, and proving memo; design in [PLAN.md](PLAN.md).

Quick start: `just jeth bench`, which runs `cargo run --release -p jeth-host -- bench` (one-time Jolt CLI build: RESULTS.md → Reproduce). `just` alone lists the build, format, lint and test recipes.

Upstreamed: [a16z/jolt#1746](https://github.com/a16z/jolt/pull/1746) — O(1) size-class guest allocator (the 85.7%-of-cycles finding).

### Optional short-input Keccak cache

The standalone guest feature `keccak-cache` memoizes 32- and 64-byte Keccak
inputs in a fixed 64-entry cache (8 KiB of bucket storage). Enable it alongside
`guest` in both the advice and proven guest builds. It is disabled by default:
repeated inputs save Keccak permutations, while misses add lookup/copy work.
Every hit compares the complete input and length; index-hash collisions only
cause eviction. The feature changes guest execution, not the Jolt protocol.

On mainnet blocks 26235156, 26235168, and 26235159, matched full guest traces
used 0.79%, 1.00%, and 1.43% fewer expanded rows, respectively. These are trace
counts, not end-to-end prover timings. An isolated stream of distinct,
deliberately colliding 32-/64-byte inputs added 8.5% to the hash routine's row
count. Workloads with little reuse may therefore be slower with the cache.

## Native advice generation

The optional `jeth-core/native-advice` feature records the untrusted advice words
emitted during native validation. It can replace the interpreted `compute_advice`
pass; the proven guest still checks every hint and executes the same instructions.

For the ordinary guest with the code library enabled:

```sh
cargo run --release -p jeth-core --example native-advice \
  --features native-advice,secp-inline,code-library,bigint-inline -- input.bin advice.bin
```

Pass the resulting bytes as Jolt's runtime advice tape, not its trusted-advice
input. Match the native and guest source revisions, dependencies, and features.
The example runs one block per process and does not support the trusted-digest
guest or optional guest features such as lazy analysis. Existing tracing commands
continue to use the interpreted pass.

The library entry point is `jeth_core::advice::native::record`. It captures words
from both jeth-core and the vendored MPT encoder, on the calling thread only.
Recording scopes cannot nest; unwinding discards the tape. The recorder does not
reset other validation state after a failed run.

### Smaller committed-input guest

`jeth trace --input input.bin --small-input` selects `validate_block_16m`,
which reserves 16 MiB for committed input. It executes the same validation as
`validate_block`; the default remains 128 MiB for large and multi-block inputs.
The limit includes the postcard argument-length prefix. Oversized inputs fail
before guest compilation or execution; rerun without `--small-input` to use the
larger guest. This option cannot be combined with the trusted-advice modes.

The smaller I/O reservation can reduce the RAM address domain the prover must
handle, even with the same execution trace length. The effect depends on the
program's highest accessed RAM address. The two capacities have separate ELF,
compute-advice ELF, and trace-summary paths, including with `--skip-build`.
Other guest features can still be selected with `--guest-features`.

For a custom prover integration, build with
`JOLT_FUNC_NAME=validate_block_16m` and use a matching `MemoryConfig`:
`max_input_size = 16777216`; output and both advice budgets remain 4096 bytes,
stack 33554432 bytes, and heap 1610612736 bytes. Build both advice and proven
ELFs for that entry point. Keep the usual `validate_block` setup for inputs
that need the larger region.
