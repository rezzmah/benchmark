# Conduit State Root Strategy Benchmark Guide

This document explains how to run the benchmark comparing `deferred` vs `every-flashblock` state root strategies in the conduit op-rbuilder, and how to interpret the results.

## Overview

The benchmark measures gas/second throughput under two `--conduit.state_root_strategy` modes:

- **`every-flashblock`** (default): computes a state root after every flashblock.
- **`deferred`**: defers state root computation to the last flashblock only.

It also varies the flashblock interval (`--flashblocks.block-time`) to amplify the difference:

- **250ms** (~4 flashblocks per 1s block)
- **10ms** (~100 flashblocks per 1s block)

This produces 4 test runs (2 strategies x 2 intervals), each building 10 blocks.

## Prerequisites

| Tool | Notes |
|------|-------|
| Go 1.22+ | For building the benchmark CLI |
| Rust / Cargo | For building op-rbuilder and reth |
| Foundry (`forge`) | For building benchmark contracts |
| Node.js + npm | Optional, for the interactive report dashboard |

The following repos must be cloned locally:

```
~/Code/benchmark            # github.com/base/base-bench
~/Code/conduit-op-rbuilder  # your conduit op-rbuilder fork
```

## Step 1: Build Binaries

### 1a. Benchmark CLI

```bash
cd ~/Code/benchmark
make build
```

This produces `./bin/base-bench`.

### 1b. Contracts

The `make contracts` target runs from a subdirectory that can't find the root `foundry.toml`. Run forge manually from the repo root instead:

```bash
cd ~/Code/benchmark

# Build ABI
forge inspect contracts/src/Simulator.sol:Simulator abi --json > contracts/abi/Simulator.json

# Build bytecode
forge build --extra-output-files bin --force

# Generate Go bindings
go run github.com/ethereum/go-ethereum/cmd/abigen \
  --abi contracts/abi/Simulator.json \
  --pkg abi \
  --type Simulator \
  --out runner/payload/simulator/abi/Simulator.go \
  --bin ./contracts/out/Simulator.sol/Simulator.bin
```

### 1c. Reth validator binary

```bash
cd ~/Code/benchmark
make build-reth
```

This clones reth (version from `clients/versions.env`, currently v1.9.3), builds `op-reth` with the `maxperf` profile, and places it at `./bin/op-reth`.

### 1d. Conduit op-rbuilder

```bash
cd ~/Code/conduit-op-rbuilder
cargo build -p op-rbuilder --bin op-rbuilder --release
```

This produces `target/release/op-rbuilder`.

> **Important:** Do NOT use `make build-rbuilder` from the benchmark repo -- that clones upstream, not your local fork.

## Step 2: Benchmark Config

The config file is already checked in at:

```
~/Code/benchmark/configs/conduit-stateroot-comparison.yml
```

It defines 4 test runs with a 100 gigagas gas limit:

| Test | Strategy | Flashblock interval |
|------|----------|-------------------|
| 1 | every-flashblock | 250ms |
| 2 | deferred | 250ms |
| 3 | every-flashblock | 10ms |
| 4 | deferred | 10ms |

You can edit the `gas_limit`, `num_blocks`, or `--flashblocks.block-time` values as needed.

### How it works

The `rbuilder` client adapter automatically adds `--flashblocks.enabled`, `--flashblocks.port`, and `--flashblocks.fixed`. The `node_args` are appended after. Since `--conduit.enabled` takes precedence over `--flashblocks.enabled`, the builder runs in conduit mode.

## Step 3: Run the Benchmark

```bash
cd ~/Code/benchmark

./bin/base-bench run \
  --config configs/conduit-stateroot-comparison.yml \
  --root-dir /tmp/bench-data \
  --output-dir ./output \
  --rbuilder-bin ~/Code/conduit-op-rbuilder/target/release/op-rbuilder \
  --reth-bin ~/Code/conduit-op-rbuilder/target/release/op-rbuilder
```

> **Note on `--output-dir`:** Use `./output` (relative to the repo root) so the report viewer can find the results. The viewer's Vite build copies from `../output/**/*`. If you previously ran with a different path (e.g., `/tmp/bench-output`), you can symlink it: `ln -s /tmp/bench-output ./output`

> **Note on `--reth-bin`:** We point this at the op-rbuilder binary (not `./bin/op-reth`) because the benchmark passes `node_args` (including `--conduit.*` flags) to both the sequencer and validator nodes. Plain `op-reth` doesn't understand conduit flags and will crash. Using the op-rbuilder binary for both roles avoids this.

### Reruns

Each benchmark run gets an auto-generated `BenchmarkRunID`. When writing results, the tool reads the existing `metadata.json`, **replaces** any runs sharing the same `BenchmarkRunID`, and preserves everything else. This means:

- **Reruns accumulate** in `metadata.json` -- each invocation generates a new ID, so previous results are kept alongside new ones.
- **To get a clean slate**, delete `./output/metadata.json` (or the entire `./output` directory) before running.
- **To replace a specific run**, pass `--benchmark-run-id <id>` matching an existing run's ID. Only those entries are overwritten.

## Step 4: Interpret Results

### Quick summary

```bash
cat ./output/metadata.json | python3 -m json.tool
```

Each run appears as an entry with its tags (`strategy`, `fb_interval`) and summary metrics. The primary metric is `sequencerMetrics.gasPerSecond`.

### Key metrics

| Metric | Source | What it tells you |
|--------|--------|-------------------|
| `gasPerSecond` | `metadata.json` | Primary throughput metric |
| `getPayload` | `metadata.json` | Time to retrieve built payload (seconds) |
| `reth_op_rbuilder_state_root_calculation_duration` | `metrics-sequencer.json` | Direct state root computation time |
| `reth_op_rbuilder_flashblock_build_duration` | `metrics-sequencer.json` | Per-flashblock build time |
| `reth_op_rbuilder_total_block_built_duration` | `metrics-sequencer.json` | Total block build time |

### Per-block detail

```bash
cat ./output/test-*/metrics-sequencer.json | python3 -m json.tool
```

### Interactive dashboard

```bash
cd ~/Code/benchmark/report
npm install && npm run dev
# Open http://localhost:3000
```

### Expected behavior

- **`every-flashblock`**: Each flashblock pays the state root cost. Throughput degrades as flashblock count increases.
- **`deferred`**: Intermediate flashblocks skip state root computation. Throughput stays stable regardless of flashblock count. The last flashblock bears the full state root cost.
- **At 10ms intervals**: The difference between strategies is amplified (~100 state roots vs 1).

## Output Structure

```
./output/
  metadata.json                          # Summary of all runs (accumulates across reruns)
  test-<id>-0/
    metrics-sequencer.json               # Prometheus metrics from sequencer
    metrics-validator.json               # Prometheus metrics from validator
    result-sequencer.json                # Per-block sequencer results
    result-validator.json                # Per-block validator results
    logs-sequencer.gz                    # Compressed sequencer logs
    logs-validator.gz                    # Compressed validator logs
```

## Verification Checklist

After a run, confirm both strategies executed correctly:

1. All runs show `"success": true` in `metadata.json`
2. `reth_op_rbuilder_block_built_success` counter incremented for all runs
3. `reth_op_rbuilder_state_root_calculation_duration` shows different patterns between strategies
4. Logs (`logs-sequencer.gz`) show `conduit` builder mode activated

## Troubleshooting

### Validator crashes with "connection refused"

The benchmark passes `node_args` to both the sequencer and validator. If `--reth-bin` points to a plain `op-reth` binary, it won't recognize `--conduit.*` flags and will crash immediately. Fix: use the op-rbuilder binary for `--reth-bin` as shown above.

### Contracts fail to build with `make contracts`

The root `foundry.toml` sets `src = "contracts/src"` but `make contracts` runs forge from the `contracts/` subdirectory. Use the manual forge commands from Step 1b instead.

### Benchmark takes too long

Reduce `num_blocks` in the config (e.g., from 10 to 5), or remove the 10ms flashblock interval tests which produce ~100 flashblocks per block.

## Known Caveats

1. **`--flashblocks.fixed` is hardcoded** in the rbuilder adapter. Both runs use fixed-interval flashblocks (not dynamic). This is fine for an apples-to-apples comparison.

2. **gas/second includes BlockTime sleep.** The `blockBuildingDuration` used to compute gas/second spans from `engine_forkchoiceUpdatedV3` through `engine_getPayloadV4`, including the 1s block time sleep. Both strategies are measured identically so the comparison is valid, but absolute numbers include idle time.

3. **Metric name mismatch.** `reth_op_rbuilder_payload_tx_simulation_duration` won't be collected (benchmark expects `payload_tx_simulation_duration`, builder emits `payload_transaction_simulation_duration`). This doesn't affect key metrics.
