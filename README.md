# [simple-base64](https://crates.io/crates/simple-base64)

[![](https://img.shields.io/crates/v/simple-base64.svg)](https://crates.io/crates/simple-base64) [![Docs](https://docs.rs/simple-base64/badge.svg)](https://docs.rs/simple-base64) [![unsafe forbidden](https://img.shields.io/badge/unsafe-forbidden-success.svg)](https://github.com/rust-secure-code/safety-dance/)

This library is a fork of a base64 library with aim to be easy to use and to have an API that is easy to figure out
when looking at the documentation.

See the [docs](https://docs.rs/simple-base64) for the details.

## Usage

### Standard encoding/decoding

Standard encoding and decoding can be achieved by using the crate level `encode` and `decode` functions:

```rust
use simple_base64::{encode, decode};

let s = String::from("a string");
let encoded = encode(s.clone());
let decoded = decode(encoded).unwrap();
// decode returns a vector of bytes, thus we have to convert it to String
let decoded = String::from_utf8(decoded).unwrap();
assert_eq!(s, decoded);
```

### Additional engines

If you need a different engine you can also use it with `encode_engine` and `decode_engine` methods:

```rust
use simple_base64::{encode_engine, decode_engine};
use simple_base64::prelude::BASE64_STANDARD_NO_PAD as NO_PAD;

let s = String::from("a string");
let encoded = encode_engine(s.clone(), &NO_PAD);
let decoded = decode_engine(encoded, &NO_PAD).unwrap();
// decode returns a vector of bytes, thus we have to convert it to String
let decoded = String::from_utf8(decoded).unwrap();
assert_eq!(s, decoded);
```

## FAQ

### I need to decode base64 with whitespace/null bytes/other random things interspersed in it. What should I do?

Remove non-base64 characters from your input before decoding.

If you have a `Vec` of base64, [retain](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.retain) can be used to
strip out whatever you need removed.

If you have a `Read` (e.g. reading a file or network socket), there are various approaches.

- Use [iter_read](https://crates.io/crates/iter-read) together with `Read`'s `bytes()` to filter out unwanted bytes.
- Implement `Read` with a `read()` impl that delegates to your actual `Read`, and then drops any bytes you don't want.

### I need to line-wrap base64, e.g. for MIME/PEM.

[line-wrap](https://crates.io/crates/line-wrap) does just that.

### I want canonical base64 encoding/decoding.

First, don't do this. You should no more expect Base64 to be canonical than you should expect compression algorithms to
produce canonical output across all usage in the wild (hint: they don't).
However, [people are drawn to their own destruction like moths to a flame](https://eprint.iacr.org/2022/361), so here we
are.

There are two opportunities for non-canonical encoding (and thus, detection of the same during decoding): the final bits
of the last encoded token in two or three token suffixes, and the `=` token used to inflate the suffix to a full four
tokens.

The trailing bits issue is unavoidable: with 6 bits available in each encoded token, 1 input byte takes 2 tokens,
with the second one having some bits unused. Same for two input bytes: 16 bits, but 3 tokens have 18 bits. Unless we
decide to stop shipping whole bytes around, we're stuck with those extra bits that a sneaky or buggy encoder might set
to 1 instead of 0.

The `=` pad bytes, on the other hand, are entirely a self-own by the Base64 standard. They do not affect decoding other
than to provide an opportunity to say "that padding is incorrect". Exabytes of storage and transfer have no doubt been
wasted on pointless `=` bytes. Somehow we all seem to be quite comfortable with, say, hex-encoded data just stopping
when it's done rather than requiring a confirmation that the author of the encoder could count to four. Anyway, there
are two ways to make pad bytes predictable: require canonical padding to the next multiple of four bytes as per the RFC,
or, if you control all producers and consumers, save a few bytes by requiring no padding (especially applicable to the
url-safe alphabet).

All `Engine` implementations must at a minimum support treating non-canonical padding of both types as an error, and
optionally may allow other behaviors.

## Rust version compatibility

The minimum supported Rust version is 1.48.0.

# Contributing

Contributions are very welcome. However, because this library is used widely, and in security-sensitive contexts, all
PRs will be carefully scrutinized. Beyond that, this sort of low level library simply needs to be 100% correct. Nobody
wants to chase bugs in encoding of any sort.

All this means that it takes me a fair amount of time to review each PR, so it might take quite a while to carve out the
free time to give each PR the attention it deserves. I will get to everyone eventually!

## Developing

Benchmarks are in `benches/`.

```bash
cargo bench
```

## no_std

This crate supports no_std. By default the crate targets std via the `std` feature. You can deactivate
the `default-features` to target `core` instead. In that case you lose out on all the functionality revolving
around `std::io`, `std::error::Error`, and heap allocations. There is an additional `alloc` feature that you can activate
to bring back the support for heap allocations.

## Profiling

On Linux, you can use [perf](https://perf.wiki.kernel.org/index.php/Main_Page) for profiling. Then compile the
benchmarks with `cargo bench --no-run`.

Run the benchmark binary with `perf` (shown here filtering to one particular benchmark, which will make the results
easier to read). `perf` is only available to the root user on most systems as it fiddles with event counters in your
CPU, so use `sudo`. We need to run the actual benchmark binary, hence the path into `target`. You can see the actual
full path with `cargo bench -v`; it will print out the commands it runs. If you use the exact path
that `bench` outputs, make sure you get the one that's for the benchmarks, not the tests. You may also want
to `cargo clean` so you have only one `benchmarks-` binary (they tend to accumulate).

```bash
sudo perf record target/release/deps/benchmarks-* --bench decode_10mib_reuse
```

Then analyze the results, again with perf:

```bash
sudo perf annotate -l
```

You'll see a bunch of interleaved rust source and assembly like this. The section with `lib.rs:327` is telling us that
4.02% of samples saw the `movzbl` aka bit shift as the active instruction. However, this percentage is not as exact as
it seems due to a phenomenon called *skid*. Basically, a consequence of how fancy modern CPUs are is that this sort of
instruction profiling is inherently inaccurate, especially in branch-heavy code.

```text
 lib.rs:322    0.70 :     10698:       mov    %rdi,%rax
    2.82 :        1069b:       shr    $0x38,%rax
         :                  if morsel == decode_tables::INVALID_VALUE {
         :                      bad_byte_index = input_index;
         :                      break;
         :                  };
         :                  accum = (morsel as u64) << 58;
 lib.rs:327    4.02 :     1069f:       movzbl (%r9,%rax,1),%r15d
         :              // fast loop of 8 bytes at a time
         :              while input_index < length_of_full_chunks {
         :                  let mut accum: u64;
         :
         :                  let input_chunk = BigEndian::read_u64(&input_bytes[input_index..(input_index + 8)]);
         :                  morsel = decode_table[(input_chunk >> 56) as usize];
 lib.rs:322    3.68 :     106a4:       cmp    $0xff,%r15
         :                  if morsel == decode_tables::INVALID_VALUE {
    0.00 :        106ab:       je     1090e <base64::decode_config_buf::hbf68a45fefa299c1+0x46e>
```

## Fuzzing

This uses [cargo-fuzz](https://github.com/rust-fuzz/cargo-fuzz). See `fuzz/fuzzers` for the available fuzzing scripts.
To run, use an invocation like these:

```bash
cargo +nightly fuzz run roundtrip
cargo +nightly fuzz run roundtrip_no_pad
cargo +nightly fuzz run roundtrip_random_config -- -max_len=10240
cargo +nightly fuzz run decode_random
```

## License

This project is dual-licensed under MIT and Apache 2.0.

