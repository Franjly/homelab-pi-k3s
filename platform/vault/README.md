# vault
##### Init vault
```sh
kubectl exec -ti vault-0 -n vault -- vault operator init
```
Unseal (3x times)
```sh
kubectl exec -ti vault-0 -n vault -- vault operator unseal {{ unseal_key }}
```
##### CLI Login
```sh
export VAULT_ADDR=https://vault.local.tecno-fly.com/
export VAULT_SKIP_VERIFY=true

vault login
```
##### Enable k8s auth
```sh
vault auth enable kubernetes

vault write auth/kubernetes/config \
    token_reviewer_jwt="`kubectl get secret argocd-repo-server -n argocd -o go-template='{{ .data.token }}' | base64 --decode`" \
    kubernetes_host="`kubectl config view --raw --minify --flatten --output='jsonpath={.clusters[].cluster.server}'`" \
    kubernetes_ca_cert="`kubectl config view --raw --minify --flatten -o jsonpath='{.clusters[].cluster.certificate-authority-data}' | base64 --decode`" \
    disable_local_ca_jwt="true"
```
##### Create role & policy
```sh
vault policy write argocd - <<EOF
path "pi-k3s/*" {
  capabilities = ["read"]
}
EOF

vault write auth/kubernetes/role/argocd \
  bound_service_account_names=argocd-repo-server \
  bound_service_account_namespaces=argocd \
  policies=argocd \
  ttl=24h
```