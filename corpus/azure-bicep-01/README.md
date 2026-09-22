# Test case: azure-bicep-01

> **WARNING: `target/main.bicep` is deliberately insecure and must never be deployed.**
>
> It contains 18 intentional misconfigurations and exists solely as evaluation
> material. Deploying it would create publicly readable storage containing
> apparent customer data, an internet facing SQL server with a firewall rule
> permitting every IP address, and a Key Vault with no deletion protection and
> a wildcard access policy.

---

## Contents

```
target/
  main.bicep        The artefact under test. Mount THIS FOLDER read only.
ANSWER-KEY.md       The 18 planted issues and scoring guidance. Never mount this.
```

The directory split is deliberate. Mounting `target/` rather than this folder
keeps the answer key structurally out of reach, so an agent cannot read the
solution by accident. If the key reaches the agent's context the result is
meaningless, and the failure is silent.

## What it deploys

A small Azure environment: a storage account with a blob container, a virtual
network with an NSG, a Key Vault holding a secret, and a SQL server with one
database. Realistic enough that the misconfigurations read as plausible
mistakes rather than an obvious puzzle.

## Do not modify main.bicep

Line numbers cited in `results/` refer to this exact file. Adding so much as a
comment at the top shifts every reference and invalidates the groundedness
checks recorded against previous runs.

If the case needs to change, create `azure-bicep-02` rather than editing this
one. Results are only comparable across runs when the input is identical.

## Scoring

See `ANSWER-KEY.md` for the planted issues, the bonus list of acceptable extra
findings, and the dimensions each run is scored on.
