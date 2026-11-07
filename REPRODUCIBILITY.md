# Reproducibility notes

The review targets upstream commit:

```text
9ec5ff39be536871584e1b0a13b76d83c51078c0
```

The included patch was produced from the local experimental copy against that revision.

Core checks used during the review:

```bash
go test ./circuits/partialdecrypt ./circuits/decryptcombine
go test ./node -run '^$'
go test ./circuits/partialdecrypt -run TestPartialDecryptRejectsCrossEpochReplay
```

The patch changes the circuit's public input layout. Existing proving/verifying artifacts intentionally no longer match it. Production use requires regeneration of all circuit artifacts and the Solidity verifier, followed by full integration testing.

Benchmark figures in the report were measured on Apple M4 using the repository's `BenchmarkProve`. They are local engineering measurements, not a cross-platform performance guarantee.
