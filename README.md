> Developed in the novo-lang monorepo under `orbit/rand-nv`, which is the source of truth until this package graduates out of it.  This repository is a mirror: it is where CI runs and where releases are tagged, and changes are made upstream.

# rand-nv

Random numbers from two sources, kept apart on purpose. The operating
system's cryptographically secure generator is for anything an attacker
would gain by predicting — a key, a token, a nonce, a password salt. A
seedable xoshiro256** is for anything that has to produce the same
numbers twice — a test, a simulation, a shuffled fixture. Choosing
between them is one question, and the API keeps it in front of you
rather than behind a default.

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

```
novo pkg add rand-nv
```

## The two sources

| | `OsRandom` | `Rng` |
|---|---|---|
| Algorithm | the operating system's CSPRNG | xoshiro256** |
| Seedable | no | yes |
| Predictable from its output | no | **yes** — a handful of draws give the rest |
| Fails | yes, it is a file | no |
| Costs | a `read` per draw | a few instructions per draw |
| Use for | keys, tokens, nonces, salts | tests, simulations, fixtures, sampling |

If anyone gains anything by predicting the number, it is the first
column. Everything else is the second, seeded from a constant so a
failure can be replayed.

## What it gives you

Both sources answer the same four questions, so switching between them
changes the construction and nothing else.

| Function | |
|---|---|
| `rng.os_open() -> ?OsRandom` | a handle held across many draws |
| `o.u64() -> ?Int` / `o.range(lo, hi) -> ?Int` | one draw |
| `o.fill(dst: Cursor, n) -> Bool` / `o.bytes(n) -> ?Bytes` | `n` bytes |
| `o.close()` | give the device back |
| `rng.os_bytes(n) -> ?Bytes` / `rng.os_u64() -> ?Int` | one-shot: open, read, close |
| `rng.seeded(seed: Int) -> Rng` | a stream fixed by `seed` |
| `rng.from_os() -> ?Rng` | a stream seeded from the operating system |
| `r.u64() -> Int` / `r.range(lo, hi) -> Int` | one draw |
| `r.fill(dst: Cursor, n)` / `r.bytes(n) -> Bytes` | `n` bytes |

`u64` answers the bit pattern of an unsigned 64-bit value, so half the
draws are negative `Int`s. That is the value, not a fault; `range` and
`fill` are the doors that never hand one back.

## `range` is uniform

`range(lo, hi)` is half-open and unbiased. A plain `draw % span`
favours the low residues whenever the span does not divide the draw's
range — invisible for a die and total for a span near the top of the
range — so a draw landing in the ragged tail above the last whole
multiple of the span is discarded and another taken. The expected cost
is a fraction of one extra draw for any span a caller actually asks
for.

An empty or reversed range answers `lo` rather than panicking, so a
caller that computed an empty range sees the mistake where the range
was computed.

## Filling a buffer you already hold

`fill` takes a cursor and allocates nothing. The offset lives in the
cursor rather than in a parameter because a `Bytes` passed as a
parameter is borrowed — the caller still holds it, so a write through it
lands in a copy the caller never sees.

```novo
use rng
use std.bytes

var c = bytes.cursor_be(bytes.zeros(64))
var r = rng.seeded(1)
r.fill(c, 64)
```

`bytes(n)` is the allocating convenience on top of it.

## Hold the handle

`os_bytes` and `os_u64` open the device, read, and close it. That is
right for a token drawn once and wrong inside a loop: hold an
`OsRandom` and call `fill` on it, then `close` when done. A handle that
outlives its use holds a file descriptor, and a process has a finite
number of those.

## Host only

The operating system source is `/dev/urandom`, so `OsRandom` works
where that file does: Linux, macOS and the BSDs. On a platform without
it, `os_open` answers `None` and every one-shot form answers `None` —
there is no fallback to a weaker source, because a caller who asked for
the secure generator should be told it is absent rather than handed
something else.

`Rng` has no such limit. It is arithmetic on `Int` with no allocation,
no syscall and no operating system, so it runs wherever the language
does. A target with a hardware entropy peripheral should read that and
pass the value to `seeded`.

## What it does not do

No distributions beyond the uniform one — no normal, no exponential, no
Zipfian. No shuffling and no weighted choice. Those belong in a package
that takes a generator rather than in the one that is a generator.

`Rng` is not a CSPRNG and no amount of seeding makes it one. If the
number matters to someone else, use the first column.

## Tests

```
novo test tests/rng_tests.nv
```

The external vector is splitmix64's published first output for seed 0,
which pins the seeding routine to the one xoshiro's authors specify.
The rest asserts what a caller relies on: the same seed replays
exactly, neighbouring seeds do not, every output bit moves, `range`
stays in bounds and reaches every value in them, `fill` writes exactly
what it was asked for and touches nothing past it, and the operating
system's device is there and gives different bytes each time.
