# ZTA Policy Attestor

## Overview
The **ZTA Policy Attestor** is a custom GitHub Action designed for Zero-Trust supply chain security. It cryptographically binds a set of infrastructure and runtime security boundaries directly to a Docker image. 

This action reads a developer-friendly `.yaml` security policy, automatically converts it into a strict `.json` payload, and leverages **Sigstore Cosign** (Keyless/OIDC) to attach it to the image in the OCI registry as a custom in-toto attestation (`custom-zta-policy`). Kubernetes Admission Controllers or Operators can then verify this attestation at deployment time to prevent runtime drift and privilege escalation.

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

## Example Usage

Below is a complete workflow example demonstrating how to build an image and attach the ZTA security policy.

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
        uses: SabinGhost19/zta-policy-attestor@v1.0.0
        with:
          image: ghcr.io/${{ github.repository_owner }}/demo-api@${{ steps.build-image.outputs.digest }}
          policy-path: ./security-policy.yaml
```

## Specifications: `security-policy.yaml` (The "Cryptographic Backpack")

### Role of the File
The `security-policy.yaml` file represents the security contract of your application. This file defines the **maximum privileges** that your Docker image can request in the Kubernetes cluster.
During the CI/CD pipeline, this file is converted to JSON, cryptographically signed, and attached to the Docker image. The Zero-Trust Operator will reject any deployment that attempts to request more privileges at runtime than those declared here.

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
