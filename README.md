# eks-config
Configuration for shared EKS clusters
Dynatrace Deployment
Create dynakube secret by fetching the api token from password safe
```
kubectl -n dynatrace create secret generic dynakube --from-literal="apiToken=dt0c01.3DAT44JOUDG2JETVMBRIVQLP.D76JDHJWZ3Y2ZEXWAE27CPHKL3DOSRUY6X5JNDTNQFTRAFYC3QHV4E4EMA5JANAY"
```
## Deployment
1. Apply proxy manifest (https://aws.amazon.com/premiumsupport/knowledge-center/eks-http-proxy-configuration-automation/)
```
kubectl apply -f manifests/kube-system/proxy-environment-variables.yaml
```
2. Patch daemon sets
```
kubectl patch -n kube-system -p '{ "spec": {"template": { "spec": { "containers": [ { "name": "aws-node", "envFrom": [ { "configMapRef": {"name": "proxy-environment-variables"} } ] } ] } } } }' daemonset aws-node
kubectl patch -n kube-system -p '{ "spec": {"template":{ "spec": { "containers": [ { "name": "kube-proxy", "envFrom": [ { "configMapRef": {"name": "proxy-environment-variables"} } ] } ] } } } }' daemonset kube-proxy
```
3. Apply Calico manifest
```
kubectl apply -f manifests/kube-system/calico.yaml 
```
4. Add a secret containing the S3 credentials for Thanos
```
kubectl apply -f manifests/prometheus/namespace.yaml
echo "type: s3" > object-store.yaml
echo "config:" >> object-store.yaml
echo "  bucket: <S3 bucket>" >> object-store.yaml
echo "  endpoint: s3.eu-west-2.amazonaws.com" >> object-store.yaml
echo "  access_key: <Access key>" >> object-store.yaml
echo "  secret_key: <Secret key>" >> object-store.yaml
kubectl -n prometheus create secret generic thanos-s3-config --from-file=object-store.yaml
rm object-store.yaml
```
5. Add a secret containing the image pull secret for Artifactory:
```
kubectl apply -f manifests/jenkins/namespace.yaml
kubectl -n jenkins create secret docker-registry artifactory-rackspace --docker-server='docker.artifactory.mgmt.aws.uk.tsb' --docker-username='rackspace' --docker-password='<PASSWORD>'
```
6. Install Flux and the Helm Operator using Helm
```
AWS_ACCOUNT=<AWS account number>
sed -i -e "s/{{ AWS_ACCOUNT }}/$AWS_ACCOUNT/g" helm/flux.yaml
kubectl apply -f manifests/flux/namespace.yaml
kubectl apply -f manifests/flux/kustomize.yaml
kubectl -n flux create secret generic flux-git-auth --from-literal GIT_AUTHKEY=<GitHub authentication token> --from-literal GIT_AUTHUSER=<GitHub user>
helm repo add fluxcd https://charts.fluxcd.io
helm install flux fluxcd/flux --namespace flux --values helm/flux.yaml --version 1.5.0 --set git.branch=<dev|nonprod|preprod|master>
helm install helm-operator fluxcd/helm-operator --namespace flux --values helm/helm.yaml --version 1.2.0
sed -i -e "s/$AWS_ACCOUNT/{{ AWS_ACCOUNT }}/g" helm/flux.yaml
```