+++
date = '2026-06-23T02:35:27+02:00'
draft = false
title = 'Hashicorp vault'
tags = ['vault', 'pki', 'istio', 'kubernetes', 'security', 'secrets']
+++

my notes on vault, pki and istio integration

## dev mode
`vault server -dev`
You may need to set the following environment variables:

```
``$ export VAULT_ADDR='http://127.0.0.1:8200'
```

it will display this:
```
The unseal key and root token are displayed below in case you want to
seal/unseal the Vault or re-authenticate.

Unseal Key: FAKE_KEYdqJbMKGrDhiTaWbqQDmGQWt7Ys9i5ffGMs=
Root Token: hvs.uewakWY6selwjfLBLAHBLAH
```
Root Token is the password to access the GUI

## prod mode
### starting vault server with policy
`vault server -config=vault.hcl`

### policy
dir structure
```bash
.
├── config
     ├── dev.hcl
     └── sandbox.hcl
└── vault-data
├── dev
└── sandbox

create [[vault policy]] config/sandbox.hcl and config/dev.hcl

path "secret/data/{{identity.entity.id}}/*" {
capabilities = ["create", "update", "patch", "read", "delete"]
}
storage "file" {  
path = "/home/dev1/project/mine/vault_basic/vault-data/dev"  
}
storage "file" {  
path = "/home/dev1/project/mine/vault_basic/vault-data/dev"  
}
```

### start it
```bash
vault server -config=config/dev.hcl
vault server -config=config/sandbox.hcl
```

- **Key Shares** – total number of unseal keys Vault will generate
- **Key Threshold** – how many of those keys you need to **unseal Vault**

Defaults are usually:
- Shares = 5
- Threshold = 3

# or use docker
dev env XXX.XXXXXXXXXXXXXXXXXXXX
key1 FAKEKEYFAKEKEYFAKEKEYFAKEKEYFAKEKEYFAKEKEYFAKEKEY

sandbox XXX.XXXXXXXXXXXXXXXXXXXXX
token key FAKEKEYFAKEKEYFAKEKEYFAKEKEYFAKEKEYFAKEKEY

# enable the pki engine
`vault secrets enable pki`
`vault secrets tune -max-lease-ttl=87600h pki`

# create a root CA (trust anchor)
```bash
vault write pki/root/generate/internal \  
common_name="sandbox.local" \  
ttl=87600h
```


### or load what we already have by argoCD
```bash
vault write pki/config/ca \  
pem_bundle=@sandbox-root.pem
```

### validate alignment git vs vault
```bash
vault read pki/cert/ca
diff sandbox-root.pem <(vault read -field=certificate pki/cert/ca)
```

# create intermediate CA (per env)
```bash
vault secrets enable -path=pki-sandbox pki -description "CA for sandbox env (istio workload)"
vault secrets tune -max-lease-ttl=43800h pki-sandbox

vault secrets enable -path=pki-dev pki -description "CA for dev env (istio workload)"
vault secrets list
```

## create csr
```
vault write -format=json pki-sandbox/intermediate/generate/internal \  
common_name="sandbox intermediate" \  
| jq -r '.data.csr' > sandbox.csr
```

if run into fish command line problem
```bash
bash -c "vault write -format=json pki-test/intermediate/generate/internal common_name='example.com' | jq -r '.data.csr' > test.csr"
```

## sign it with root
```bash
vault write -format=json pki/root/sign-intermediate \  
csr=@sandbox.csr \  
format=pem_bundle \  
ttl=43800h \  
| jq -r '.data.certificate' > sandbox.pem
```

## set it
```bash
vault write pki-sandbox/intermediate/set-signed \  
certificate=@sandbox.pem
```

# create a role (how certs are issued)
```bash
vault write pki-sandbox/roles/istio \
allowed_domains="svc.cluster.local" \
allow_subdomains=true \
max_ttl="72h"
```

## possible architechture
```bash
Vault  
├── pki-root (trust anchor)  
├── pki-sandbox (intermediate)  
├── pki-staging (intermediate)  
└── pki-prod (intermediate)

Istio  
├── uses Vault for certs  
└── trusts root CA
```
# connect vault to istio

Istio will:
- Request certs from Vault
- Trust Vault’s CA chain

## export the trust anchor to istio
```bash
vault read -field=certificate pki/cert/ca > root.pem
```

# Switching Trust Anchors
option 1
use different CA path
update istio path to it


# integrate into our kubernetes manifest
```bash
run as podman image
docker run \  
--name vault \  
--cap-add=IPC_LOCK \  
-p 8200:8200 \  
hashicorp/vault server -dev
```
## read token
we can even generate a token to read kv or secrets
```bash
`vault token create -policy=app-policy -ttl=1m`
```
