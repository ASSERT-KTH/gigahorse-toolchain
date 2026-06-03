# Running Gigahorse without Docker

This documents a one-time bootstrap that extracts the necessary native binaries
from the official Docker image so that `gigahorse.py` can run directly on the
host, with no Docker overhead on subsequent invocations.

Tested on Ubuntu 25.10 (host) against image
`ghcr.io/nevillegrech/gigahorse-toolchain:latest` (Ubuntu 22.04 inside).

## What needs to come out of the image

Gigahorse's Python source is pure stdlib — no pip packages required.  The only
native pieces are the Souffle Datalog engine and two addon shared libraries:

| File in image | Purpose |
|---|---|
| `/usr/local/bin/souffle` | Souffle 2.4.1 binary |
| `/usr/local/bin/souffle-compile.py` | Souffle's C++ compilation helper |
| `/usr/local/include/souffle/` (96 headers) | Headers needed to compile `.dl` → executable |
| `souffle-addon/libsoufflenum.so` | Gigahorse numeric functor library |
| `souffle-addon/libfunctors.so` | Symlink → `libsoufflenum.so` |
| `souffle-addon/functor_includes.dl` | Datalog declarations for the functors |

## One-time bootstrap

```bash
GIGAHORSE_ROOT=~/.gigahorse
TOOLCHAIN=~/workspace/proposals/gigahorse-experiments/gigahorse-toolchain

mkdir -p "$GIGAHORSE_ROOT/bin" "$GIGAHORSE_ROOT/include"

CID=$(docker create ghcr.io/nevillegrech/gigahorse-toolchain:latest)

# Souffle binary and compile helper
docker cp $CID:/usr/local/bin/souffle            "$GIGAHORSE_ROOT/bin/souffle-extracted"
docker cp $CID:/usr/local/bin/souffle-compile.py "$GIGAHORSE_ROOT/bin/souffle-compile.py"
chmod +x "$GIGAHORSE_ROOT/bin/souffle-extracted"

# C++ headers (needed to compile .dl files on first analysis run)
docker cp $CID:/usr/local/include/souffle "$GIGAHORSE_ROOT/include/souffle"
mkdir -p "$GIGAHORSE_ROOT/bin/include"
ln -sfn "$GIGAHORSE_ROOT/include/souffle" "$GIGAHORSE_ROOT/bin/include/souffle"

# Functor shared libraries (must live in the toolchain's souffle-addon/)
docker cp $CID:/opt/gigahorse/gigahorse-toolchain/souffle-addon/libsoufflenum.so \
    "$TOOLCHAIN/souffle-addon/libsoufflenum.so"
docker cp $CID:/opt/gigahorse/gigahorse-toolchain/souffle-addon/functor_includes.dl \
    "$TOOLCHAIN/souffle-addon/functor_includes.dl"
ln -sf libsoufflenum.so "$TOOLCHAIN/souffle-addon/libfunctors.so"

docker rm $CID
```

> **Note:** The `libsoufflenum.so` from the Docker image is older and lacks the `hex_normalized`
> functor (needed by `clientlib/multi_contract.dl` and any client that imports it).  Rebuild it
> from the submodule source so clients that use `@hex_normalized` can link:
>
> ```bash
> cd "$TOOLCHAIN/souffle-addon"
> CPLUS_INCLUDE_PATH="$GIGAHORSE_ROOT/include" make libsoufflenum.so
> ```
>
> This requires `libz3-dev` (`apt install libz3-dev`).  The `libfunctors.so` symlink is
> recreated automatically by `make`.

## Patch souffle-compile.py

`souffle-compile.py` detects the `include/souffle/` directory sitting next to
itself, but only injects the resulting `-I` flag for SWIG builds — not for the
normal executable compilation path.  Add three lines in the `else` branch that
builds the compile command (around line 215):

```diff
     cmd.append(conf['includes'])
+    # inject detected souffle include dir when running outside Docker
+    if souffle_include_dir:
+        cmd.append("-I{}".format(souffle_include_dir.parent))
     cmd.append(conf['std_flag'])
```

## Wrapper script

Save as `~/.gigahorse/bin/gigahorse-native` and `chmod +x`:

```bash
#!/usr/bin/env bash
GIGAHORSE_ROOT="$(realpath ~/.gigahorse)"
TOOLCHAIN="$(realpath "$(dirname "$0")/../../workspace/proposals/gigahorse-experiments/gigahorse-toolchain")"
SOUFFLE="${GIGAHORSE_ROOT}/bin/souffle-extracted"
CACHE="${GIGAHORSE_ROOT}/cache"
TEMP="${GIGAHORSE_ROOT}/.temp"

mkdir -p "${CACHE}" "${TEMP}"

# souffle-extracted looks for souffle-compile.py in the same directory as itself
export PATH="${GIGAHORSE_ROOT}/bin:${PATH}"
export LD_LIBRARY_PATH="${TOOLCHAIN}/souffle-addon:${LD_LIBRARY_PATH}"

exec python3 "${TOOLCHAIN}/gigahorse.py" \
    -S "${SOUFFLE}" \
    --cache_dir "${CACHE}" \
    -w "${TEMP}" \
    "$@"
```

## First run

The first run compiles all `.dl` files to C++ executables and caches them.
This takes several minutes (same as inside Docker on a cold start):

```bash
gigahorse-native contracts/foo.hex
```

## Subsequent runs

Pass `--reuse_datalog_bin` to skip recompilation and use the cached
executables.  A three-contract analysis then takes ~150 ms:

```bash
gigahorse-native --reuse_datalog_bin contracts/*.hex
```

## Running a custom client

```bash
gigahorse-native --reuse_datalog_bin \
    -C gigahorse-toolchain/clients/reentrancy.dl \
    contracts/*.hex
```

Results land in `~/.gigahorse/.temp/<ContractName>/out/*.csv`.

## Discrepancy between the Docker image and the Git repository

The published image (`ghcr.io/nevillegrech/gigahorse-toolchain:latest`) lags
behind the `main` branch of the repository.  As of June 2025 the gap is
significant: **6 new source files** and **43 modified files** across `logic/`,
`clientlib/`, and `src/`.

### Files present in the repo but absent from the Docker image

These are compiled from source on first native run (the patch above makes that
possible):

| File | What it adds |
|---|---|
| `logic/last_resort.dl` | Final-fallback decompiler pass used when both the precise and scalable fallbacks still leave unresolved jumps |
| `clientlib/multi_contract.dl` | Cross-contract call resolution library |
| `clientlib/storage_modeling/storage_modeling_api.dl` | Public API surface for the storage model |
| `clientlib/storage_modeling/type_inference.dl` | Slot-level type inference for storage variables |
| `clientlib/storage_modeling/clienthelpers.dl` | Helper predicates for storage-model clients |
| `src/tac_schema.py` | Python-side schema description of the TAC output relations |

### Modified files (same name, different content)

43 files differ.  The changes are spread across all subsystems:

- **`logic/`** (15 files): context-sensitivity strategies, decompiler output
  schema, analytics, local/global analysis passes, statement insertor.
- **`clientlib/`** (20 files): TAC instruction set, data-flow, storage and
  memory modeling, function inliner, loop analysis.
- **`src/`** (8 files): disassembler, block parser, exporter, runners — all the
  Python glue that drives Souffle.

### Practical impact

- The compiled executables cached by Docker (e.g. `main.dl_compiled`) are built
  from the **image's** source, not the repo's.  If you use `--reuse_datalog_bin`
  after a Docker bootstrap, Gigahorse runs the older compiled logic even though
  it is driven by the newer Python front-end.  The two are broadly compatible but
  the newer decompiler passes (`last_resort.dl`, improved storage typing) are not
  exercised.

- The safest setup is to let the native run recompile everything from source
  once (the patch ensures this succeeds) so that the compiled logic matches the
  Python front-end exactly.  After that one compilation pass, `--reuse_datalog_bin`
  is safe because you are reusing *your* compiled executables, not Docker's.
