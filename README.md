# Breeze 5

> **Important:** The engine hasn't been tested bit for bit against a trusted evaluator.


> **Note:** Breeze Engine will have double releases in the future, one will be the UCI compatible version, and the other one will be the main version of the engine where I, Sapphire, will put a bigger quantity of my work, as this newer version will use my own interface, BCI(Breeze Chess Interface), this is to ensure compatibility and to avoid trouble with commands later, the UCI version is assured to NEVER be deprecated, I will put as much work as I can to ensure both the Universal Chess Interface and Breeze Chess Interface versions have the same playing strength.

## Included

- C++17 engine executable named `Breeze5`.
- Original 64-square board, FEN parser/serializer, legal move generation, castling, en-passant, promotion, make/unmake, and incremental Zobrist hashing.
- Iterative-deepening alpha-beta search with PVS, aspiration windows, quiescence search, transposition table, null-move pruning, late-move reductions, killer/history ordering, repetition and 50-move handling, and insufficient-material detection.
- UCI protocol support for the common ChessBase workflow: `uci`, `isready`, `ucinewgame`, `setoption`, `position`, `go`, `stop`, `ponderhit`, and `quit`; plus useful diagnostic commands `eval`, `d`, and `perft`.
- `go depth`, `go nodes`, `go movetime`, clock controls, `go searchmoves`, `go mate`, `go infinite`, `go ponder`, and `go perft`.
- UCI options: `Hash`, `Threads` (currently offering 1 to 64), `Move Overhead`, `Use NNUE`, `EvalFile`, and `Clear Hash`.
- SFNNv20 loader and original scalar inference for K32H3, FullThreats, PP_3Wide, 92 compact layer stacks, and the 1024→32→32→1 head.
- Perft regression tests covering six reference positions.

## Build

### Windows — MSYS2 UCRT64

Open the **MSYS2 UCRT64** terminal, make sure the UCRT64 GCC toolchain and `make` are installed, then enter the project folder:

```bash
pacman -S --needed mingw-w64-ucrt-x86_64-gcc make
./scripts/build_ucrt64.sh
```

The script builds `Breeze5.exe` with static GCC, C++, and winpthreads runtime linkage. This is important when launching the engine from ChessBase: ChessBase may not inherit the MSYS2 `UCRT64\bin` directory in its `PATH`, which otherwise causes missing-runtime errors such as `libstdc++-6.dll`, `libgcc_s_seh-1.dll`, or `libwinpthread-1.dll`.

The executable will be `Breeze5.exe` in the project directory. You can also build manually with:

```bash
make windows-static -j"$(nproc)" TARGET=Breeze5.exe AVX2=1
make TARGET=Breeze5.exe AVX2=1 test
```

To inspect the remaining imported DLL names, run:

```bash
objdump -p Breeze5.exe | grep 'DLL Name'
```

Windows system libraries (including the Universal C Runtime) are still provided by Windows and are not bundled into the engine. The automated build and inference tests run for this package were executed on Linux; the included UCRT64 build script is provided for your Windows/MSYS2 environment, but a Windows executable was not built or tested in this session.

### Linux / WSL

```bash
make -j"$(nproc)" TARGET=Breeze5
make TARGET=Breeze5 test
```

### CMake (optional)

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build --config Release -j
ctest --test-dir build --output-on-failure
```

## Run

Start the UCI engine:

```bash
./Breeze5
```

In ChessBase, add the executable as a UCI engine. The executable communicates through standard input/output and does not print non-protocol diagnostics to standard output during normal UCI operation.

Command-line diagnostics:

```bash
./Breeze5 --perft 5
./Breeze5 --probe /path/to/nn-68752b755c93.nnue
```

The uploaded network can remain outside the source ZIP. Its observed SHA-256 is:

```text
68752b755c93e419e714829c4aac1bbd66e9509a96cf175be28a00164b6d7f21
```

## Performance and compatibility notes

This is a functional experimental baseline, not a Stockfish-strength engine. The board is a mailbox representation rather than a bitboard/magic-bitboard implementation. Incremental NNUE accumulator stacks, tablebase probing, Chess960, and MultiPV are not included. AVX2 kernels are optional at build time. The `Hash` table persists between searches and can be cleared with the UCI `Clear Hash` button option.

The current HCE is deliberately transparent and deterministic. Its positional terms are independent code, not a pretrained model. Do not compare its playing strength to Breeze 4.5, Stockfish, or a network-enabled Breeze 5 build without running a proper match test.

## Source layout

```text
include/breeze.hpp     Engine interfaces and shared types
src/main.cpp           Entry point and command-line diagnostics
src/chess.cpp          Board, FEN, move generation, hashing, perft
src/eval.cpp           Independent handcrafted evaluator
src/nnue.cpp           SFNNv20 file loader, feature extraction, scalar inference
src/search.cpp         Search, move ordering, persistent transposition table
src/uci.cpp            UCI protocol and option handling
tests/perft_test.cpp   Perft + incremental-hash regression tests
tests/uci_smoke.sh     UCI protocol/search smoke test
tests/nnue_smoke.sh    SFNNv20 network-load/eval/search smoke test
docs/                  Architecture and network-format notes
```

## Using the supplied SFNNv20 network

In ChessBase or another UCI GUI, set `EvalFile` to the full path of `nn-68752b755c93.nnue`. A successful load automatically activates SFNNv20; the `Use NNUE` option remains available to switch back to the handcrafted evaluator or re-enable the network. If the network fails to load, Breeze explicitly falls back to HCE and reports the reason.

The first SFNNv20 implementation is C++ with AVX2-accelerated kernels when compiled for an AVX2 CPU (and scalar fallback otherwise). It still recomputes active features on each evaluation; incremental accumulators remain future work. The current NNUE evaluation output is mapped into Breeze's centipawn scale using a conventional historical 100/208 conversion; it is not identical to Stockfish's current UCI score normalization. Treat this as an experimental build, not a rated-strength release.


### SFNNv20 UCI setup

1. Add `Breeze5.exe` to ChessBase as a UCI engine.
2. Open its UCI options.
3. Set `EvalFile` to the full path of your `nn-68752b755c93.nnue`.
4. Restart analysis or start a new game. Loading the file automatically activates SFNNv20.

The engine reports `SFNNv20 network loaded successfully` and `evaluation mode: SFNNv20`. The network file is not bundled in this ZIP; use the exact file you already have. You can manually set `Use NNUE` to `false` to compare against HCE.

To run the network integration smoke test from Linux/WSL or MSYS2 after building:

```bash
make nnue-test NETWORK=/path/to/nn-68752b755c93.nnue
```

### Current limitations

The first implementation is portable scalar C++ and rebuilds the active feature accumulators from the current board on each evaluation. It does not yet have make/unmake incremental accumulator stacks, AVX2/AVX-VNNI kernels, or bit-for-bit reference-parity certification. The NNUE head returns the network's internal fixed-point value; Breeze maps it to its HCE-style 100-cp-per-pawn convention using the historical 100/208 ratio. This is an approximate display/search scale, not the current Stockfish win-rate-model normalization.


## AVX2 performance repair

This repair adds AVX2 kernels for sparse feature-transformer accumulation and the dense 1024→32→32→1 layers, and reuses per-thread active-feature buffers to avoid repeated heap allocations. It does **not** yet implement incremental feature accumulators; features are still regenerated for each evaluation. On an AVX2 CPU, the MSYS2 UCRT64 build script enables `-mavx2`. For a portable build use `make portable`, or set `AVX2=0`.

AVX2 builds are intended only for CPUs that support AVX2. The repair must be benchmarked with the actual SFNNv20 network file on the target computer; no NPS gain is promised without that measurement, and scalar-versus-AVX2 evaluation parity should be checked before strength testing.
