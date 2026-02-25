# ZTA Policy Attestor

## Overview
The **ZTA Policy Attestor** is a custom GitHub Action designed for Zero-Trust supply chain security. It cryptographically binds a set of infrastructure and runtime security boundaries directly to a Docker image. 

This action reads a developer-friendly `.yaml` security policy, automatically converts it into a strict `.json` payload, and leverages **Sigstore Cosign** (Keyless/OIDC) to attach it to the image in the OCI registry as a custom in-toto attestation.

If you provide `manifest-dir`, the action also computes a canonical SHA-256 hash of the `ZeroTrustApplication.spec` and injects it into the predicate as `expected_infra_hash`. By default, it normalizes `.spec.image` to the current `image` input (`sync-manifest-image: true`) so the hash matches the newly built digest. This allows the operator to enforce strict GitOps manifest integrity at deploy/runtime.

## Prerequisites
To use this action, your GitHub Actions workflow must have the necessary permissions to generate OIDC tokens (for Keyless signing) and write to the container registry.

```yaml
permissions:
  contents: read
  packages: write
  id-token: write
```

## Inputs

| Input | Required | Description | Example |
| --- | --- | --- | --- |
| `image` | **Yes** | The full OCI registry path to the Docker image. **Must include the SHA256 digest**, not a tag. | `ghcr.io/user/app@sha256:123...` |
| `policy-path` | **Yes** | The relative path to the YAML security policy file in your repository. | `./security-policy.yaml` |
| `manifest-dir` | No | Directory containing GitOps manifests. The action finds YAML files, extracts exactly one `ZeroTrustApplication.spec`, computes SHA-256, and injects `expected_infra_hash` into the attestation predicate. | `./k8s-repo/demo-api` |
| `sync-manifest-image` | No | Defaults to `true`. When enabled, hash computation uses the current `image` input as `ZeroTrustApplication.spec.image`, avoiding digest drift between freshly built image and manifest snapshot. | `true` |
| `attestation-type` | No | Predicate type used by `cosign attest`. Defaults to a URI form accepted by Cosign. | `https://devsecops.licenta.ro/attestations/custom-zta-policy/v1` |

## Output Behavior

- Always attests the policy payload generated from `policy-path`.
- When `manifest-dir` is provided, enriches payload with `expected_infra_hash`.
- Signs and uploads attestation to OCI registry bound to the immutable image digest.
- Uses keyless OIDC signing (`id-token: write` required).

## Example Usage

### Policy only

```yaml
name: Build and Attest

on:
  push:
    branches: [ "main" ]

permissions:
  contents: read
  packages: write
  id-token: write 

jobs:
  build-and-secure:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and Push Docker Image
        id: build-image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository_owner }}/demo-api:${{ github.sha }}

      - name: Attach ZTA Security Attestation
        uses: SabinGhost19/policyAttestor-action@v1
        with:
          image: ghcr.io/${{ github.repository_owner }}/demo-api@${{ steps.build-image.outputs.digest }}
          policy-path: ./security-policy.yaml
          attestation-type: https://devsecops.licenta.ro/attestations/custom-zta-policy/v1
```

### Policy + strict manifest hash (separate config repo)

```yaml
name: Build and Attest (Policy + Hash)

on:
  push:
    branches: [ "main" ]

permissions:
  contents: read
  packages: write
  id-token: write

jobs:
  build-and-secure:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout app repository
        uses: actions/checkout@v4

      - name: Checkout GitOps manifests repository
        uses: actions/checkout@v4
        with:
          repository: SabinGhost19/my-k8s-manifests
          ref: main
          path: k8s-repo

      - name: Build and Push Docker Image
        id: build-image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository_owner }}/demo-api:${{ github.sha }}

      - name: Attach ZTA Security Attestation (with expected_infra_hash)
        uses: SabinGhost19/policyAttestor-action@v1
        with:
          image: ghcr.io/${{ github.repository_owner }}/demo-api@${{ steps.build-image.outputs.digest }}
          policy-path: ./security-policy.yaml
          manifest-dir: ./k8s-repo/demo-api
          attestation-type: https://devsecops.licenta.ro/attestations/custom-zta-policy/v1
```

## Specifications: `security-policy.yaml` (The "Cryptographic Backpack")

### Role of the File
The `security-policy.yaml` file represents the security contract of your application. This file defines the **maximum privileges** that your Docker image can request in the Kubernetes cluster.
During the CI/CD pipeline, this file is converted to JSON, cryptographically signed, and attached to the Docker image. The Zero-Trust Operator will reject any deployment that attempts to request more privileges at runtime than those declared here.

When strict hash mode is enabled in `SupplyChainAttestation.spec.strictManifestHash`, the Operator also compares the runtime `spec` hash with `expected_infra_hash` from this attestation.

### Structure and Fields

#### 1. Metadata
* **`policyVersion`** (String): Security schema version (current: `"1.0"`).
* **`workloadIdentity`** (String): The unique name of the application (e.g., `"demo-api"`). Must match the name in the Kubernetes manifest.

#### 2. Network Security (`securityBoundaries.network`)
Defines who the container is allowed to communicate with.
* **`allowGlobalInternet`** (Boolean): If `false`, the Operator will block any Egress traffic requests to external IPs/domains (0.0.0.0/0).
* **`allowedEgressNamespaces`** (List of Strings): Specifies the internal namespaces the application is allowed to communicate with (e.g., `["database", "redis-cluster"]`). Any other namespace will be blocked.
* **`restrictedPorts`** (List of Integers): Ports the application is **not** allowed to open or access, regardless of destination (e.g., `[22, 3306]`).

#### 3. Secrets Management (`securityBoundaries.secrets`)
Controls access to HashiCorp Vault.
* **`requireZeroTrustSecrets`** (Boolean): If `true`, the application is not allowed to use native Kubernetes `Secret`s stored in clear text (Base64). The Operator will force the use of secure injection via External Secrets Operator (ESO) or RAM-disk.
* **`allowedVaultPaths`** (List of Strings): The exact paths in Vault from where the application is allowed to extract passwords. If the K8s manifest requests a password from `secret/data/prod/admin`, but it is not listed here, the deployment is rejected.

#### 4. Runtime Constraints (`securityBoundaries.runtime`)
Controls the security context of the container.
* **`allowPrivilegeEscalation`** (Boolean): Must be `false` to prevent child processes from obtaining more privileges than the parent process.
* **`runAsRoot`** (Boolean): If `false`, the Operator will force the container to run with a non-root UID (e.g., 1000). Any attempt to run as root will cause a conflict.

### Policy File Example

The action expects a YAML file defining the boundaries. Example:

```yaml
policyVersion: "1.0"
workloadIdentity: "demo-api"
securityBoundaries:
  network:
    allowGlobalInternet: false
    allowedEgressNamespaces: 
      - "database"
      - "internal-services"
    restrictedPorts: 
      - 22 
      - 3306
  secrets:
    requireZeroTrustSecrets: true
    allowedVaultPaths:
      - "secret/data/prod/postgres/demo-api"
      - "secret/data/common/api-keys"
  runtime:
    allowPrivilegeEscalation: false
    runAsRoot: false
```
