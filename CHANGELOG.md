# Changelog

Newest first.  Below `1.0.0` a breaking change bumps the **minor**
number and a compatible one the **patch**; see [Version numbers in the
Orbit package registry](https://novo-lang.org/docs/registry/semver.html).

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
