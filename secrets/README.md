# Secret Management

To manage secrets safely at least some portion of secret should be stored outside of Git. This portion should be used to either encode/decode or access secrets.

In Kubernetes context this should be implemented via in-cluster controller/operator that would handle retrieval and presentation of secrets via k8s native `Secret` resources. Applications in turn should reference those secrets in ENVs.

Here are couple options available.

## External Secrets Operator

[ESO](https://external-secrets.io/) is a k8s operator that defines `ClusterSecretStore` backend and `ExternalSecret` instance of a secret. Former is usually configured by cluster administrator by providing operator pod access to the backend of choice, while latter is a reference to some secret value in that backend. Example can be seen in [Getting started](https://external-secrets.io/v0.7.2/introduction/getting-started/#create-a-secret-containing-your-aws-credentials) section.

It supports many popular cloud providers' secret management products like `Secrets Manager` in AWS case and standalone solutions like Vault, so there should be no compatibility issue.

After installation and initial configuration day-to-day workflow consists of putting secret into backend (for example via AWS dashboard in `Secrets Manager` section) and adding `ExternalSecret` referencing this secret to application manifests.

It's easy to set up and use, but has some drawbacks:
1. It requires giving access to sensitive part of AWS to everyone who is intended to manage those secrets, which in a "shift left" tendency would mean "a lot of devs".
2. It's not completely GitOps solution, since Git contains only references to secrets, not actual values. Those values can be changed via AWS dashboard and application behavior would change without anything happening in Git.

## Mozilla SOPS Operator

An alternative solution to that problem could be [SOPS Operator](https://github.com/isindir/sops-secrets-operator) it's a k8s wrapper for [Mozilla SOPS](https://github.com/mozilla/sops).

It's similar to previous solution in a way that it introduces it's own CRD `SopsSecret` that is transformed into regular k8s `Secret` using some secret storage backend.

The main difference is that `SopsSecret` contains not references to entries in some secret storage backend, but actual encrypted secret values. And secret storage backend is used to store only decryption key. It's cluster admin's job to configure operator in a way that would allow retrieval of that decryption key.

After installation and initial configuration day-to-day workflow consists of decrypting `SopsSecret` manifest with correct key, updating it, re-encrypting it back and committing the change to Git.

Main benefits compared to previous solution are:
1. Less permision sprawl, since it's easier to limit access to a single `KMS Key` rather than to subset of secrets in `Secrets Manager`.
2. Truly GitOps solution that makes tracing credential problems easier e.g.: "thnigs stopped working after commit that introduced typo in database password".

The downside of this solution is in steeper learning curve, since using AWS dashboard is usually more intuitive than using CLI crpytographic tools.
