# Design Journal — issue-102-cloud-kms-agent-signer

### 2026-06-29 · §10

Extended the cryptographic verification and signing infrastructure to support EC (Elliptic Curve) algorithms alongside the existing Ed25519 support. The platform now recognizes P-256, P-384, and P-521 curves and derives their corresponding JCA signature algorithms (SHA256withECDSA, SHA384withECDSA, SHA512withECDSA) from the key material. This was a prerequisite for cloud KMS adapters — AWS KMS, GCP Cloud KMS, and Azure Key Vault all produce EC signatures, and the existing verification path could only handle Ed25519. A shared `SignatureAlgorithms` utility now maps EC curve parameters to signature algorithms, and the trial-load lists in both `AgentCryptographicVerifier` and `LedgerPemUtil` now include "EC" after "Ed25519". The design preserves algorithm transparency: no signature algorithm is hardcoded; all are derived from the stored X.509 public key bytes.
