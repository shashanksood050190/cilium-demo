# Cilium Setup
## Create the Cluster on GCP
```bash
#Create the node pool with the following taint to guarantee that
#Pods are only scheduled in the node when Cilium is ready.

gcloud container clusters create "cilium-demo" \
--node-taints node.cilium.io/agent-not-ready=true:NoSchedule \
--zone us-west2-a
    
gcloud container clusters get-credentials "cilium-demo" --zone      us-west2-a

kubectl get nodes
```
## Install the Cilium CLI
```bash
curl -L --remote-name-all https://github.com/cilium/cilium-cli/releases/latest/download/cilium-linux-amd64.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-amd64.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-amd64.tar.gz /usr/local/bin
rm cilium-linux-amd64.tar.gz{,.sha256sum}
```
## Install Cilium
```bash
#Setup Helm repository:
helm repo add cilium https://helm.cilium.io/

export NAME=cilium-demo
export ZONE=us-west2-a

NATIVE_CIDR="$(gcloud container clusters describe "${NAME}" --zone "${ZONE}" --format 'value(clusterIpv4Cidr)')"
echo $NATIVE_CIDR

helm install cilium cilium/cilium --version 1.10.3 \
  --namespace kube-system \
  --set nodeinit.enabled=true \
  --set nodeinit.reconfigureKubelet=true \
  --set nodeinit.removeCbrBridge=true \
  --set cni.binPath=/home/kubernetes/bin \
  --set gke.enabled=true \
  --set ipam.mode=kubernetes \
  --set nativeRoutingCIDR=$NATIVE_CIDR
```
## Validate the Installation
```bash
cilium status --wait

kubectl get pods -n kube-system
```
## Deploy the demo app
```bash
kubectl create -f https://raw.githubusercontent.com/cilium/cilium/1.10.3/examples/minikube/http-sw-app.yaml

#Kubernetes will deploy the pods and service in the background. Running kubectl get pods,svc will inform you about the progress of the operation.
 
kubectl get pods,svc

#Each pod of our application will be represented as an Endpoint. We can invoke the cilium tool inside the Cilium pod to list them:
 
kubectl -n kube-system get pods -l k8s-app=cilium
kubectl -n kube-system exec cilium-1c2cz -- cilium endpoint list
```
## Check the current access
```bash
#From the perspective of the deathstar service, only the ships with label org=empire are allowed to connect and request landing. Since we have no rules enforced, both xwing #and tiefighter will be able to request landing. To test this, use the commands below.

kubectl exec xwing -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing

kubectl exec tiefighter -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
```
# Setting up Hubble Observability
```bash
helm upgrade cilium cilium/cilium --version 1.10.3 \
   --namespace kube-system \
   --reuse-values \
   --set hubble.relay.enabled=true \
   --set hubble.ui.enabled=true
   
#Install the Hubble Client

export HUBBLE_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/hubble/master/stable.txt)
curl -L --remote-name-all https://github.com/cilium/hubble/releases/download/$HUBBLE_VERSION/hubble-linux-amd64.tar.gz{,.sha256sum}
sha256sum --check hubble-linux-amd64.tar.gz.sha256sum
sudo tar xzvfC hubble-linux-amd64.tar.gz /usr/local/bin
rm hubble-linux-amd64.tar.gz{,.sha256sum}
```
## Validate the Hubble API access

```bash
#In order to access the Hubble API, create a port forward to the Hubble service from your local machine. 
#This allows us to connect the Hubble client to the local port 4245 and access the Hubble Relay service in your Kubernetes cluster.

Kubectl get pods -n kube-system
cilium hubble port-forward&

#Validate that you can access the Hubble API via the installed CLI
hubble status
```
## Inspecting network flows with the CLI
```bash
kubectl exec xwing -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
Ship landed

kubectl exec tiefighter -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing
Ship landed

hubble observe -n default --since=1m
```
Apply an L3/L4 Policy
```bash
#We can use the labels assigned to the pods to define security policies
#The policies will be applied to the right pods based on the labels irrespective of where or when it is running within the cluster.
#We’ll apply the basic policy restricting deathstar landing requests to only the ships that have label (org=empire)

kubectl create -f https://raw.githubusercontent.com/cilium/cilium/1.10.3/examples/minikube/sw_l3_l4_policy.yaml
```
## Inspect the policy details
```bash
kubectl get cnp
kubectl describe cnp rule1
```
## Inspecting the network policy and network flows with the CLI

```bash
kubectl -n kube-system get pods -l k8s-app=cilium
kubectl -n kube-system exec cilium-1c2cz -- cilium endpoint list

#Let’s issue some requests to emulate some traffic again.
kubectl exec xwing -- curl -s -XPOST deathstar.default.svc.cluster.local/v1/request-landing

hubble observe -n default --since=1m
```




