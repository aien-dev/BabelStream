# omega-stream

This is an implementation of BabelStream in [Omega](https://github.com/aien-dev/omega-mv).
The kernels, the timing loop, `check_solution`, option parsing and all output are written in Omega
(`fmt.om`, `babelstream.om`). The seeded compiler `omc` builds them into a static AArch64 Linux executable.
There is no C, C++ or runtime library in the binary.

This is a serial CPU model. It mirrors `src/main.cpp` together with `src/serial`:

* the same six kernels: Copy, Mul, Add, Triad, Dot and Nstream;
* the same start values (a = 0.1, b = 0.2, c = 0.0, scalar = 0.4);
* the same Classic and Isolated run orders;
* the same rule that the first timing is dropped;
* the same `check_solution` tolerances: epsilon × 100, and epsilon × 10^7 for Dot;
* the same options and the same table and CSV output.

### Build and run

`omc` runs on AArch64 Linux only (for example GitHub's `ubuntu-24.04-arm` runner, AWS Graviton,
or NVIDIA Grace). Builds need `make`, `curl` and `sha256sum`.

```shell
cd src/omega
make                      # fetches omc from aien-dev/omega-mv at commit e5b32567, checks its sha256, builds ./omega-stream
./omega-stream --arraysize 2048
./omega-stream --float --only All
make check                # short runs; each must pass check_solution (exit status 0)
make negative-control     # a copy with a broken Triad kernel must fail check_solution (exit status 1)
```

If you already have the pinned `omc`, pass its path with `make OMC=/path/to/omc`. Its sha256 is still checked.
The compiler pin is set in the `Makefile` by `OMEGA_MV_COMMIT` and `OMC_SHA256`.

### Options

These are the same as the C++ models: `--list`, `--device`, `-s/--arraysize`, `-n/--numtimes`, `--float`,
`-o/--only`, `--print-names`, `--order Classic|Isolated`, `--csv`, the six unit options, `--silence-errors` and `-h/--help`.
For the same arguments, the output (apart from timings) and the validation messages are byte-for-byte
the same as `serial-stream`. Numbers are printed with exact decimal rounding, as `printf` does.

### Limits

* Serial, one CPU thread, AArch64 Linux only. There is no GPU path.
* Omega has no heap yet: arrays are fixed global pools of at most 2^24 elements, 1 GiB in total.
  `--arraysize` is therefore at most 33554432 (the default) for double and 16777216 for `--float`.
  Larger sizes fail with "Invalid array size."
* `--numtimes` is at most 1000000, because Omega counted loops are bounded.
* Every array access is bounds-checked, and floating-point values travel through general registers
  (no SIMD). Expect lower bandwidth than the C++ serial model built with an optimising compiler.
