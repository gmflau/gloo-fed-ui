## Gloo Edge Federation (dedicated cluster) on Kind clusters and Manual Registration

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

# hostport for 1st workload: 7002
./scripts/deploy.sh 2 $CLUSTER1
export ${CLUSTER1}_host_port=7002

# hostport for 2nd workload: 7003
./scripts/deploy.sh 3 $CLUSTER2
export ${CLUSTER2}_host_port=7003
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

Manual registration of workload clusters:
```sh
clusters=($CLUSTER1 $CLUSTER2) 
for CLUSTER in "${clusters[@]}"; do
  kubectl --context $CLUSTER -n gloo-system create sa $CLUSTER
  token=$(kubectl create token $CLUSTER -n gloo-system --context $CLUSTER)
  kubectl create secret generic $CLUSTER -n gloo-system --context $CLUSTER --from-literal=token="$token"

  kubectl --context $CLUSTER -n gloo-system apply -f - <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: gloo-federation-controller
rules:
- apiGroups:
  - gloo.solo.io
  - gateway.solo.io
  - enterprise.gloo.solo.io
  - ratelimit.solo.io
  - graphql.gloo.solo.io
  resources:
  - '*'
  verbs:
  - '*'
- apiGroups:
  - apps
  resources:
  - deployments
  - daemonsets
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - pods
  - nodes
  - services
  verbs:
  - get
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: ${CLUSTER}-gloo-federation-controller-clusterrole-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: gloo-federation-controller
subjects:
- kind: ServiceAccount
  name: $CLUSTER
  namespace: gloo-system
EOF

  cat > kc-${CLUSTER}.yaml <<EOF1
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: "" # DO NOT empty the value here unless you are using KinD
    server: https://host.docker.internal:$(eval "echo \${${CLUSTER}_host_port}") # enter the api-server address of the target cluster
    insecure-skip-tls-verify: true # DO NOT use this unless you know what you are doing
  name: $CLUSTER
contexts:
- context:
    cluster: $CLUSTER
    user: $CLUSTER
  name: $CLUSTER
current-context: $CLUSTER
kind: Config
preferences: {}
users:
- name: $CLUSTER
  user:
    token: $token
EOF1

  export kc_${CLUSTER}=$(cat kc-${CLUSTER}.yaml | base64)
  kubectl --context $MGMT apply -f - <<EOF2
---
apiVersion: multicluster.solo.io/v1alpha1
kind: KubernetesCluster
metadata:
  name: $CLUSTER
  namespace: gloo-system
spec:
  clusterDomain: cluster.local
  secretName: $CLUSTER
---
apiVersion: v1
kind: Secret
metadata:
  name: $CLUSTER
  namespace: gloo-system
type: solo.io/kubeconfig
data:
  kubeconfig: $(eval "echo \${kc_${CLUSTER}}") 
EOF2
  
done
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