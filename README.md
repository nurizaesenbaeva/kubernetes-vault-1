1)ClusterRoles: Kubernetes ClusterRoles are entities that have been assigned certain special permissions.
2)ServiceAccounts: Kubernetes ServiceAccounts are identities assigned to entities such as pods to enable their interaction with the Kubernetes APIs using the role’s permissions.
3)ClusterRoleBindings: ClusterRoleBindings are entities that provide roles to accounts i.e. they grant permissions to service accounts.
We need to create them first:
 # kubectl apply -f rbac.yaml

4)Creating Vault ConfigMaps.
A ConfigMaps in Kubernetes lets us mount files on containers without the need to make changes to the Dockerfile or rebuilding the container image. 
Create the configmap

# kubectl apply -f configmap.yaml
 


5) Deploy vault services . 
Services in Kubernetes are the objects that pods use to communicate with each other.
ClusterIP type services are usually used for inter-pod communication.

# kubectl apply -f services.yaml

6)StatefulSet .
StatefulSet is the Kubernetes object used to manage stateful applications.
It’s preferred over deployments for this use case as it provides guarantees about the ordering and uniqueness of these Pods i.e. the management of volumes is better with stateful sets.
Vault is a stateful application i.e. it stores data (like configurations, secrets, metadata of vault operations) inside a volume. If the data is stored in memory, then the data will get erased once the pod restarts.

# kubectl apply -f statefulset.yaml

7) Then we need to initialize the vault and save the ouput in a safe place:

# kubectl exec -ti vault-0 -- /bin/sh
# vault operator init


8) unseal the vault and login 

# vault operator unseal ... (3 times)
# vault login ..
# exit


INJECTOR
9) Here we have injector (everything is done in a injector folder).
To create a Service account , clusterrolebinding and cluster role
# kubectl apply -f rbac.yaml
10) creating mutatingwebhookconfiguration...
 This webhook is responsible for intercepting and sending pod events to the injector controller. It sends the webhook to the injector controller’s service endpoint on /mutate path.

# kubectl apply -f mutating-webhook.yaml


11) creating a deployment 
This deployment assumes you have vault server running on default namespace with service endpoint http://vault.default.svc:8200
we are using the  vault injector image hashicorp/vault-k8s:0.11.0
# kubectl apply -f deployment.yaml

12) Create a service.yaml with the following manifest. The mutation webhook will use this service endpoint.
# kubectl apply -f service.yaml



			INJECTING SECRETS FOR APPLICATION
13) Exec into vault pod.
			vault 

# kubectl exec -it vault-0 -- /bin/sh  

14) Enable the vault kv engine (key-value store).

# vault secrets enable -version=2 -path="kv" kv

15) Create two secrets under kv/dev/apps/service01 path. appkey & apptoken

# vault kv put kv/dev/apps/service01 appkey="zsdkfjhj4534" apptoken="zsdasdfaskfjhj4534" 

16)Create a vault policy named svc-policy that allowed read operation on secrets under kv/data/dev/apps/service01 path.

# vault policy write svc-policy - <<EOH
path "kv/data/dev/apps/service01" {
  capabilities = ["read"]
}
EOH

17) Enable Kubernetes authentication.
# vault auth enable kubernetes
# vault write auth/kubernetes/config \
    token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
    kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" \ kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

18) Create a vault role named webapp that binds svc-policy and vault-auth kubernetes service account.

# vault write auth/kubernetes/role/webapp \
        bound_service_account_names=vault-auth \
        bound_service_account_namespaces=default \
        policies=svc-policy \
        ttl=72h

19) after exiting :
 Vault agents will use this service account to authenticate to the vault server and retrieve the required secrets.
# kubectl create serviceaccount vault-auth 


CREATING AN APPLICATION

20) First, we will look at a vault injector example using a simple pod definition with the vault agent annotations and see if the sidecar and init container gets injected into the pods.
It deploys an Nginx container.
# kubectl apply -f pod.yaml

21) Now, let’s exec into the Nginx app container and see if we can access the secret file.

# kubectl exec webapp -c nginx -- cat /vault/secrets/config.txt
You should be getting the following output.

data: map[appkey:zsdkfjhj4534 apptoken:zsdasdfaskfjhj4534]
metadata: map[created_time:2021-08-08T11:29:42.495211138Z deletion_time: destroyed:false version:1]



IN A REAL CASES WE WOULD RATHER USE HELM INSTEAD OF MANUAL WORK :
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update
helm install vault hashicorp/vault
BUT IT WAS FOR EXPLAINING ALL THE STUFF 
