# k8s-vault-secret
## Vault secrets for Kubernetes applications

1. Running consul and vault
2. Create Mutating and Validating Web hooks to intercept the deployment
3. Communicate to vault to get the secrets and replace env

## Requirements:
1. Working K8s cluster
2. Kubectl & Helm
3. Go 
    ```
    $ brew update
    $ brew install go
    $ mkdir $HOME/go
    $ export GOPATH=$HOME/go
    $ export PATH=$PATH:$GOPATH/bin
    ```
4. SSL toolkit form cloudflare
    ```
    $ go get -u github.com/cloudflare/cfssl/cmd/cfssl
    $ go get -u github.com/cloudflare/cfssl/cmd/cfssljson
    ```
5. Consul & Vault
    ```
    $ brew install consul
    $ brew install vault
    ```
6. .
    ├── certs
    │   ├── ca-key.pem
    │   ├── ca.csr
    │   ├── ca.pem
    │   ├── config
    │   │   ├── ca-config.json
    │   │   ├── ca-csr.json
    │   │   ├── consul-csr.json
    │   │   └── vault-csr.json
    │   ├── consul-key.pem
    │   ├── consul.csr
    │   ├── consul.pem
    │   ├── vault-key.pem
    │   ├── vault.csr
    │   └── vault.pem
    ├── consul
    │   ├── config.json
    │   ├── service.yaml
    │   └── statefulset.yaml
    └── vault
        ├── config.json
        ├── deployment.yaml
        └── service.json

## Running consul and vault

### Local Certificate Authority
```
$ cfssl gencert -initca certs/config/ca-csr.json | cfssljson -bare certs/ca
```

### Generate consul certs
```
$ cfssl gencert \
    -ca=certs/ca.pem \
    -ca-key=certs/ca-key.pem \
    -config=certs/config/ca-config.json \
    -profile=default \
    certs/config/consul-csr.json | cfssljson -bare certs/consul
```

### Generate vault certs
```
$ cfssl gencert \
    -ca=certs/ca.pem \
    -ca-key=certs/ca-key.pem \
    -config=certs/config/ca-config.json \
    -profile=default \
    certs/config/vault-csr.json | cfssljson -bare certs/vault
```

### Consul setup
#### secrets & configmaps
```
$ export GOSSIP_ENCRYPTION_KEY=$(consul keygen)
$ kubectl create secret generic consul \
  --from-literal="gossip-encryption-key=${GOSSIP_ENCRYPTION_KEY}" \
  --from-file=certs/ca.pem \
  --from-file=certs/consul.pem \
  --from-file=certs/consul-key.pem

$ kubectl create configmap consul --from-file=consul/config.json
```

#### deploy consul
```
$ kubectl create -f consul/service.yaml
$ kubectl create -f consul/statefulset.yaml
```
#### vefrification
```
$ kubectl port-forward consul-1 8500:8500

$ consul members
```

### Vault setup
#### secrets & configmaps
```
$ kubectl create secret generic vault \
    --from-file=certs/ca.pem \
    --from-file=certs/vault.pem \
    --from-file=certs/vault-key.pem

$ kubectl create configmap vault --from-file=vault/config.json
```

#### deploy vault
```
$ kubectl create -f vault/service.yaml
$ kubectl apply -f vault/deployment.yaml
```

#### verification
```
$ kubectl port-forward vault-xxxxxxxx-xxxxxx 8200:8200

$ export VAULT_ADDR=https://127.0.0.1:8200
$ export VAULT_CACERT="certs/ca.pem"

$ vault operator init -key-shares=1 -key-threshold=1
$ vault operator unseal

$ vault status
$ vault login
$ vault kv put secret/test/vault/auto/secret AWS_SECRET_ACCESS_KEY=sssshhhhIAMSecret
$ vault kv get secret/test/vault/auto/secret
```

#### enable k8s auth backend
```
$ vault auth enable kubernetes
```
#### vault policies
```
$ vault policy write test_policy -<<EOF
path "secret/test/vault/auto/secret" {
  capabilities = ["read"]
}
EOF

$ vault write auth/kubernetes/role/tester \
    bound_service_account_names=tester \
    bound_service_account_namespaces=default \
    policies=test_policy \
    ttl=1h
```

## Create Mutating and Validating Web hooks to intercept the deployment

```
$ export WEBHOOK_NS=`<namepsace>`
$ WEBHOOK_NS=${WEBHOOK_NS:-vault-secrets-webhook}
$ echo kubectl create namespace "${WEBHOOK_NS}"
$ echo kubectl label ns "${WEBHOOK_NS}" name="${WEBHOOK_NS}"
$ cd kubernetes-mutation-webhook-vault-secrets
$ helm upgrade --namespace "${WEBHOOK_NS}" --install vault-secrets-webhook helm-chart
$ cd ..
```

## Communicate to vault to get the secrets and replace env
```
$ kubectl create secret vault-consul-ca --from-file=certs/ca-key.pem --from-file=certs/ca.pem
$ kubectl apply -f app/tester-sa.yaml
$ kubectl apply -f app/deployment.yaml
```

## verification
```
check the logs for the deployment.
```