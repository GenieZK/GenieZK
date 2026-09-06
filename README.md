# GenieZK

Decompose a field element into 254 bits over BN254 and, for about a third of the possible values,
the circuit will accept two different bit strings for the same one. Nothing in the constraint
system says which of them counts, so the prover picks. That gap is the kind of thing I look for: a primitive that works exactly as specified,
sitting under code that assumed the specification closed a door it left open.

I work on proof systems — arithmetization, polynomial commitments, recursion — and on the circuit
bugs that survive review.

Privacy, to me, is a question of control: which single fact you let someone verify, and what they
can infer past it. Zero-knowledge proofs are what make that question answerable in a form another
party can check.

## Current work

**OriProof** — proof of origin for digital content. Specification stage. The next thing to land is
a reference circuit for the one claim that actually needs zero knowledge: *this re-upload derives
from a work I registered*, proven without revealing the original, the owner, or the license terms.

Crawling platforms and matching perceptual hashes at web scale is the other half of that problem,
and it belongs to infrastructure rather than to cryptography. I keep the two apart. Conflating
them is how a protocol ends up claiming a trust model it does not have.

## What I write about

- **Proof systems compared on the axes that decide a design**: proof size, verifier cost, setup
  assumptions, recursion cost. Adjectives do not appear in the comparison.
- **Circuit failure modes**: under-constrained witnesses, aliasing in bit decomposition,
  unconstrained denominators, nullifiers that replay across scopes, Fiat–Shamir transcripts that
  absorb too little. Each one gets a minimal circuit that reproduces the bug, plus a test that
  fails before the fix and passes after it.
- **Cost, measured.** Constraint counts and prover times come with the parameters, the library
  version, and the hardware they were taken on. A number without those is not a number.

## How I state things

Claims carry their assumptions. I distinguish an argument from a proof, and computational
soundness from statistical soundness; a scheme is zero-knowledge only against the adversary its
simulator was built for. Where I have not measured something, I say so.

## Notes

- [Two bit strings, one field element](notes/0001-bit-decomposition-aliasing.md) — 254-bit
  decomposition over BN254 is under-constrained in one direction only, which is why the check that
  looks like the obvious demonstration is the one case that holds.

## Contact

Open an issue on any repository here.
