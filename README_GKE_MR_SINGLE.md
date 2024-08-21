## Gloo Edge Federation (dedicated admin cluster) on GKE clusters and Manual Registration

Install glooctl:
```sh
curl -sL https://run.solo.io/gloo/install | sh
export PATH=$HOME/.gloo/bin:$PATH
```

Create 2 GKE clusters:
```sh
export PROJECT_ID=customer-success-386314
export REGION=us-west1
export ZONE1=us-west1-a
export ZONE2=us-west1-b
export VPC_NAME=glau-solo-vpc
export SUBNET_NAME=glau-solo-vpc-subnet1
export MGMT_CLUSTER=glau-solo-mgmt-cluster
export CLUSTER1=glau-solo-cluster1

gcloud compute networks create $VPC_NAME \
    --subnet-mode=custom \
    --bgp-routing-mode=regional

gcloud compute networks subnets create $SUBNET_NAME \
    --network=$VPC_NAME \
    --region=$REGION \
    --range=10.0.1.0/24

gcloud container clusters create $MGMT_CLUSTER \
  --zone=$ZONE1 --num-nodes=1 \
  --image-type=COS_CONTAINERD \
  --machine-type=e2-standard-4 \
  --network=$VPC_NAME \
  --subnetwork=$SUBNET_NAME \
  --enable-ip-alias \
  --logging=SYSTEM,WORKLOAD \
  --labels=created-by=gilbert_lau,team=customer-success,purpose=poc

gcloud container clusters create $CLUSTER1 \
  --zone=$ZONE2 --num-nodes=1 \
  --image-type=COS_CONTAINERD \
  --machine-type=e2-standard-4 \
  --network=$VPC_NAME \
  --subnetwork=$SUBNET_NAME \
  --enable-ip-alias \
  --logging=SYSTEM,WORKLOAD \
  --labels=created-by=gilbert_lau,team=customer-success,purpose=poc
```

Grab GKE cluster contexts:
```sh
gcloud container clusters get-credentials $MGMT_CLUSTER --zone $ZONE1 --project $PROJECT_ID
export MGMT_CONTEXT=$(kubectl config current-context)

gcloud container clusters get-credentials $CLUSTER1 --zone $ZONE2 --project $PROJECT_ID
export CLUSTER1_CONTEXT=$(kubectl config current-context)
```

View clusters:
```
kubectl get nodes --context $MGMT_CONTEXT
kubectl get nodes --context $CLUSTER1_CONTEXT
```

Install Gloo Gateway Federation on the management cluster:
```sh
export GLOO_VERSION=1.17.0-rc5
```

```sh
helm repo add gloo-fed https://storage.googleapis.com/gloo-fed-helm
helm repo update
```
```sh
helm install gloo-fed gloo-fed/gloo-fed --version $GLOO_VERSION --set license_key=$GLOO_LICENSE_KEY -n gloo-system --create-namespace --kube-context $MGMT_CONTEXT
```
View status:
```sh
kubectl get all -n gloo-system --context $MGMT_CONTEXT
```

Install Gloo Gateway Enterprise on the workload clusters:
```sh
helm repo add glooe https://storage.googleapis.com/gloo-ee-helm
```
```sh
helm install gloo glooe/gloo-ee --namespace gloo-system \
  --create-namespace --set-string license_key=$GLOO_LICENSE_KEY \
  --version $GLOO_VERSION --kube-context $CLUSTER1_CONTEXT
```



Manual Registration of workload cluster:    
    

Build kubeconfig yaml for the workload cluster from ~/.kube/config file


    
```sh
kubectl --context $CLUSTER1_CONTEXT -n gloo-system create sa $CLUSTER1
token=$(kubectl create token $CLUSTER1 -n gloo-system --context $CLUSTER1_CONTEXT)
kubectl create secret generic $CLUSTER1 -n gloo-system --context $CLUSTER1_CONTEXT --from-literal=token="$token"

kubectl --context $CLUSTER1_CONTEXT -n gloo-system apply -f - <<EOF
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
  name: ${CLUSTER1}-gloo-federation-controller-clusterrole-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: gloo-federation-controller
subjects:
- kind: ServiceAccount
  name: $CLUSTER1
  namespace: gloo-system
EOF

  cat > kc-${CLUSTER1}.yaml <<EOF1
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: $(eval kubectl config view --raw | yq ".clusters[] | select(.name == \"$CLUSTER1_CONTEXT\") | .cluster.\"certificate-authority-data\"")
    server: $(eval kubectl config view --raw | yq ".clusters[] | select(.name == \"$CLUSTER1_CONTEXT\") | .cluster.server")
  name: $CLUSTER1_CONTEXT
contexts:
- context: 
    cluster: $CLUSTER1_CONTEXT
    user: $CLUSTER1_CONTEXT
  name: $CLUSTER1_CONTEXT
current-context: $CLUSTER1_CONTEXT
kind: Config
preferences: {}
users:
- name: $CLUSTER1_CONTEXT
  user:
    token: $token
EOF1
  
  export cluster_name=$(echo $CLUSTER1 | tr '-' '_')
  export kc_${cluster_name}=$(cat kc-${CLUSTER1}.yaml | base64)
  kubectl --context $MGMT_CONTEXT apply -f - <<EOF2
---
apiVersion: multicluster.solo.io/v1alpha1
kind: KubernetesCluster
metadata:
  name: $CLUSTER1
  namespace: gloo-system
spec:
  clusterDomain: cluster.local
  secretName: $CLUSTER1
---
apiVersion: v1
kind: Secret
metadata:
  name: $CLUSTER1
  namespace: gloo-system
type: solo.io/kubeconfig
data:
  kubeconfig: $(eval "echo \${kc_${cluster_name}}") 
EOF2
```

Verify discovered Gloo Gateway instances:
```sh
kubectl get glooinstances -n gloo-system --context $MGMT_CONTEXT
```

Deploy a sample app:
```sh
kubectl create ns httpbin --context $CLUSTER1_CONTEXT
kubectl -n httpbin apply -f https://raw.githubusercontent.com/solo-io/gloo-mesh-use-cases/main/policy-demo/httpbin.yaml --context $CLUSTER1_CONTEXT
kubectl apply -f ./data/upstream.yaml --context $CLUSTER1_CONTEXT
kubectl apply -f ./data/authconfig.yaml --context $CLUSTER1_CONTEXT
kubectl apply -f ./data/virtualservice.yaml --context $CLUSTER1_CONTEXT
kubectl apply -f ./data/routetable-parent.yaml --context $CLUSTER1_CONTEXT
kubectl apply -f ./data/routetable-child.yaml --context $CLUSTER1_CONTEXT
```

View Read-only Console:
```sh
kubectl port-forward svc/gloo-fed-console -n gloo-system 8090:8090 --context $MGMT_CONTEXT
```
```
http://localhost:8090
```

Tear down the environment:
```sh
gcloud container clusters delete $MGMT_CLUSTER --zone $ZONE1 --quiet 
gcloud container clusters delete $CLUSTER1 --zone $ZONE2 --quiet
gcloud compute networks subnets delete $SUBNET_NAME --region=$REGION --quiet
gcloud compute networks delete $VPC_NAME --quiet
```

