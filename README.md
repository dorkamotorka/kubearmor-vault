# Protecting HashiCorp Vault using KubeArmor

HashiCorp Vault is commonly used for storing and managing encryption keys, passwords, and API credentials in on-prem clusters.

While these secrets are encrypted by default, encryption alone doesn't prevent attackers from deleting or re-encrypting them.

This alone can compromise the availability of your services that depend on these secrets, potentially resulting in significant financial and operational impacts.

The code in this repository, show you how you can harden the security using KubeArmor.

<img width="1181" alt="Screenshot 2024-10-17 at 3 05 34 PM" src="https://github.com/user-attachments/assets/b4d6f915-4ffd-4b69-8457-277fb37cf3ea">

## Prerequisites

In order to deploy Vault. I used Helm for this - here's the [guide](https://developer.hashicorp.com/vault/docs/platform/k8s/helm).

We also have to set up KubeArmor - please folow their [Getting Started Guide](https://docs.kubearmor.io/kubearmor/quick-links/deployment_guide).

To ease the configuration and debugging of KubeArmor I'm gonna also use [kArmor CLI](https://docs.kubearmor.io/kubearmor/quick-links/deployment_guide#install-karmor-cli-optional). 

## Security Enforcement

KubeArmor runs as a DaemonSet on your Kubernetes Cluster. 

In order to secure HashiCorp Vault, we can use kArmor to suggest as the policies.

```
karmor recommend -n vault # In this example, vault is deployed in the "vault" namespace
```

This inspects the deployments in the provided namespaces `vault` and suggest the pre-configured security policies. These are based on industry-leading compliance and attack frameworks such as CIS, MITRE, NIST-800-53, and STIGs.

Policies are generated in `out` directory. In this repository I retained them in `policies` directory.

We can then go ahead and utilize any of the recommendations like:

- `hashicorp-vault-k8s-1-4-2-crypto-miners.yaml`: Prevents the execution of known cryptocurrency mining processes and blocks access to specific directories.
- `hashicorp-vault-k8s-1-4-2-pkg-mngr-exec.yaml`: Blocks the execution of package management processes.
- `hashicorp-vault-k8s-1-4-2-remote-file-copy.yaml`: Prevents the execution of remote file copy tools like rsync and scp to mitigate the risk of data exfiltration

Policies are applied using `kubectl apply -f <policy-file>`.

Then we can use `karmor logs` to inspect the KubeArmor policy enforcement.

Since your infrastructure can be complex, and the amount of KubeArmor policies can quickly grow, it might be better to utilize `karmor profile` for a more concise output.

**Advanced**: KubeArmor also exposes API to export it's logs in a JSON format and be consumed by any observability platform like Grafana or ELK.

## Zero Trust Policy

Since the recommended security policies only prevent the known attacks, it's important to think about how to make your service secure in future.

This is achieved through Zero Trust Policy (ZTP) - which is another way of saying — don't trust anything by default, just the actions you specifically allow.

To demonstrate this on HashiCorp Vault, let's create a custom KubeArmor policy that **ONLY** allows execution of the `/bin/vault` binary and blocks the rest.

```
apiVersion: security.kubearmor.com/v1
kind: KubeArmorPolicy
metadata:
  name: vault-zero-trust 
  namespace: vault
spec:
  action: Allow
  selector:
    matchLabels:
      app.kubernetes.io/instance: vault
      app.kubernetes.io/name: vault 
  process:
    matchPaths:
    - path: /bin/vault
  message: Alert! Untrusted execution detected
  severity: 5
```
**Note**: This is not a production-ready policy, but rather just the proof-of-concept.

Important to know is that KubeArmor by default doesn't block anything, but rather audits. In other words, when infering the ZTP, all actions will initially be passed through.

This allows the developer to understand the operations of the workload through the `karmor logs` and design a ZTP policy based on that. 

And after that, you can configure KubeArmor, to block all actions (by default) by modifying KubeArmor configuration:
```
defaultFilePosture: block # audit by default
defaultNetworkPosture: block # audit by default
defaultCapabilitiesPosture: block # audit by default
```
In the [KubeArmor Helm Chart](https://artifacthub.io/packages/helm/kubearmor/kubearmor), this is done through the `kubearmorConfigMap` parameter.

**Note**: This is a global setting so it will affect all of your Kubernetes workload. If you want to further restrict it to a namespace, please check [KubeArmor Default Posture](https://github.com/kubearmor/KubeArmor/blob/main/getting-started/default_posture.md).
