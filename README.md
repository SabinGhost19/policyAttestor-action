# ZTA Policy Attestor (Custom GitHub Action)

Această acțiune transformă un fișier `security-policy.yaml` în JSON și îl atașează imaginii Docker ca atestare Cosign de tip `custom-zta-policy`.

## Input-uri

- `image` (required): referință imagine (ideal digest)
- `policy-path` (required): calea către policy YAML
- `attestation-type` (optional, default `custom-zta-policy`)

## Exemplu utilizare

```yaml
- name: Attach ZTA policy attestation
  uses: ./customCRD/customGithubActionAction-zta-policy-attestor
  with:
    image: ${{ steps.imagevars.outputs.image_repo }}@${{ steps.build-and-push.outputs.digest }}
    policy-path: ./customCRD/demo-app/security-policy.yaml
    attestation-type: custom-zta-policy
```

## Cerințe

- workflow permissions: `id-token: write`, `packages: write`
- `cosign` instalat în job înainte de apelarea acțiunii
