git clone https://github.com/techiescamp/kubernetes-vault.git

kubectl apply -f rbac.yaml

kubectl apply -f configmap.yaml

kubectl apply -f services.yaml

kubectl apply -f statefulset.yaml

kubectl exec vault-0 -- vault operator init -key-shares=1 -key-threshold=1 -format=json > keys.json

sudo apt install jq

VAULT_UNSEAL_KEY=$(cat keys.json | jq -r ".unseal_keys_b64[]")
echo $VAULT_UNSEAL_KEY

VAULT_ROOT_KEY=$(cat keys.json | jq -r ".root_token")
echo $VAULT_ROOT_KEY

kubectl exec vault-0 -- vault operator unseal $VAULT_UNSEAL_KEY

VAULT_ROOT_KEY=%KEY%

kubectl exec vault-0 -- vault login $VAULT_ROOT_KEY

kubectl port-forward service/vault 8200:8200

http://localhost:8200

kubectl exec -it vault-0 -- /bin/sh

vault secrets enable -version=2 -path="demo-app" kv

vault kv put demo-app/user01 name=devopscube
vault kv get demo-app/user01 

vault policy write demo-policy - <<EOH
path "demo-app/*" {
  capabilities = ["read"]
}
EOH

vault policy list

vault auth enable kubernetes

vault write auth/kubernetes/config \
  token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
  kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

kubectl create serviceaccount vault-auth

vault write auth/kubernetes/role/webapp \
        bound_service_account_names=vault-auth \
        bound_service_account_namespaces=default \
        policies=demo-policy \
        ttl=72h

kubectl apply -f pod.yaml

kubectl exec -it vault-client -- /bin/bash

apt-get update && apt-get install -y jq

jwt_token=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)

curl --request POST \
    --data '{"jwt": "'$jwt_token'", "role": "webapp"}' \
    http://10.109.12.9:8200/v1/auth/kubernetes/login | jq

curl -H "X-Vault-Token: %TOKEN%"      -H "X-Vault-Namespace: vault"      -X GET http://10.109.12.9:8200/v1/demo-app/data/user01?version=1 | jq

http://localhost:8200/v1/sys/init


