# Changelog

Newest first.  Below `1.0.0` a breaking change bumps the **minor**
number and a compatible one the **patch**; see [Version numbers in the
Orbit package registry](https://novo-lang.org/docs/registry/semver.html).

## 0.1.5 — 2026-09-12

### Changed — The README is rewritten in plain technical-writer prose; no signature changed.

## 0.1.4 — 2026-09-08

- **Declares its layer**: `layer = "host"` in the manifest — the public API reaches the host for its random bytes (`[fs, rand]`), and `novo pkg publish` now checks the code against that budget.  No code changed.  The layers are described under Design in the [publishing guide](https://novo-lang.org/docs/publishing.html#design).
- **Sources reformatted to the canonical form** `novo fmt` prints today (spacing and alignment only; no code changed).

## 0.1.3

Documentation: the reference is generated from the code, and the
examples in it are doctests.  No code changed — the seeded stream is
the same stream 0.1.2 produced, which the examples now pin.

- **Every `pub` item is documented under Go's rule**, the comment block
  directly above the declaration, its first sentence the summary a
  reader meets before opening anything.  The methods on `OsRandom` and
  `Rng` carry their own.  `novo doc` turns the lot into
  [the package's page](https://novo-lang.org/packages/rand-nv).
- **Nine worked examples, and they run.**  The seeded generator's
  examples print exact values — the first two draws from `seeded(1)`,
  three dice rolls, twelve bytes in hex — so the promise that a seed
  fixes a stream is now checked on every run rather than asserted.  The
  three that read `/dev/urandom` write their own `main`, since drawing
  needs more than `[io]`.  A fenced `novo` block in a documentation
  comment is compiled by `novo doc` and run by `novo test src/rng.nv`.

## 0.1.2

Developed in its own repository from this version.  `novolang/rand-nv` is
where the sources live, where CI runs and where releases are tagged;
the novo-lang monorepo no longer carries a copy.  No code changed —
every signature and every byte on the wire is what 0.1.1 shipped.

- **`LICENSE` ships with the package.**  It is on the publish
  allow-list, so the tarball now carries the Apache-2.0 text rather
  than only naming it in the manifest.

## 0.1.1

A patch: the streams are the streams they were.  The splitmix64 seeding
vector, the byte-for-byte agreement between the two legs on a seeded
stream, and the uniformity and bit-balance checks all still pass, which
is what says so.

- **The generator's arithmetic is written with the operators.**
  xoshiro256**'s state advance is five `^=` lines that a reader can set
  beside the published algorithm, where it was five `bits.xor` calls
  naming each state word twice.  `rotl` is `(x << k) | (x >>> back)`,
  splitmix64's mixing rounds are `z ^ (z >>> 30)`, and the byte tail of
  a `fill` steps with `word >>>= 8`.  Twenty-four `bits.*` calls in the
  module and the suite; `std.bits` is no longer imported.
- **The test module moved out of `src/`.**  A package's `src/` ships
  whole and a consumer compiles every module in it, so the suite is
  under `tests/` where it is not published.  Run it with
  `novo test tests/rng_tests.nv`.

## 0.1.0

First release: `os_open`, `os_bytes`, `os_u64`, `seeded`, `from_os`,
and the `OsRandom` and `Rng` values, each answering `u64`, `range`,
`fill` and `bytes`.

- **Two sources, kept apart.**  The operating system's CSPRNG for
  anything an attacker would gain by predicting, and a seedable
  xoshiro256** for anything that has to repeat.  The names say which is
  which and neither is a default.
- **`range` is uniform.**  A draw landing above the last whole multiple
  of the span is discarded rather than folded, so there is no modulo
  bias; an empty or reversed range answers its low bound rather than
  panicking.
- **`fill` allocates nothing.**  It writes at a cursor the caller owns;
  `bytes` is the allocating convenience on top.
- **Seeding is splitmix64**, the routine xoshiro's authors specify, and
  its published first output for seed 0 is asserted — so neighbouring
  seeds open with unrelated streams.
- **Host only, and it says so.**  The operating-system source is
  `/dev/urandom`; where that is absent every entry point answers `None`
  rather than falling back to something weaker.  `Rng` needs no
  operating system at all.
