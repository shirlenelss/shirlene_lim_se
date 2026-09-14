+++
date = '2026-09-14T22:47:09+02:00'
draft = false
title = 'Why secret extraction matters'
tags = ['kubernetes', 'argo', 'gitops', 'secrets', 'security']
+++

Recently, I have been working on our kubernetes cluster on argo and gitops.
A senior colleague mention that we had a pretty significant high security concern.
Woohoo, what was it? Secrets in the git repo. Unsealed and laid out in clear text, certs in all their glory up, whole and up for grab. Stupid passwords that children can guess.

Should we do something about it? Of course! Let's get dirty!
Will I layout my plan for my organisation? Nope! But here's the gist of the matter,

## Why Manifest Extraction of Secrets Matters
- **Prevents GitLeaks:** Standard git commits keep history forever. Hardcoded secrets are easily exposed.
- **Enables RBAC:** Separates cluster infrastructure management (DevOps) from credential rotation and security policy (SecOps).
- **Automates Rotation:** Simplifies updating expiring certificates or compromised passwords without redeploying the entire app stack.

Available solutions?
## Architecture injection?
```
[ Git / Cluster Manifest ] ──> References Secret Key    
                                    │                                                         
                                    ▼
[ External Secret Store ] ───> [ Injection Mechanism ] ───> [ Running Pod ]

(Vault, AWS, Azure, GCP)     (External Secrets / CSI)       (Env Var / Vol Mount)
```

Let's get the agent / controller to inject the secret

# Implementation Strategies

##  The Modern Cloud-Native Approach:
## External Secrets Operator (ESO)
ESO is a Kubernetes operator that integrates with external secret management systems (like HashiCorp Vault, AWS Secrets Manager, Azure Key Vault, or Google Secret Manager).
How it works: You write an ExternalSecret manifest.
The operator fetches the secret from your external provider and automatically generates a standard Kubernetes Secret in the background.

The Manifest:yaml
```yaml
 apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: database-credentials
spec:
  refreshInterval: "1h" # Automatic rotation check
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: app-db-secret # The local K8s secret generated automatically
  data:
    - secretKey: password
      remoteRef:
        key: prod/database
        property: password
```
so long as the etcd is encrypted and not accessible to most users and roles, is this secure enough for most usage?
yes? maybe...

## The Direct-to-App Approach: Secrets Store CSI Driver
f you do not want secrets stored as regular Kubernetes secrets in etcd at all, use the Container Storage Interface (CSI) driver.

How it works: The driver mounts secrets directly from your external vault into the pod's file system as a volume. The data exists only in the pod's volatile memory.

Best For: High-security environments or compliance frameworks (e.g., PCI-DSS) that restrict writing secrets to disk. hmm...

## For Managing Certificates: cert-manager
Never store SSL/TLS certificates manually in your manifests.
Use cert-manager to automate the lifecycle.How it works: It acts as a Kubernetes native controller. It interfaces with Certificate Authorities (like Let's Encrypt, DigiCert, or an internal HashiCorp Vault CA), requests certificates automatically, and provisions them directly into the cluster.

| Tool / Strategy           | Best For                                    | Storage Location                       | Automatic Rotation      |
| ------------------------- | ------------------------------------------- | -------------------------------------- | ----------------------- |
| External Secrets Operator | General app secrets & environment variables | Replicated to K8s etcd                 | Yes                     |
| Secrets Store CSI Driver  | Ultra-secure workloads, bypassing etcd      | Ephemeral pod memory                   | Yes                     |
| cert-manager              | TLS/SSL Certificates                        | K8s etcd (as TLS type)                 | Yes (Automatic renewal) |
| Sealed Secrets            | Small teams without an external Cloud Vault | Encrypted in Git, decrypted inside K8s | Manual                  |

### Crucial SecOps Anti-Patterns to Avoid
Encoding vs. Encrypting:
Base64 encoding (standard Kubernetes secrets) is not encryption.
Anyone with read access to the namespace can decode it instantly (base64 --decode).

Committing .env files:
Ensure your .gitignore explicitly blocks private environment configurations.

Using Git Crypt tools for shared keys:
While tools like git-crypt encrypt files in your repo, they make key rotation highly cumbersome if a team member leaves.

Use a centralized identity management provider instead.

## Note: As for me, I'll be implementing extractions stage by stage. 
First, our certificates and secret being plucked out from clear text to kubernetes resources. 
I'll reweigh my options again when I get to the stage of implementing a vault or secrets store.
Gitops, pipelines and devops to think about. 
