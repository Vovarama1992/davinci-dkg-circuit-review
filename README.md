# DaVinci DKG: ZK Circuit Review

Independent technical review of the `PartialDecrypt` and `DecryptCombine` paths in [vocdoni/davinci-dkg](https://github.com/vocdoni/davinci-dkg), connecting the protocol mathematics to Go/gnark constraints, witness construction, proof generation, node integration, and on-chain verification.

Reviewed revision: [`9ec5ff3`](https://github.com/vocdoni/davinci-dkg/tree/9ec5ff39be536871584e1b0a13b76d83c51078c0)  
Review date: 7 November 2026

This is a scoped independent review, not a full security audit of the entire DaVinci system. No confirmed exploitable vulnerability was found in the reviewed path.

## Main result

`PartialDecrypt` already proves inside Groth16 that one private share `d_i` explains both public points:

```text
D_i     = [d_i]G
sigma_i = [d_i]C1
```

The circuit then embeds a second Chaum–Pedersen/DLEQ proof of the same relation. That inner proof is redundant: the outer SNARK already proves the existence of the same `d_i` for both equations and already binds the public context through its public inputs.

In an experimental copy, I removed:

- the nested DLEQ transcript and response equations;
- output on-curve checks already implied by scalar-multiplication constraints;
- the secret range check, under the explicit assumption that `C1` is always validated as a member of the prime-order subgroup before proof generation.

The resulting circuit decreased from **26,194 to 5,208 constraints** (`-80%`) and local proving time on Apple M4 decreased from approximately **76.6 ms to 17.6 ms** (`~4.4x`). Package tests passed, the node compiled, and a proof-replay test confirmed that changing the epoch, application, ciphertext index, or participant index makes verification fail.

The numbers above measure the combined experimental simplification. They must not be attributed to the DLEQ removal alone without a separate benchmark.

## Protocol relation

The committee encrypts a vote point `M = [m]G` under the common public key `P = [s]G`:

```text
C1 = [k]G
C2 = M + [k]P
```

Committee member `i`, holding share `d_i`, publishes:

```text
D_i     = [d_i]G
sigma_i = [d_i]C1
```

Lagrange coefficients recover the shared secret in the exponent:

```text
s = sum(lambda_i * d_i)
sum([lambda_i]sigma_i) = [s]C1
```

The `DecryptCombine` relation also includes an independent organizer mask and finally requires:

```text
C2 = [plaintext]G + sum([lambda_i]sigma_i) + [d_org]C1
```

## What was traced

The review follows the full path rather than treating the circuit in isolation:

1. Go code constructs an `Assignment` from stored ciphertext and committee data.
2. `BuildWitness` derives the complete private/public witness.
3. `Define` has already been compiled into the constraint system.
4. `ProveAndVerify` creates a Groth16 proof and verifies it locally.
5. The node serializes the proof, public inputs, and transcript.
6. `DKGManager` checks that calldata matches stored on-chain state and invokes the on-chain verifier.

This distinction matters: local proof verification, contract state binding, and the generated Groth16 verifier are separate security layers.

## Findings

### 1. Nested DLEQ duplicates the outer SNARK relation

**Classification:** confirmed optimization, not a vulnerability.

The original circuit directly checks `D_i=[d_i]G` and `sigma_i=[d_i]C1`, then introduces a nonce `w`, points `A1=[w]G`, `A2=[w]C1`, Fiat–Shamir challenge `e`, and response `z=w+e*d_i mod l`, followed by:

```text
[z]G  = A1 + [e]D_i
[z]C1 = A2 + [e]sigma_i
```

These equations form a standalone Chaum–Pedersen proof inside Groth16. They restate a relation that the outer proof already enforces.

The experimental replay test creates one real Groth16 proof, then modifies each context field separately. Verification fails in every case, confirming that removing the inner transcript does not remove the outer proof's public-input binding.

### 2. Secret range check is conditionally redundant

**Classification:** conditional optimization, not a vulnerability.

If `G` and `C1` are in the subgroup of order `l`, then:

```text
[d_i + l]G  = [d_i]G
[d_i + l]C1 = [d_i]C1
```

Therefore `Secret <= l-1` chooses a canonical witness representation but does not change either public point. The reference node validates the ciphertext before creating the partial-decryption proof.

The boundary is important: the circuit and contract do not themselves prove subgroup membership of `C1`. Removing this range check is safe only if subgroup validation is mandatory for every permitted proof generator, or if the invariant is moved into a stronger trusted boundary.

### 3. Output on-curve checks duplicate stronger constraints

**Classification:** confirmed small optimization, not a vulnerability.

For the current Edwards gadgets, the scalar-multiplication equations already constrain the output points. Removing the four output checks in isolation reduced the circuit by 16 constraints. Honest proofs still verified; replacing an output with `(0,0)` failed in both Groth16 and PLONK test paths. The external input `C1` still requires validation.

### 4. Non-canonical Lagrange representatives are currently harmless

**Classification:** informational.

`DecryptCombine` computes coefficients in the subgroup scalar field but encodes them in the circuit field. The witness builder uses canonical integers, while the constraints may also accept representatives differing by a multiple of `l`.

In the current circuit the coefficients are applied directly to points of order `l`, so `[a+l]P=[a]P` and decryption does not change. This would stop being safe if a future version performed ordinary circuit-field arithmetic on two such representatives before the group operation. The source comment claiming a unique witness is therefore stronger than the actual guarantee.

### 5. Contract bindings were checked

For the reviewed `DecryptCombine` path, the contract prevents the combiner from freely replacing inputs:

- `C1` and `C2` are matched to the stored ciphertext hash;
- active committee indices must be present in the stored bitmap;
- every `sigma_i` is re-hashed and matched to an accepted partial decryption;
- the organizer key is matched to the current application;
- transcript commitments are recomputed from the submitted data.

No new vulnerability was confirmed in this integration path.

## Reproduction evidence

| Check | Result |
|---|---|
| Baseline package tests | passed |
| Simplified `PartialDecrypt` package tests | passed |
| `DecryptCombine` package tests | passed |
| Node compile-only test | passed |
| One Groth16 proof with modified context inputs | all replay attempts rejected |
| Combined simplified circuit size | `26,194 -> 5,208` constraints |
| Apple M4 proving benchmark, three 5x runs | approximately `76.6 -> 17.6 ms/proof` |
| Isolated output on-curve removal | `-16` constraints; invalid `(0,0)` rejected |
| `DecryptCombine` local proof series | `100/100` passed on the first attempt |

The experimental code is included as [`PARTIALDECRYPT_SIMPLIFICATION.patch`](PARTIALDECRYPT_SIMPLIFICATION.patch). It is a research patch, not a production-ready deployment change.

## Required work before any merge

1. Make the subgroup-validation trust boundary for `C1` explicit.
2. Regenerate circuit, proving, verifying, and Solidity verifier artifacts.
3. Update the Solidity public-input layout.
4. Run the complete integration and contract suites on real calldata/transcript paths.
5. Benchmark each removed constraint class separately.
6. Update the misleading `DecryptCombine` uniqueness comment.

## Scope limitations

Out of scope: a full review of DKG contribution/finalization, the curve implementation, Groth16 itself, trusted setup, the entire Solidity contract, network consensus/liveness, gas/DoS, and third-party node implementations.

The complete Russian-language technical report, with detailed code links and mathematical derivations, is available in [`REPORT_RU.md`](REPORT_RU.md).

## Author

Independent review by [Vovarama1992](https://github.com/Vovarama1992). I focus on Go, gnark circuits, DKG, threshold encryption, and verifying that a proof system's implemented constraints match the claimed cryptographic statement.
