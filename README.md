# jeth

## Combined performance branch (2026-11-09)

This branch combines Jeth PRs #23–#36 for integration testing; each change remains
reviewable in its individual PR. It includes the bounded, checked trie-prefix cache
from #36, but no interpreter replacement or unsubmitted packed-trie changes. The guest SDK remains
pinned to `8e536f199735ac8187156aa589ed1387722a079a`.

With native advice and guest features
`guest,keccak-cache,kzg-residue,bn254-pairing-residue,keccak-prefix-cache` (no default features), expanded
trace counts are:

| Ethereum block | Main `dac77ea` | Combined PRs |
|---|---:|---:|
| 26235156 | 91,332,907 | 84,128,991 |
| 26235168 | 549,152,419 | 459,358,991 |
| 26235159 | 919,682,516 | 782,446,835 |

All three preserve canonical block hash and gas. The small block also has a
complete CPU Akita proof accepted by both parallel and single-threaded verifiers
(106,497 bytes). Local default workspace tests, feature tests, guest differential
tests and strict Clippy checks pass. These counts are
not prover-time speedup measurements. The faster
research-build counts quoted elsewhere include additional changes and a newer SDK.

Use Rust 1.95 and a Jolt CLI from the pinned SDK revision. Build the guest from
`crates/guest` with `JOLT_FUNC_NAME=validate_block`, `JOLT_BACKTRACE=1`, and:

```sh
jolt build -p jeth-guest --backtrace off --stack-size 33554432 --heap-size 1610612736 -- \
  --locked --release --no-default-features --features guest,keccak-cache,kzg-residue,bn254-pairing-residue,keccak-prefix-cache
```

Generate matching advice from the repository root:

```sh
cargo run --release --locked -p jeth-core --example native-advice --features \
  native-advice,secp-inline,sha2-inline,blake2-inline,blake2f-inline,p256-inline,bigint-inline,bn254-inline,bls12-381-inline,code-library,kzg-residue,bn254-pairing-residue,keccak-prefix-cache \
  -- INPUT.jef.bin advice.bin
```

Supply this as **untrusted runtime advice**. The measured configuration uses the
128 MiB committed-input entry point, not `validate_block_16m`: maximum input
134217728, output 4096, heap 1610612736, stack 33554432, and both advice budgets 4096.
The residue and prefix-cache features must match between the native producer and guest.


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

### Checked Keccak prefixes

The optional `keccak-prefix-cache` feature reuses states computed by the guest
while hashing trie nodes of 136–544 bytes. It retains up to three complete
136-byte input blocks. Advice selects a stored prefix; the guest compares every
prefix byte before continuing the hash. Advice never supplies the sponge state.

Enable the feature in both the guest and its matching native advice generator.
The native recorder plans which prefixes will be reused and emits cache misses
after 131,072 eligible calls. Guest storage is capped at 65,536 entries (22 MiB of
entry payload); a full cache keeps hashing normally. Hint consumption depends
only on input length, so different buffer alignments cannot change tape order.
The interpreted `compute_advice` pass emits cache misses and remains correct;
use the native planner to obtain the row savings.

The feature is disabled by default. On the combined #23–#35 branch, with native
advice and all other settings held fixed, it reduces live rows from
88,170,922 / 473,666,778 / 807,159,643 to
84,128,991 / 459,358,991 / 782,446,835 for blocks
26235156 / 26235168 / 26235159: 4.58%, 3.02%, and 3.06% fewer rows.
These are execution counts, not measured prover-time speedups. Inputs with
little prefix reuse can be slower with this feature enabled.

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
