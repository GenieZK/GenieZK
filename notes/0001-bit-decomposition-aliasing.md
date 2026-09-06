# Two bit strings, one field element

Decomposing a field element into 254 bits over BN254 is under-constrained. circomlib has shipped
the fix since December 2018
([`aliascheck.circom`](https://github.com/iden3/circomlib/blob/2d43178/circuits/aliascheck.circom)),
Trail of Bits lints for it under
["Non-strict binary conversion"](https://github.com/trailofbits/circomspect/blob/main/doc/analysis_passes.md),
and Marco Besier covered it for zkSecurity in "Common Circom Pitfalls and How to Dodge Them,
Part 1".

The direction of that freedom decides which checks survive it, and the direction is fixed. An
element `v` has a second 254-bit encoding exactly when `v + r < 2^254`, and that second encoding is
`v + r` itself. So **the prover can add `r`; it can never subtract.**

The bits make that concrete:

```
G = 2^254 - r = 7059779437489773633646340506914701874769131765994106666166191815402473914367
2^252         = 7237005577332262213973186563042994240829374041602535252466099000494570602496
```

`G < 2^252`, and `r > 3 * 2^252`. Every aliasable `v` is below `G`, hence below `2^252`, so its
canonical encoding carries `00` in the top two bits. Its alias lands in `[r, 2^254)`, above
`3 * 2^252`, so that one carries `11`. Every time, with no reverse case.

Which is why the check that looks like the obvious demonstration of this bug — force the top bit to
zero, conclude the value is small — is the one case that holds.

## Where the freedom comes from

`r` is 254 bits and sits below `2^254`:

```
r = 21888242871839275222246405745257275088548364400416034343698204186575808495617
```

A 254-bit string `B` maps to the field element `B mod r`. Since `r < 2^254 < 2r`, that reduction is
the identity when `B < r` and subtracts `r` exactly once otherwise. So `v` has two encodings
precisely when `v < G`, a set covering 32.25% of the field's elements, and no element has three,
since `v + 2r >= 2r` exceeds `2^254`.

Now `Num2Bits(n)`, verbatim from circomlib at
[`35e54ea`](https://github.com/iden3/circomlib/blob/35e54ea/circuits/bitify.circom):

```
for (var i = 0; i<n; i++) {
    out[i] <-- (in >> i) & 1;
    out[i] * (out[i] -1 ) === 0;
    lc1 += out[i] * e2;
    e2 = e2+e2;
}
lc1 === in;
```

Two families of constraints: each output is boolean, and the weighted sum equals the input. The
first line is `<--`, a witness assignment that generates no constraint at all. It tells an honest
prover how to fill the signals; nothing compels a dishonest one to agree, and for `v < G` both
encodings satisfy everything the system does impose.

At that same commit, `bitify.circom` contains no `assert`, while `comparators.circom` opens
`LessThan(n)` with `assert(n <= 252)`. The compiler will stop you from writing a 253-bit
comparison. It will let `Num2Bits(254)` through.

## What survives and what falls

**Upper bounds survive.** A circuit that constrains `out[253] === 0` and concludes `in < 2^253` is
sound against a malicious prover: aliases carry `1` in bit 253, so the constraint excludes all of
them and the surviving witness is canonical. Zeroing bit 252 instead also excludes every alias,
since aliases carry `1` there too, and that step needs `r > 3 * 2^252` rather than
`B < 2^253 < r` — which is why the two-bit version of the fact earns its place. What zeroing bit
252 alone does not buy you is a bound: `B = 2^253` clears that bit and is canonical, so the
constraint says nothing about the size of `in`. Excluding aliases and bounding a value are two
different jobs, and only the first one comes free.

Veridise's UniRep audit report classifies exactly this pattern (finding V-UNI-VUL-013) as an
unnecessary-constraints warning rather than a vulnerability. That reading is correct.

**Lower bounds fall.** Constrain `out[253] === 1` to conclude `in >= 2^253`, and for `in = 1` the
encoding of `r + 1` satisfies it. The claim is false and the forgery costs nothing.

**Hand-rolled comparators fall.** The standard trick computes `x = a + 2^k - b`, decomposes `x`,
and reads the top bit as a sign. At 254 bits the sign is forgeable whenever `x` lands in the gap:
with `a = 0` and `b = 2^253 - 1`, `x = 1`, whose alias `r + 1` has bit 253 set, and the circuit
concludes `a >= b`. This is why circomlib's own `LessThan` refuses `n > 252`.

**Bits used as identity fall hardest.** Hashing the bits, XOR-ing them, splitting them into
chunks, deriving a nullifier from them. Here the two encodings act as two distinct inputs, so one
field element yields two distinct outputs: two valid nullifiers for the same note.

That last class has a production record, though the recorded instance sits in the verifier contract
rather than in a circuit. [Semaphore issue #16](https://github.com/semaphore-protocol/semaphore/issues/16),
filed in 2019, reports that the contract never checked the nullifier against the modulus, so
`nullifier_hash + r` passed verification as a distinct nullifier and allowed a double spend. Same
root cause: a value with two encodings and no decision about which one counts.

I have not found a public post-mortem of an exploited *in-circuit* instance. What I searched: the
0xPARC bug tracker, zkSecurity's bug database, circomspect's lint documentation, and the Veridise
report above. That is an empty search result, and I would not read more into it than that.

## Fixes

**Use at most 253 bits.** Any `n`-bit string with `n <= 253` is below `2^253 <= r`, so the map from
bit string to field element is injective. The exact condition is `2^n <= r`, which for any 254-bit
modulus means `n <= 253`, and the threshold is tight: `0` and `r` collide the moment `2^n > r`.
There is a trade. `n <= 253` also imposes `in < 2^n`, so an honest prover holding a larger input
can no longer produce a witness at all. That converts an unsoundness into an incompleteness, which
is the right direction and still a cost.

**Or enforce canonicity.** `Num2Bits_strict` runs `AliasCheck`, which feeds the bits to
`CompConstant(-1)` and constrains `compConstant.out === 0` — the encoding, read as an integer, must
be at most `r - 1`. The implementation accumulates over 127 bit-pairs and reads out a carry, rather
than scanning from the top bit down. And `AliasCheck`, `CompConstant`, and both `_strict` templates
hard-code 254, so the bound they enforce is BN254's modulus rather than the modulus you compiled
against. I have not tested what circom does when `-p` selects a 255-bit prime; the check is at
minimum measuring the wrong field, and may not compile at all.

## The general shape

A constraint system defines a relation, not a function. Wherever the relation admits several
witnesses for one public input, the prover holds the choice and everything downstream inherits it.

So the audit question is not whether a circuit is under-constrained, but which direction the extra
freedom points.
