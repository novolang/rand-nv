# rand-nv

Random numbers for novo-lang, from two sources kept apart on purpose.

The operating system's cryptographically secure pseudorandom number generator
(CSPRNG) is for anything an attacker would gain by predicting: a key, a token, a
nonce, a password salt. A seedable
[xoshiro256\*\*](https://prng.di.unimi.it/) is for anything that has to produce
the same numbers twice: a test, a simulation, a shuffled fixture.

Choosing between them is one question, and this package keeps it in front of you
rather than behind a default.

> This repository is where rand-nv is developed and published. It began inside
> the novo-lang monorepo and moved out with its history. Issues and pull
> requests belong here.

## The two sources

| | `OsRandom` | `Rng` |
| --- | --- | --- |
| Algorithm | the operating system's CSPRNG | xoshiro256\*\* |
| Seedable | no | yes |
| Predictable from its output | no | yes — a handful of draws give the rest |
| Can fail | yes, it is a file | no |
| Cost | a `read` per draw | a few instructions per draw |
| Use for | keys, tokens, nonces, salts | tests, simulations, fixtures, sampling |

If anyone gains anything by predicting the number, use the first column.
Everything else is the second, seeded from a constant so a failure can be
replayed.

A CSPRNG is a generator whose output cannot be distinguished from random by any
practical computation, and from which past and future output cannot be
recovered even given some of it. xoshiro256\*\* has none of those properties. It
is fast, it passes statistical tests, and a handful of its outputs reveal its
whole state.

## Install

```
novo pkg add rand-nv
```

## Example

```novo
use rng

fn main() [io, fs, rand]
    // A token nobody can guess.
    match rng.os_bytes(32)
        Some(token) => println(bytes.to_hex(token))
        None        => eprintln("no /dev/urandom on this platform")

    // A run that can be replayed.
    var r = rng.seeded(20260905)
    println("${r.range(1, 7)}")
```

## What the package contains

One module.

| Module | Contents |
| --- | --- |
| `rng` | Both sources. `OsRandom` with `os_open`, its methods `bytes`, `fill`, `u64`, `range` and `close`, and the one-shot forms `os_bytes` and `os_u64`. `Rng` with `seeded` and `from_os`, and its methods `u64`, `positive`, `range`, `fill` and `bytes`. |

Both sources answer the same four questions — a 64-bit value, a value in a
range, a buffer filled, a buffer allocated — so switching between them changes
the construction and nothing else.

The full reference is on
[the package's page](https://novo-lang.org/packages/rand-nv), generated from
these sources: every `pub` declaration with its signature, its declared effects
and the comment block above it. A table of names here would be a second
original, and the second original is the one that goes stale.

## How to choose an entry point

**Call `rng.os_bytes` or `rng.os_u64` for a single secure draw.** Each opens the
device, reads, and closes it.

**Open an `OsRandom` and hold it when you draw in a loop.** Call `fill` on the
handle, then `close` when done. A handle that outlives its use holds a file
descriptor, and a process has a finite number of those.

**Call `rng.seeded(n)` for a reproducible stream**, or `rng.from_os()` to seed
one from the operating system when the stream should differ per run but need not
resist prediction.

## The rules a user needs

1. **`u64` answers the bit pattern of an unsigned 64-bit value**, so half the
   draws are negative `Int`s. That is the value, not a fault. `range` and `fill`
   never hand one back, and `positive` masks off the sign bit.
2. **`range(lo, hi)` is half-open and unbiased.** A plain `draw % span` favours
   the low residues whenever the span does not divide the draw's range. The bias
   is invisible for a die and total for a span near the top of the range. A draw
   landing in the ragged tail above the last whole multiple of the span is
   discarded and another taken. The expected cost is a fraction of one extra
   draw for any span a caller actually asks for.
3. **An empty or reversed range answers `lo`** rather than panicking, so a
   caller that computed an empty range sees the mistake where the range was
   computed.
4. **`fill` takes a cursor and allocates nothing.** The offset lives in the
   cursor rather than in a parameter, because a `Bytes` passed as a parameter is
   borrowed: the caller still holds it, and a write through it would land in a
   copy the caller never sees. `bytes(n)` is the allocating convenience on top
   of `fill`.

   ```novo
   use rng
   use std.bytes

   var c = bytes.cursor_be(bytes.zeros(64))
   var r = rng.seeded(1)
   r.fill(c, 64)
   ```

5. **`Rng` is not a CSPRNG and no amount of seeding makes it one.** If the
   number matters to someone else, use `OsRandom`.

## Where each source runs

The operating system source reads `/dev/urandom`, so `OsRandom` works where that
file does: Linux, macOS and the BSDs. On a platform without it, `os_open`
answers `None` and every one-shot form answers `None`. There is no fallback to a
weaker source: a caller who asked for the secure generator is told it is absent
rather than handed something else.

`Rng` has no such limit. It is arithmetic on `Int` with no allocation, no system
call and no operating system, so it runs wherever the language does. A target
with a hardware entropy peripheral should read that peripheral and pass the
value to `seeded`.

## What is not included

- **Distributions beyond the uniform one.** No normal, no exponential, no
  Zipfian.
- **Shuffling and weighted choice.** Both belong in a package that takes a
  generator, rather than in the one that is a generator.
- **A fallback when `/dev/urandom` is absent.** See "Where each source runs".

## Related packages

Several packages on the registry take their randomness as an argument rather
than drawing it, so that they can run where there is no operating system. This
is the package that supplies it: [x25519-nv](https://novo-lang.org/packages/x25519-nv)
and [ed25519-nv](https://novo-lang.org/packages/ed25519-nv) want 32 bytes for a
key pair, [p256-nv](https://novo-lang.org/packages/p256-nv) the same, and
[argon2-nv](https://novo-lang.org/packages/argon2-nv) and
[scrypt-nv](https://novo-lang.org/packages/scrypt-nv) want a salt.

## Tests

```
novo test tests/rng_tests.nv
```

The external vector is splitmix64's published first output for seed 0, which
pins the seeding routine to the one xoshiro's authors specify.

The rest asserts what a caller relies on: the same seed replays exactly,
neighbouring seeds do not, every output bit moves, `range` stays in bounds and
reaches every value in them, `fill` writes exactly what it was asked for and
touches nothing past it, and the operating system's device is there and gives
different bytes each time.

## Implementation status

Everything listed here is implemented and passing.

| Item | Implemented |
| --- | --- |
| `rng.OsRandom`, `rng.os_open`, `.close` | yes |
| `OsRandom.bytes`, `.fill`, `.u64`, `.range` | yes |
| `rng.os_bytes`, `rng.os_u64` | yes |
| `rng.Rng`, `rng.seeded`, `rng.from_os` | yes |
| `Rng.u64`, `.positive`, `.range`, `.fill`, `.bytes` | yes |

## Licence

Apache-2.0. See `LICENSE`.
