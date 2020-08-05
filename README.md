# eks-config
Configuration for shared EKS clusters

## Deployment
1. Ensure that EKS is deployed witht the IAM policy `eks-iam.json` applied.
2. Apply proxy manifest (https://aws.amazon.com/premiumsupport/knowledge-center/eks-http-proxy-configuration-automation/)
```
kubectl apply -f manifests/kube-system/proxy-environment-variables.yaml
```
3. Patch daemon sets
```
kubectl patch -n kube-system -p '{ "spec": {"template": { "spec": { "containers": [ { "name": "aws-node", "envFrom": [ { "configMapRef": {"name": "proxy-environment-variables"} } ] } ] } } } }' daemonset aws-node
kubectl patch -n kube-system -p '{ "spec": {"template":{ "spec": { "containers": [ { "name": "kube-proxy", "envFrom": [ { "configMapRef": {"name": "proxy-environment-variables"} } ] } ] } } } }' daemonset kube-proxy
```
4. Apply EBS manifest
```
kubectl apply -f manifests/kube-system/ebs.yaml
```
5. Install Flux and the Helm Operator using Helm
```
kubectl apply -f manifests/flux/namespace.yaml
kubectl apply -f manifests/flux/kustomize.yaml
kubectl -n flux create secret generic flux-git-auth --from-literal GIT_AUTHKEY=<GitHub authentication token> --from-literal GIT_AUTHUSER=<GitHub user>
helm repo add fluxcd https://charts.fluxcd.io
helm install flux fluxcd/flux --namespace flux --values helm/flux.yaml --set git.branch=<dev|nonprod|preprod|master>
helm install helm-operator fluxcd/helm-operator -namespace flux --values helm/helm.yaml
``