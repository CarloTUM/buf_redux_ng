# buf_redux_ng

[![crates.io](https://img.shields.io/crates/v/buf_redux_ng.svg)](https://crates.io/crates/buf_redux_ng)
[![docs.rs](https://docs.rs/buf_redux_ng/badge.svg)](https://docs.rs/buf_redux_ng)
[![License: MIT OR Apache-2.0](https://img.shields.io/crates/l/buf_redux_ng.svg)](#license)
[![CI](https://github.com/CarloTUM/buf_redux_ng/actions/workflows/ci.yml/badge.svg)](https://github.com/CarloTUM/buf_redux_ng/actions/workflows/ci.yml)

Drop-in replacements for `std::io::BufReader`, `BufWriter` and `LineWriter` that give you
direct control over the buffer and pluggable buffering policies.

`buf_redux_ng` is a maintained fork of [`buf_redux`](https://crates.io/crates/buf_redux) 0.8.4,
whose repository is archived
([RUSTSEC-2023-0028](https://rustsec.org/advisories/RUSTSEC-2023-0028.html)).
The API matches `buf_redux` 0.8.4 exactly. The only code change fixes the warning
*"the following packages contain code that will be rejected by a future version of Rust:
buf_redux v0.8.4"* (lint `trailing_semicolon_in_macro`).

## Migrating from `buf_redux`

Change one line in `Cargo.toml`. Your `use buf_redux::...` imports keep working:

```toml
[dependencies]
buf_redux = { package = "buf_redux_ng", version = "0.8", default-features = false }
```

You can also depend on it under its own name and import from `buf_redux_ng::` instead:

```toml
[dependencies]
buf_redux_ng = { version = "0.8", default-features = false }
```

**Getting `buf_redux` through another crate?** A `[patch]` section can't do this swap,
because Cargo only patches a package with another package of the same name. The crate that
depends on `buf_redux` has to switch. For `multipart`, use
[`multipart_ng`](https://crates.io/crates/multipart_ng), which already depends on `buf_redux_ng`.

## Feature flags

| Feature       | Default | Description |
|---------------|---------|-------------|
| `slice-deque` | yes     | Ringbuffer constructors (`new_ringbuf()`, `with_capacity_ringbuf()`) backed by the [`slice-deque`](https://crates.io/crates/slice-deque) crate. Unix and Windows only. |
| `nightly`     | no      | Legacy. Relied on unstable APIs that have since been removed from Rust, so it does not build on current nightly compilers. Leave it off. |

> **Disable default features unless you need ringbuffers.** `slice-deque` is unmaintained
> ([RUSTSEC-2020-0158](https://rustsec.org/advisories/RUSTSEC-2020-0158.html)) and has an
> unpatched advisory ([RUSTSEC-2021-0047](https://rustsec.org/advisories/RUSTSEC-2021-0047.html),
> double drop in `SliceDeque::drain_filter`). `buf_redux_ng` never calls `drain_filter`, but
> `cargo audit` and `cargo deny` still flag the dependency. The feature stays on by default
> for compatibility with `buf_redux` 0.8.4.

## Usage

Swap the import of the type you want to replace:

```diff
- use std::io::BufReader;
+ use buf_redux::BufReader;
```

```diff
- use std::io::BufWriter;
+ use buf_redux::BufWriter;
```

```diff
- use std::io::LineWriter;
+ use buf_redux::LineWriter;
```

These types keep the method names, signatures and implemented traits of their `std` counterparts.
(If you didn't use the rename shown above, write `buf_redux_ng::` instead of `buf_redux::`.)

### More direct control

All types provide methods to:
* increase the capacity of the buffer
* get the number of buffered bytes and the total capacity
* consume the wrapper without losing data

`BufReader` provides methods to:
* access the buffer through a `&`-reference without doing any I/O
* force unconditional reads into the buffer
* get a `Read` adapter that drains the buffer and then reads from the inner reader directly
* shift bytes to the beginning of the buffer to make room for more reads
* get the inner reader together with a trimmed buffer of the remaining data

`BufWriter` and `LineWriter` provide methods to:
* flush the buffer and unwrap the inner writer unconditionally

### Customizable buffering policies

The types in the `policy` module let you adjust buffering to your use case:

* Implement `ReaderPolicy` to customize `BufReader`, or use an existing policy such as
  `MinBuffered`, which keeps a minimum number of bytes in the buffer (until the inner reader
  is exhausted).
* Implement `WriterPolicy` to customize `BufWriter`, or use an existing policy such as
  `FlushOn`, which flushes when a given byte appears in the buffer (`LineWriter` is built on it).

### Example: `MinBuffered`

`MinBuffered` guarantees a minimum amount of lookahead, which is useful for parsers:

```rust
use buf_redux::BufReader;
use buf_redux::policy::MinBuffered;
use std::io::{BufRead, Cursor};

let data = (1 .. 16).collect::<Vec<u8>>();

// normally you should use `BufReader::new()` or give a capacity of several KiB or more
let mut reader = BufReader::with_capacity(8, Cursor::new(data))
    // always at least 4 bytes in the buffer (or until the source is empty)
    .set_policy(MinBuffered(4));

// first buffer fill, same as `std::io::BufReader`
assert_eq!(reader.fill_buf().unwrap(), &[1, 2, 3, 4, 5, 6, 7, 8]);
reader.consume(3);

// enough data in the buffer, another read isn't done yet
assert_eq!(reader.fill_buf().unwrap(), &[4, 5, 6, 7, 8]);
reader.consume(4);

// `std::io::BufReader` would return `&[8]`
assert_eq!(reader.fill_buf().unwrap(), &[8, 9, 10, 11, 12, 13, 14, 15]);
reader.consume(5);

// no data left in the reader
assert_eq!(reader.fill_buf().unwrap(), &[13, 14, 15]);
```

### Making room and ringbuffers

Policies like `MinBuffered` consume bytes without emptying the buffer, so the free space ends up
at the front of the buffer and there is no room left to read into. When only a little data is
buffered, call `.make_room()` to move it to the start. `MinBuffered` does this automatically.

With the `slice-deque` feature you can use a ringbuffer instead: construct the type with
`::new_ringbuf()` or `::with_capacity_ringbuf()` instead of `::new()` or `::with_capacity()`.
With a ringbuffer, consuming or flushing bytes immediately frees room at the end. The caveats:

* Ringbuffers need virtual memory support, so they only work on full operating systems
  (Windows, Linux, macOS, the BSDs).
* Capacities are rounded up to a multiple of the platform's allocation granularity, usually the
  page size. On Windows that is **64 KiB**, much more than the 8 KiB default of a normal buffer.
* The buffer reserves twice its capacity in virtual address space. Physical memory use is the
  same as a normal buffer, but this can matter on 32-bit targets.
* `slice-deque` brings in the advisories described in [Feature flags](#feature-flags).

## Scope of this fork

- The public API of `buf_redux` 0.8.4 stays as it is. The 0.8.x line gets no new features and
  no breaking changes.
- Compatibility fixes for new Rust releases and security fixes are in scope. Please report
  problems on the [issue tracker](https://github.com/CarloTUM/buf_redux_ng/issues).
- Cargo produces no future-incompatibility report for this crate on current stable Rust
  (checked with 1.99).

## Credits

`buf_redux` was written by Austin Bonander
([abonander/buf_redux](https://github.com/abonander/buf_redux), archived).
All credit for the design and implementation goes to them.

## License

Licensed under either of

 * Apache License, Version 2.0 ([LICENSE-APACHE](LICENSE-APACHE) or http://www.apache.org/licenses/LICENSE-2.0)
 * MIT license ([LICENSE-MIT](LICENSE-MIT) or http://opensource.org/licenses/MIT)

at your option.

### Contribution

Unless you explicitly state otherwise, any contribution intentionally submitted
for inclusion in the work by you, as defined in the Apache-2.0 license, shall be dual licensed as above, without any
additional terms or conditions.
