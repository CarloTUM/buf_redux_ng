# Changelog

## 0.8.5 (2026-11-19)

No code changes.

- Rewrote the README: migration guide, feature flags, notes on the `slice-deque` advisories.
- Crate docs explain the fork and the `package = "buf_redux_ng"` rename.
- Fixed the `MinBuffered` doctest, which still used the old crate name.
- Updated crates.io description, keywords and categories.
- Replaced Travis CI with GitHub Actions (Linux, macOS, Windows; weekly run against the
  latest stable Rust).

## 0.8.4 (2026-05-31)

Fork of `buf_redux` 0.8.4.

- Removed the trailing semicolons in the `do_read!` and `flush_amt!` macros, which triggered
  the `trailing_semicolon_in_macro` future-incompatibility warning.
