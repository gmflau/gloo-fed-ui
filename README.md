## Gloo Edge Federation (dedicated cluster) on Kind clusters

Install glooctl:
```sh
curl -sL https://run.solo.io/gloo/install | sh
export PATH=$HOME/.gloo/bin:$PATH
```

Create 2 kind clusters:
```sh
export MGMT=mgmt
export CLUSTER1=cluster1
export CLUSTER2=cluster2
```
```sh
./scripts/deploy.sh 1 $MGMT
./scripts/deploy.sh 2 $CLUSTER1
./scripts/deploy.sh 3 $CLUSTER2
```
View clusters:
```
kubectl get nodes --context $MGMT
kubectl get nodes --context $CLUSTER1
kubectl get nodes --context $CLUSTER2
```

Install Gloo Gateway Federation on management cluster:
```sh
export GLOO_VERSION=1.17.0-rc5
```
```sh
helm repo add gloo-fed https://storage.googleapis.com/gloo-fed-helm
helm repo update
```
```sh
helm install gloo-fed gloo-fed/gloo-fed --version $GLOO_VERSION --set license_key=$GLOO_LICENSE_KEY -n gloo-system --create-namespace --kube-context $MGMT
```
View status:
```sh
kubectl get all -n gloo-system --context $MGMT
```

Install Gloo Gateway Enterprise on workload clusters:
```sh
helm repo add glooe https://storage.googleapis.com/gloo-ee-helm
```
```sh
helm install gloo glooe/gloo-ee --namespace gloo-system \
  --create-namespace --set-string license_key=$GLOO_LICENSE_KEY \
  --version $GLOO_VERSION --kube-context $CLUSTER1

helm install gloo glooe/gloo-ee --namespace gloo-system \
  --create-namespace --set-string license_key=$GLOO_LICENSE_KEY \
  --version $GLOO_VERSION --kube-context $CLUSTER2
```

Register workload clusters    
Set context to $MGMT cluster:
```sh
kubectl config use-context $MGMT
```
```sh
glooctl cluster register --cluster-name $CLUSTER1 --remote-context $CLUSTER1 --local-cluster-domain-override host.docker.internal --federation-namespace gloo-system
glooctl cluster register --cluster-name $CLUSTER2 --remote-context $CLUSTER2 --local-cluster-domain-override host.docker.internal --federation-namespace gloo-system
```

Verify registration:
```sh
kubectl get secret -n gloo-system $CLUSTER1-secret --context $CLUSTER1
kubectl get secret -n gloo-system $CLUSTER2-secret --context $CLUSTER2
```
```sh
kubectl --context $CLUSTER1 get serviceaccount $CLUSTER1 -n gloo-system
kubectl --context $CLUSTER2 get serviceaccount $CLUSTER2 -n gloo-system
```
```sh
kubectl --context $CLUSTER1 get clusterrole gloo-federation-controller
kubectl --context $CLUSTER2 get clusterrole gloo-federation-controller
```
```sh
kubectl --context $CLUSTER1 get clusterrolebinding \
  $CLUSTER1-gloo-federation-controller-clusterrole-binding
kubectl --context $CLUSTER2 get clusterrolebinding \
  $CLUSTER2-gloo-federation-controller-clusterrole-binding
```
Verify discovered Gloo Gateway instances:
```sh
kubectl get glooinstances -n gloo-system --context $MGMT
```
    
Deploy a sample app:
```sh
kubectl create ns httpbin --context $CLUSTER1
kubectl -n httpbin apply -f https://raw.githubusercontent.com/solo-io/gloo-mesh-use-cases/main/policy-demo/httpbin.yaml --context $CLUSTER1
kubectl apply -f ./data/upstream.yaml --context $CLUSTER1
kubectl apply -f ./data/authconfig.yaml --context $CLUSTER1
kubectl apply -f ./data/virtualservice.yaml --context $CLUSTER1
kubectl apply -f ./data/routetable-parent.yaml --context $CLUSTER1
kubectl apply -f ./data/routetable-child.yaml --context $CLUSTER1
```
    
View Read-only Console:
```sh
kubectl port-forward svc/gloo-fed-console -n gloo-system 8090:8090 --context $MGMT
```
```
http://localhost:8090
```

Tear down the environment:
```sh
kind delete cluster --name kind1 
kind delete cluster --name kind2
kind delete cluster --name kind3
```