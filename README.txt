
## -Namespace
#   infrastructure
#   developer

## -Roles
#   infra-role
#   dev-role

## -Users
#   infra-user
#   dev-user

## -Rolebinding
#   infra-rolebinding
#   dev-rolebinding



Recreate
========
#create 2 namespaces, infrastructure and developer
kubectl create ns infrastructure
kubectl create ns developer

#pull some test docker images to use
docker pull alpine
docker pull hello-world
docker images -a

#deploy a diff pod to each NS
kubectl -n infrastructure create deploy infra-alpine --image alpine
kubectl -n developer create deploy dev-hello-world --image hello-world
kubectl get deploy -A

#copy the ca.crt and ca.key for the cluster to the working directory
cp /home/jamesb/.minikube/ca.{crt,key} .

#create the infra-user as part of the infrastructure group
openssl genrsa -out infra-user.key 2048
openssl req -new -key infra-user.key -out infra-user.csr -subj "/CN=infra-user/O=infrastructure"
#sign infra-user certificate with the CA for a year
openssl x509 -req -in infra-user.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out infra-user.crt -days 365

#create the dev-user as part of the developer group
openssl genrsa -out dev-user.key 2048
openssl req -new -key dev-user.key -out dev-user.csr -subj "/CN=dev-user/O=developer"
#sign infra-user certificate with the CA for a year
openssl x509 -req -in dev-user.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out dev-user.crt -days 365

#create a kubeconfig for infra-user to use
>>> file: infra-user.kubeconfig
apiVersion: v1
clusters:
- cluster:
    certificate-authority: /home/jamesb/.minikube/ca.crt
    extensions:
    - extension:
        last-update: Mon, 06 Sep 2021 15:15:18 SAST
        provider: minikube.sigs.k8s.io
        version: v1.22.0
      name: cluster_info
    server: https://192.168.49.2:8443
  name: minikube
contexts:
- context:
    cluster: minikube
    extensions:
    - extension:
        last-update: Mon, 06 Sep 2021 15:15:18 SAST
        provider: minikube.sigs.k8s.io
        version: v1.22.0
      name: context_info
    namespace: infrastructure
    user: infra-user
  name: infra-user-kubernetes
current-context: infra-user-kubernetes
kind: Config
preferences: {}
users:
- name: infra-user
  user:
    client-certificate: /home/jamesb/Documents/Projects/rbac_example/infra-user.crt
    client-key: /home/jamesb/Documents/Projects/rbac_example/infra-user.key

#create a kubeconfig for dev-user to use
>>> file: dev-user.kubeconfig
apiVersion: v1
clusters:
- cluster:
    certificate-authority: /home/jamesb/.minikube/ca.crt
    extensions:
    - extension:
        last-update: Mon, 06 Sep 2021 15:15:18 SAST
        provider: minikube.sigs.k8s.io
        version: v1.22.0
      name: cluster_info
    server: https://192.168.49.2:8443
  name: minikube
contexts:
- context:
    cluster: minikube
    extensions:
    - extension:
        last-update: Mon, 06 Sep 2021 15:15:18 SAST
        provider: minikube.sigs.k8s.io
        version: v1.22.0
      name: context_info
    namespace: developer
    user: dev-user
  name: dev-user-kubernetes
current-context: dev-user-kubernetes
kind: Config
preferences: {}
users:
- name: dev-user
  user:
    client-certificate: /home/jamesb/Documents/Projects/rbac_example/dev-user.crt
    client-key: /home/jamesb/Documents/Projects/rbac_example/dev-user.key

#create a role infra-role that can get and list pods from ns infrastructure
kubectl create role infra-role --verb=get,list --resource=pods --namespace infrastructure

#create a role dev-role that can get and list pods from ns infrastructure
kubectl create role dev-role --verb=get,list --resource=pods --namespace developer

#insert magic here :)

#create a new rolebinding for developer
kubectl create rolebinding dev-rolebinding --role=dev-role --group=developer --namespace developer

#create a new rolebinding for infrastructure
kubectl create rolebinding infra-rolebinding --role=infra-role --group=infrastructure --namespace infrastructure

#create a new rolebinding for infrastructure to access developer
kubectl create rolebinding infra-dev-rolebinding --role=dev-role --group=infrastructure --namespace developer




#test
###DEVELOPER
kubectl --kubeconfig=dev-user.kubeconfig get pods -n developer
kubectl --kubeconfig=dev-user.kubeconfig get pods -n infrastructure

###INFRASTRUCTURE
kubectl --kubeconfig=infra-user.kubeconfig get pods -n developer
kubectl --kubeconfig=infra-user.kubeconfig get pods -n infrastructure