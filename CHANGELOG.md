# Changelog

Newest first.  Below `1.0.0` a breaking change bumps the minor number
and a compatible one the patch number.  See [Version numbers in the
Orbit package registry](https://novo-lang.org/docs/registry/semver.html).

## 0.1.6 — 2026-09-18

The documentation and comments in plain prose; no signature changed.
The README's first example calls `println` where it called a function
that does not exist, so every example in the package now compiles.

## 0.1.5 — 2026-09-12

The README written to the package README style guide
(docs/writing-a-readme.md); no signature changed.

## 0.1.4 — 2026-09-08

- The manifest declares the package's layer as `host`.  The public API
  reaches the host for its random bytes, which is the `[fs, rand]` on
  those signatures, and `novo pkg publish` checks the code against that
  layer.  No code changed.  The layers are described under Design in the
  [publishing guide](https://novo-lang.org/docs/publishing.html#design).
- The sources are in the canonical form `novo fmt` prints.  Spacing and
  alignment only; no code changed.

## 0.1.3

The reference is generated from the code, and the examples in it are
compiled.  No code changed, and the seeded stream is the stream 0.1.2
produced.

- Every `pub` item carries the comment block directly above its
  declaration, and the first sentence of each is the summary a reader
  meets before opening anything.  The methods on `OsRandom` and `Rng`
  carry their own.  `novo doc` turns the lot into
  [the package's page](https://novo-lang.org/packages/rand-nv).
- Nine worked examples, and they run.  The seeded generator's examples
  print exact values: the first two draws from `seeded(1)`, three dice
  rolls, and twelve bytes in hexadecimal.  The three that read
  `/dev/urandom` write their own `main`, because drawing needs more than
  `[io]`.  A fenced `novo` block in a documentation comment is compiled
  by `novo doc` and run by `novo test src/rng.nv`.

## 0.1.2

No code changed.  Every signature and every byte on the wire is what
0.1.1 shipped.

- `LICENSE` ships with the package.  It is on the publish allow-list, so
  the archive now carries the Apache-2.0 text rather than only naming it
  in the manifest.

## 0.1.1

The streams are the streams they were.  The splitmix64 seeding vector,
the uniformity check and the bit-balance check all still pass, which is
what says so.

- The generator's arithmetic is written with the operators.
  xoshiro256**'s state advance is five `^=` lines that a reader can set
  beside the published algorithm, where it was five `bits.xor` calls
  naming each state word twice.  `rotl` is `(x << k) | (x >>> back)`,
  splitmix64's mixing rounds are `z ^ (z >>> 30)`, and the byte tail of
  a `fill` steps with `word >>>= 8`.  That is twenty-four `bits.*` calls
  gone from the module and the suite, and `std.bits` is no longer
  imported.
- The test module sits under `tests/` rather than under `src/`.  A
  package's `src/` ships whole and a consumer compiles every module in
  it, so the suite lives where it is not published.  Run it with
  `novo test tests/rng_tests.nv`.

## 0.1.0

The two sources and their calls: `os_open`, `os_bytes`, `os_u64`,
`seeded`, `from_os`, and the `OsRandom` and `Rng` values, each answering
`u64`, `range`, `fill` and `bytes`.

- Two sources, kept apart.  The operating system's CSPRNG is for
  anything an attacker would gain by predicting, and a seedable
  xoshiro256** is for anything that has to repeat.  The names say which
  is which, and neither is a default.
- `range` is uniform.  A draw landing above the last whole multiple of
  the span is discarded rather than folded, so there is no modulo bias.
  An empty or reversed range answers its low bound rather than
  panicking.
- `fill` allocates nothing.  It writes at a cursor the caller owns, and
  `bytes` is the allocating convenience on top of it.
- Seeding is splitmix64, the routine xoshiro's authors specify, and its
  published first output for seed 0 is asserted.  Neighbouring seeds
  therefore open with unrelated streams.
- The operating system source is `/dev/urandom`.  Where that file is
  absent every entry point into this source answers `None` rather than
  falling back to something weaker.  `Rng` needs no operating system at
  all.
