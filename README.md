# Steps

### Start minikube
 * `minikube start --kubernetes-version v1.13.3`

### Start vault on the cluster
 * `kubectl apply -f vault-deployment.yml`

### Create the service to access our vault deployment
 * `kubectl apply -f vault-service.yml`

### Get the Vault root token
 * `kubectl logs <vault-pod-name> | grep "Root Token"`

### Expose the vault url to the host
 * `export VAULT_ADDR=$(minikube service --url vault)`

### Log in to vault
 * `vault login -method=token <ROOT TOKEN>`

### Disable secrets engine (It is kv2 by default and we'd like kv1)
 * `vault secrets disable secret`

### Enable kv1 at secret path
 * `vault secrets enable -path=secret kv`

### Create service account and tokenreviewer binding
 * `kubectl apply -f vault-service-account.yml`

### Create avatao-readonly policy
 * `vault policy write avatao-readonly avatao-readonly.hcl`

### Create some secret
 * `vault kv put secret/avatao/my-secret username="avatao" password="secret" ttl='10s'`

### Enable userpass
 * `vault auth enable userpass`

### Create test user
 * `vault write auth/userpass/users/test-user password="test" policies="avatao-readonly"`

### Login with the test user
 * `vault login -method=userpass username="test-user" password="test"`

### Test whether you can see the secret
 * `vault kv get secret/avatao/my-secret`

### Log back with admin
 * `vault login -method=token <ROOT_TOKEN>`

### Get information from the cluster to setup kubernetes auth
```shell
# Get the service account token name and save it to SA_NAME
export SA_NAME=$(kubectl get sa vault -o jsonpath="{.secrets[*]['name']}")

# Set SA_JWT_TOKEN value to the service account JWT used to access the TokenReview API
export SA_JWT_TOKEN=$(kubectl get secret $SA_NAME -o jsonpath="{.data.token}" | base64 --decode; echo)

# Set SA_CA_CRT to the PEM encoded CA cert used to talk to Kubernetes API
export SA_CA_CRT=$(kubectl get secret $SA_NAME -o jsonpath="{.data['ca\.crt']}" | base64 --decode; echo)

# Set K8S_HOST to minikube IP address
export K8S_HOST=$(minikube ip)
```

### Enable kubernetes auth
 * `vault auth enable kubernetes`

### Configure kubernetes on vault side
```
vault write auth/kubernetes/config \
        token_reviewer_jwt="$SA_JWT_TOKEN" \
        kubernetes_host="https://$K8S_HOST:8443" \
        kubernetes_ca_cert="$SA_CA_CRT"
```

### Create a role with our avatao-readonly policy
```
vault write auth/kubernetes/role/avatao-reader \
        bound_service_account_names=vault \
        bound_service_account_namespaces=default \
        policies=avatao-readonly \
        period=24h
```

### Test our setup
* `kubectl run --generator=run-pod/v1 tmp --rm -i --tty --serviceaccount=vault --image alpine:3.7`

```
apk update
apk add curl jq
VAULT_ADDR=http://vault:8200
curl -s $VAULT_ADDR/v1/sys/health | jq
KUBE_TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
curl --request POST --data '{"jwt": "'"$KUBE_TOKEN"'", "role": "avatao-reader"}' $VAULT_ADDR/v1/auth/kubernetes/login | jq
```

### Create vault-agent and consul-template configuration
* `kubectl apply -f my-app/my-app-configmap.yml`

### Bring up the my-app app and a service
* `kubectl apply -f my-app/`

### Open the my-app app and examine the output
* `minikube service my-app`

### Change the secret in vault and see the results
Be careful to add the ttl parameter to tell consul to poll the value often
* `vault kv put secret/avatao/my-secret username="admin" password="verysecretmuchpassword" ttl="5s"`

