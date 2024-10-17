# Protecting HashiCorp Vault using KubeArmor

HashiCorp Vault is commonly used for storing and managing encryption keys, passwords, and API credentials in on-prem clusters.

While these secrets are encrypted by default, encryption alone doesn't prevent attackers from deleting or re-encrypting them.

This alone can compromise the availability of your services that depend on these secrets, potentially resulting in significant financial and operational impacts.

The code in this repository, show you how you can harden the security using KubeArmor.

<img width="1181" alt="Screenshot 2024-10-17 at 3 05 34 PM" src="https://github.com/user-attachments/assets/b4d6f915-4ffd-4b69-8457-277fb37cf3ea">

## Prerequisites

In order to deploy Vault. I used Helm for this - here's the [guide](https://developer.hashicorp.com/vault/docs/platform/k8s/helm).

We also have to set up KubeArmor - please folow their [Getting Started Guide](https://docs.kubearmor.io/kubearmor/quick-links/deployment_guide#install-karmor-cli-optional).

To ease the configuration and debugging of KubeArmor I'm gonna also use kArmor CLI. 

## Security Enforcement

KubeArmor runs as a DaemonSet on your Kubernetes Cluster. 

In order to secure HashiCorp Vault, we can use kArmor to suggest as the policies.

```
karmor recommend -n vault 
```

This inspects the deployments in the provided namespaces and suggest the pre-configured security policies, that are based on industry-leading compliance and attack frameworks such as CIS, MITRE, NIST-800-53, and STIGs.

Policies are generated in `out` directory. In this repoitory I retained them in `policies` directory.

We can then go ahead and utilize any of the recommendations.

Then we can use `karmor logs` to inspect the KubeArmor enforcement.

Since your infrastructure is complex, and the amount of KubeArmor policies can quickly grow, it might be better to utilize `karmor profile` for a more concise output.

KubeArmor also exposes API to export it's logs in a JSON format and be consumed by any observability platform like Grafana or ELK.

## Performance Test

TODO: show how much processing does KubeArmor takes (kubectl top pods -n kubearmor)
