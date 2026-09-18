# rand-nv

A random number generator answers either numbers nobody can predict or
numbers a caller can reproduce exactly. No generator does both. This
package brings both to novo-lang as two sources that are named apart
and never substituted for one another. The first is the operating
system's cryptographically secure generator, read from `/dev/urandom`.
The second is
[xoshiro256\*\*](https://prng.di.unimi.it/), a seedable generator
published with its reference implementation by David Blackman and
Sebastiano Vigna.

## What the two sources are

A cryptographically secure pseudorandom number generator (CSPRNG) is a
generator whose output no practical computation can tell apart from
random. An observer holding part of the output can recover neither the
rest of it nor what came before. The operating system maintains one and
hands its bytes out through the character device `/dev/urandom`. That is
the source for anything an attacker would gain by predicting. A key, a
token, a nonce and a password salt are four such things.

xoshiro256\*\* is a pseudorandom number generator whose whole state is
256 bits, held as four 64-bit words. It is fast, it passes the
statistical test suites its authors run, and it is not cryptographically
secure. An observer who has seen a handful of its outputs can compute
the rest of the stream. It is here because a test, a simulation and a
shuffled fixture all need the same numbers twice, which is exactly what
a secure generator refuses to give.

A generator's **seed** is the value its state is built from. The same
seed gives the same stream. This package builds the state with
splitmix64, the seeding routine xoshiro's authors specify. splitmix64
spreads one seed value across all 256 bits, so neighbouring seeds open
with unrelated streams rather than with nearly the same one.

| | `OsRandom` | `Rng` |
| --- | --- | --- |
| Algorithm | the operating system's CSPRNG | xoshiro256\*\* |
| State | a file descriptor | 256 bits, in four 64-bit words |
| Seedable | no | yes |
| Predictable from its own output | no | yes, from a handful of draws |
| Can fail | yes, it is a file | no |
| Cost of one 64-bit draw | one `read` | a few instructions |
| Used for | keys, tokens, nonces, salts | tests, simulations, fixtures, sampling |

Choosing between the two is one question. Does anyone gain anything by
predicting this number? If the answer is yes, the source is `OsRandom`.
If the answer is no, the source is `Rng`, seeded from a constant so that
a failure can be replayed.

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
        None        => println("no /dev/urandom on this platform")

    // A run that can be replayed.
    var r = rng.seeded(20260905)
    println("${r.range(1, 7)}")
```

## What the package contains

| Module | Contents |
| --- | --- |
| `rng` | Both sources. `OsRandom` with `os_open`, its methods `bytes`, `fill`, `u64`, `range` and `close`, and the one-shot forms `os_bytes` and `os_u64`. `Rng` with `seeded` and `from_os`, and its methods `u64`, `positive`, `range`, `fill` and `bytes`. |

Both sources answer the same four questions. A 64-bit value, a value in
a range, a caller's buffer filled, and a fresh buffer of bytes. Moving a
program from one source to the other therefore changes the construction
and nothing else.

The API reference is on
[the package's page](https://novo-lang.org/packages/rand-nv). `novo doc`
generates it from these sources: every `pub` declaration with its
signature, its effect row and the comment block written above it.

## How to choose an entry point

**`rng.os_bytes` and `rng.os_u64` take a single secure draw.** Each
opens the device, reads it and closes it again. This is the form for a
token, a salt or a nonce drawn once.

**An `OsRandom` held open is the form for a loop.** `rng.os_open`
answers one. Draw with `fill` or `bytes`, then `close` when the work is
done, because a handle owns a file descriptor and a process has a finite
number of those.

**`rng.seeded(n)` answers a stream fixed by `n`.** The same seed gives
the same numbers on every machine and every backend, which is what lets
a failing run be repeated.

**`rng.from_os()` answers a stream that differs per process.** The seed
comes from the operating system. What follows that seed is still
predictable from its own output.

## The rules a user needs

1. **`Rng` is not cryptographically secure, and no seed makes it one.**
   An observer who has seen a handful of draws can compute the rest of
   the stream. Nothing that anyone gains by predicting may come from an
   `Rng`, and that includes a key, a token, a nonce, a password salt and
   a session identifier. `OsRandom` is the source for those. `from_os`
   is not a substitute for it, because only the seed is unguessable and
   the stream that follows is not.
2. **`u64` answers the bit pattern of an unsigned 64-bit value.** Half
   the draws are therefore negative `Int`s. That is the value rather
   than a fault, because the draw is 64 bits and an `Int` is 64 bits.
   `positive` is the same draw with the top bit dropped, so 63 random
   bits. `range` and `fill` never hand a negative value back.
3. **`range(lo, hi)` is half-open and free of modulo bias.** A plain
   `draw % span` favours the low residues whenever the span does not
   divide the draw's range. The bias is invisible for a die and total
   for a span near the top of the range. A draw landing in the ragged
   tail above the last whole multiple of the span is discarded and
   another taken. The expected cost is a fraction of one extra draw for
   any span a caller actually asks for.
4. **An empty or reversed range answers `lo`.** It does not panic, so a
   caller that computed an empty range sees the mistake where the range
   was computed.
5. **`fill` takes a cursor and allocates nothing.** The offset lives in
   the cursor rather than in a parameter, because a `Bytes` passed as a
   parameter is borrowed. The caller still holds it, and a write through
   it would land in a copy the caller never sees. `bytes(n)` is the
   allocating convenience on top of `fill`.

   ```novo
   use rng
   use std.bytes

   var c = bytes.cursor_be(bytes.zeros(64))
   var r = rng.seeded(1)
   r.fill(c, 64)
   ```

6. **`OsRandom` needs `/dev/urandom`, and there is no fallback.** The
   device exists on Linux, macOS and the BSDs. Where it is absent,
   `os_open` answers `None` and so does every one-shot form. A caller
   who asked for the secure generator is told it is absent rather than
   handed a weaker source.
7. **`Rng` needs no operating system.** It is arithmetic on `Int` with
   no allocation and no system call, so it runs wherever the language
   does. A target with a hardware entropy peripheral reads that
   peripheral and passes the value to `seeded`.

## What is not included

- **Distributions beyond the uniform one.** There is no normal, no
  exponential and no Zipfian.
- **Shuffling and weighted choice.** Both belong in a package that takes
  a generator, rather than in the one that is a generator.
- **A fallback when `/dev/urandom` is absent.** A caller who asked for
  unpredictable bytes would be handed predictable ones with nothing
  saying so.
- **The jump functions.** The reference implementation publishes `jump`
  and `long_jump`, which advance a stream far enough for a parallel run
  to take a part of it nobody else uses. Neither is here, so each worker
  in a parallel run takes its own seed.
- **Erasing a buffer after use.** A buffer this package fills keeps its
  bytes until the caller overwrites it or drops it.

## Related packages

Several packages on the registry take their randomness as an argument
rather than drawing it, so that they can run where there is no operating
system. This is the package that supplies it.
[x25519-nv](https://novo-lang.org/packages/x25519-nv) and
[ed25519-nv](https://novo-lang.org/packages/ed25519-nv) want 32 bytes
for a key pair, [p256-nv](https://novo-lang.org/packages/p256-nv) wants
the same, and [argon2-nv](https://novo-lang.org/packages/argon2-nv) and
[scrypt-nv](https://novo-lang.org/packages/scrypt-nv) want a salt.

## Tests

```
novo test tests/rng_tests.nv
```

The one external vector is splitmix64's published first output for seed
0, which pins the seeding routine to the one xoshiro's authors specify
rather than to this package's idea of it.

The rest of the suite asserts what a caller relies on. The same seed
replays exactly and neighbouring seeds do not, every one of the 64
output bits is seen both set and clear, `range` stays inside its bounds
and reaches every value in them, `fill` writes exactly what it was asked
for and touches nothing past it, and the operating system's device is
there and answers different bytes each time.

Every example in a documentation comment is compiled by `novo doc` and
run by `novo test`, so an example that stopped compiling or stopped
running is a failing test. The values beside those examples are printed
rather than compared, and the suite above is what pins the stream.

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
