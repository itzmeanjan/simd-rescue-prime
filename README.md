# simd-rescue-prime
Portable SIMD powered Rescue Prime Hash

## Motivation

For sometime, I've been exploring Zk-STARK friendly Rescue Prime Hash function and writing following implementations, majorly for harnessing power of multi/ many core accelerator devices, such as CPU, GPU etc.

- [Data Parallel SYCL/ DPC++](https://github.com/itzmeanjan/ff-gpu/blob/9c57cb13e4b2d96a084da96d558fe3d4707bfcb7/rescue_prime.cpp)
- [Vectorized using OpenCL](https://github.com/itzmeanjan/vectorized-rescue-prime/blob/614500dd1f271e4f8badf1305c8077e2532eb510/kernel.cl#L345-L474)

This time I decided to implement Rescue Prime Hash function on 64-bit prime field `F(2^64 - 2^32 + 1)`, using _experimental_ `portable_simd` feature, which recently landed in Rust Nightly.

> More about it [here](http://github.com/rust-lang/rust/issues/86656).

> Notice, this library crate [overrides](https://github.com/itzmeanjan/simd-rescue-prime/blob/9e83eb579a6e7666ae33d2c86524d8e287e7f1ca/rust-toolchain) default Rust toolchain version to use `portable_simd` feature.

This implementation takes quite a lot of motivation from [this OpenCL kernel](https://github.com/itzmeanjan/vectorized-rescue-prime/blob/614500dd1f271e4f8badf1305c8077e2532eb510/kernel.cl), where I implemented Rescue Prime Hash using OpenCL vector intrinsics.

Resue Prime Hash state is represented using an array of 3 SIMD vectors, each with 4 lanes, where each element is of type `u64` i.e. `[Simd<u64, 4>; 3]` . Originally hash state was represented using `Simd<u64, 16>`, which I changed to current one, following [this](https://github.com/rust-lang/portable-simd/issues/215) suggestion. Here I'm applying 7 rounds during Rescue permutation, following [this](https://github.com/novifinancial/winterfell/blob/4eeb4670387f3682fa0841e09cdcbe1d43302bf3/crypto/src/hash/rescue/rp64_256/mod.rs#L27-L29). All arithmetics are performed following rules of 64-bit prime field `F(2^64 - 2^32 + 1)`. Necessary vector arithmetics can be found [here](https://github.com/itzmeanjan/simd-rescue-prime/blob/9efa3b4ecb5236556f1917c8ebf98d4f08b64aa3/src/ff.rs). Finally produced output takes first 4 elements of hash state ( i.e. first element of state array ) after whole input is absorbed into state --- output is 4-field elements wide i.e. 32-bytes.

## Usage

- Make sure you've Rust toolchain installed. Note, this crate will override your default toolchain version for using `std::simd` library, which is behind `portable_simd` feature gate.
- Run test cases

```bash
cargo test --lib
```

- Before running benchmark ensure you've installed benchmark runner abstraction

```bash
cargo install cargo-criterion
```

- Run benchmark on `hash_elements` and `merge` functions

```bash
RUSTFLAGS="-C target-cpu=native" cargo criterion --output-format verbose

# Above command just compiles code with default CPU features found in
# compilation runner machine
#
# Consider enabling CPU specific vector instructions by passing
#
# RUSTFLAGS=`-C target-feature=+{avx,avx2,avx512vl,avx512f,avx512dq,sse4.2,sse4.1,sse4a,neon}`
#
# check below benchmark result table for flags I've used
```

- Find usage example [here](https://github.com/itzmeanjan/simd-rescue-prime/tree/20bf40c/example)

## Benchmark Results

I'm using `criterion` benchmark runner for benchmarking following two Rescue Prime Hash functions

- `hash_element`
- `merge`

One thing to note about aforementioned two functions, given an input like

```python
input = [0, 1, 2, 3, 4, 5, 6, 7]
```

following assertion must not fail

```python
assert hash_elements(input, ...) == merge(input, ...)
```

In simple terms, `merge` function merges two Rescue Prime digests into single digest, which is 4 field elements wide.

> When running both [benchmarks](https://github.com/itzmeanjan/simd-rescue-prime/blob/dcdebc35762a0dffcfce3278c2b8a8f892058809/src/rescue_prime.rs#L569), I use 8 field elements wide input, that means Rescue Permutation needs to be performed only once, because [RATE_WIDTH = 8](https://github.com/itzmeanjan/simd-rescue-prime/blob/9efa3b4ecb5236556f1917c8ebf98d4f08b64aa3/src/rescue_prime.rs#L5)

> Following table denotes how long does it take to execute single round of `hash_elements`/ `merge` function invocation, on single core of specific CPU model.

FLAGS | CPU | `hash_elements` | `merge`
--- | --- | --- | ---
`RUSTFLAGS="-C target-feature=+avx2"` | Intel(R) Xeon(R) Platinum 8252C CPU @ 3.80GHz | 8.6894 us | 8.6826 us
`RUSTFLAGS="-C target-feature=+avx2"` | Intel(R) Core(TM) i5-8279U CPU @ 2.40GHz | 9.9605 us | 9.9561 us
`RUSTFLAGS="-C target-feature=+avx2"` | AMD EPYC 7R32 | 9.4342 us | 9.4297 us
`RUSTFLAGS="-C target-cpu=native"` | Intel(R) Xeon(R) Platinum 8275CL CPU @ 3.00GHz | 10.409 us | 10.336 us
`RUSTFLAGS="-C target-feature=+avx2"` | Intel(R) Xeon(R) CPU E5-2686 v4 @ 2.30GHz | 17.646 us | 17.675 us
`RUSTFLAGS="-C target-feature=+avx2"` | Intel(R) Core(TM) i3-5005U CPU @ 2.00GHz | 24.999 us | 25.024 us
`RUSTFLAGS="-C target-feature=+neon,+a57"` | Cortex-A57 | 159.02 us | 160.74 us

---

> Above benchmarks are obtained by running `$FLAGS cargo criterion --output-format verbose`, where FLAGS denote content of *FLAGS* column in above table, as applicable on target machine.

You may want to check your supported CPU features by running

```bash
lscpu # check *Flags* field
```

You should also check

```bash
rustc --print target-features | less
```

> [This](https://github.com/rust-lang/portable-simd/blob/7d91357875da59d52284d506dcb457f7f88bf6bf/.github/workflows/ci.yml#L61-L260) CI build/ test script is pretty useful.
